# SMSIdentityVerification
One Click User Identity Verification for Halo PSA using Microsoft Graph (CSP/MSP Friendly). Refactored for Hack Together 2023.

This tool was originally featured on my blog here: https://mspautomator.com/2022/08/27/one-click-user-identity-verification-from-halopsa/
Introduction

How do you know that person calling your helpdesk is who they say they are? Social engineering a helpdesk employee is a highly effective method of bypassing physical and logical access controls to breach an environment. This is a big enough problem in single silo organizations that have internal IT teams, but it presents a much larger attack surface for an MSP. You can’t “know” every one of your thousands of end users at clients, and that’s especially true for new employees joining your helpdesk team and starting from zero. Today we’re going to take a look at a creative way to make your own user identity verification system that avoids some of the pitfalls of commercially available products and harnesses Twilio, Microsoft Graph, and Azure Automation runbooks all from one click inside HaloPSA.

Nightmare fuel. Got it. Isn’t there already a product for this?

There’s an entire app industry that supports itself trying to fill this need. Nothing I’ve found has really hit the mark of usability and flexibility to make it widely adoptable. QuickPass and Duo are two of the biggest players in this market for MSPs and both have a really solid product. Before you even get to a cost per user (for Quickpass, Duo’s helpdesk verification product is free), the problem is threefold:

    1. It requires another app that needs to be installed during onboarding. It introduces additional support work when users switch phones and need to have Duo reactivated.
    
    2. It is dependent on the app being installed, updated, logged in (for Quickpass), and working correctly. Users delete things from phones, change permissions on apps because an article told them to, etc.
    
    3. It is driven by data provided by the user and dependent on that data being updated when it changes. Users who change phone numbers will continue to use Duo in push mode after recovering with their master password without realizing they need to update their associated phone number to get an SMS auth request. 

(Nightmare fuel intensifies) Graph, Twilio, HaloPSA, and Azure Automation to the rescue!

I will start this paragraph by saying I don’t think there is an all-in-one perfect solution for this problem. I will follow that statement by saying we can get really fucking close by invoking the holy trinity of APIs: HaloPSA, Twilio, and Microsoft Graph. Today we will implement a solution that does the following:

    1. Agent pushes an action button in HaloPSA named “Verify User” that sends an Azure Automation webhook to our script with the user’s email address and Ticket ID.
    
    2. We extract the tenant ID from the user’s email address and piggyback off the HaloPSA CSP AppID to call Microsoft Graph.
    
    3. Microsoft Graph looks up the user’s currently registered SMS MFA method in Azure AD and returns it to our script.
    
    4. Our script generates a 6 digit verification code and sends it via SMS to the registered number using the Twilio API.
    
    5. Post the code and details back to HaloPSA as a private note so the Agent can verify the information.
    
    6. (Optionally) Log the details to an Azure SQL DB.

Prerequisites

    1. An Azure Automation 5.1 Runbook or PowerShell Universal Dashboard with an API Endpoint configured. PowerShell Universal is great for this task but will run the script much slower than Azure Automation.
    
    2. The HaloPSA integration for Microsoft CSP enabled and configured (unless you want to make a separate app registration, which is fine too).
    
        2a. Alternatively, you can add your Azure RunAs Account service principal to the AdminAgents group and use that service principal to call MS Graph.
        
        2b. If you want to create a new AppID for this, the scopes you’ll need in Graph are “UserAuthenticationMethod.Read.All”, “User.Read.All”
        
    3. A HaloPSA API AppID and ClientSecret (services)
    
    4. A Twilio account, a number to send from, and your SID and Token (from the main dashboard page)
    
    5. An Azure Keyvault with your RunAs granted permission to read secrets.
    
    6. My simplified HaloAPI module for private notes uploaded to your Azure Automation account
    
    7. The following modules available in your automation account (PowerShell 5.1):
    
        7a. MSAL.PS
        
        7b. Sqlserver (if logging to SQL)
        
        7c. Az.Keyvault
        
        7d. Az.Accounts
        
        7e. Az.Automation
        
        7f. Microsoft.Graph.Authentication
        
        7g. Microsoft.Graph.Users
        
        7h. Microsoft.Graph.Identity.DirectoryManagement
        
        7i. Microsoft.Graph.Identity.SignIns
