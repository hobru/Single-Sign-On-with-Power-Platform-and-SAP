# Single Sign-On with Power Platform and SAP


This repo is intended to support the [videos on configuring Single Sign-On](https://youtu.be/NSE--fVLdUg) (SSO) for the Power Platform (including Copilot Studio) with SAP. We will concentrate on SSO for the SAP OData Connector, which allows a SAML2 / OAuth flow.
[Power Platform + SAP OData - Single Sign-On - Happy path](https://www.youtube.com/watch?v=NSE--fVLdUg)
[![Power Platform + SAP OData - Single Sign-On - Happy path](./images/Thumb.jpg)](https://www.youtube.com/watch?v=VMAHSn_QgXQ&t=750s)

😁 In this section we talk about the "Happy-Path". Without a lot of explanations and background information we just show the steps that you should configure to get it working. 

🚶‍♀️‍➡️If you want to understand more of what and why, there is a [step-by-step guide](StepByStep.md) available as well. 

Then there are lot of great (official) documentations already available as well, and this guide is supposed to support, extend, summarize and supplement all of them. Let me know if you found additional useful information. 

| Name & Link | Comment |
|----------|----------|
| [Set up Microsoft Entra ID, Azure API Management, and SAP for SSO from SAP OData connector](https://learn.microsoft.com/en-us/power-platform/sap/connect/entra-id-apim-oauth) | Official documentation to setup SSO |
| [Principal propagation in a multi-cloud solution between Microsoft Azure and SAP, Part V: Production readiness with unified API- and infrastructure management](https://community.sap.com/t5/technology-blogs-by-members/principal-propagation-in-a-multi-cloud-solution-between-microsoft-azure-and/ba-p/13544042) | Amazing Blog post series by Martin Raepple |
| [.NET speaks OData too – how to implement Azure App Service with SAP Gateway](https://community.sap.com/t5/enterprise-resource-planning-blogs-by-members/net-speaks-odata-too-how-to-implement-azure-app-service-with-sap-gateway/ba-p/13493577)  | Martin Pankraz has created all the policies and concept that we are using here. This is one of his blog posts that talks about SSO |
| [Request OAuth2 access token from SAP using AAD JWT token](https://github.com/Azure/api-management-policy-snippets/blob/master/examples/Request%20OAuth2%20access%20token%20from%20SAP%20using%20AAD%20JWT%20token.xml) | Policy for Azure APIM by Martin |
| [Episode 3.​ Configure SAP Principal Propagation with AAD and SAP OAuth server](https://www.youtube.com/watch?v=agd0ygiO1Lg&list=PLvqyDwoCkBXZ85LoFrNWv9Mj88TiDAc4g&index=5) | Video by Martin explaining the SSO setup | 
| [AzureSAPODataReader](https://github.com/MartinPankraz/AzureSAPODataReader?tab=readme-ov-file) | Repo by Martin talking explaning the SSO setup | 

# Identifying the OData Service
| Transaction | Comment |
|----------|----------|
| /nBP | In this demo we are using Business Partners; BP is the transaction to create / view business partners |
| /n/IWFND/maint_service | Find and activate SAP OData services |
| /n/IWFND/GW_CLIENT | Test SAP OData services directly in the SAP system |
| /nSAML2 | SAML 2.0 Configuration of ABAP System |
| /nSOAUTH2 | OAuth 2.0 Administration |
| https://[HOSTNAME]:[PORT]/sap/bc/webdynpro/sap/sec_diag_tool?sap-client=[CLIENT] | SAP Diag Trace |
| /nSICF | Activate and verify services in your SAP system |


## Testing the OData Service
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=260s)

You can identify SAP OData Services (and other APIs) in the [SAP Business Accelerator Hub](https://api.sap.com). Once identified use transaction */n/IWFND/maint_service* to find and activate the service in your SAP System. 
![Maint_service](images/Maint_service.jpg)


# Setup trust relationship between SAP and Microsoft Entra ID using SAML 2.0
## Configuring the SAP System - Setup SAML Provider
Open Transaction */nSAML2* and export the Metadata
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=215s)

![SAML Configuration - Provider](images/SAP-SAML2-Providername.jpg)

## Create a Microsoft Entra ID enterprise application to re-present the SAP System
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=275s)

![SAML Configuration](images/EA-SAMLConfiguration1.jpg)
Logout URL: https://microsoftintegrationdemo.com:44301/sap/saml2/sp/slo/400
![SAML Configuration](images/EA-SAML-Overview.jpg)
![User and Groups assignemnt](images/EA-UsersAndGroups.jpg)
## Set up SAP to trust Microsoft Entra ID
![SAML Configuration - List of Trusted Providers](images/SAP-SAML2-OAuth.jpg)

# Configure SAP OAuth
## Create the User
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=585s)

![Create User](images/CreateUser.jpg)

## OAuth 2.0 Administration
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=615s)

![OAuth 2.0 Administration](images/SAP-OAuth-1.jpg)
![OAuth 2.0 Administration - Scope Assignment](images/SAP-OAuth-2.jpg)


# Set up a Microsoft Entra ID application that grants access to the Microsoft Power Platform SAP OData app registration
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=715s)

![App Registration Overview](images/AR-Overview.jpg)
## Authentication
Select Authentication > Add a platform > Web.
Set Redirect URIs to https://localhost:44326/signin-oidc.

## Certificate and Secrets
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=735s)

![App Certificate & Secrets](images/AR-Certificate+Secrets.jpg)

## API Permissions
![App API Permissions](images/AR-APIPermissions.jpg)
### openid
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=765s)

### Enterprise App 
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=785s)

## Expose an API
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=825s)

![App Expose API](images/AR-ExposeAPI.jpg)

### Add Scope
### Add Enterprise App
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=865s)

### Add Power Platform SAP OData Connector App
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=890s)

### Grant Consent
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=810s)

# Usermapping via Email
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=1170s)

# Azure API Management
## Create OData API
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=925s)

## Add Named Values
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=975s)

### Collect all required properties
| Properties | Example | How to get there? | Screenshot | 
|----------|----------|----------|----------|
| HBR-APIMAADRegisteredAppClientId | 481cc3b6-1eec-474d-82b6-d675a1f925e3 | [Registered App](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps) | ![HBR-APIMAADRegisteredAppClientId](images/HBR-APIMAADRegisteredAppClientId.png) | 
| HBR-APIMAADRegisteredAppClientSecret | 45W8Q~0PxMdOWs3oMlEdXIOY1uHYcb6-L0y2WbPO | [Registered App](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps)  |![HBR-APIMAADRegisteredAppClientSecret](images/HBR-APIMAADRegisteredAppClientSecret.png) | 
| HBR-AADSAPResource | https://pm4400 | Transaction SAML2 | ![HBR-AADSAPResource](images/HBR-AADSAPResource.png) | 
| HBR-AADTenantId | bf806c73-21b2-493e-842a-bb596373bf0b | [Azure Portal Overview](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/Overview) | ![HBR-AADTenantId](images/HBR-AADTenantId.png) | 
| HBR-SAPOAuthClientID | OAUTH-HOBRUC | Transaction SOAUTH2 | ![HBR-SAPOAuthClientID](images/HBR-SAPOAuthClientID.png) | 
| HBR-SAPOAuthClientSecret | SetupSSO! | Transaction SU01 | ![HBR-SAPOAuthClientSecret](images/HBR-SAPOAuthClientSecret.png) | 
| HBRSAPOAuthRefreshExpiry | 3600 | Transaction SAML2 | see screenshot below | 
| HBR-SAPOAuthScope | ZAPI_BANKACCOUNT_SRV_0001 ZAPI_MATERIAL_STOCK_SRV_0001 | Transaction SAML2 | ![HBR-SAPOAuthScope](images/HBR-SAPOAuthScope.png) | 
| HBR-SAPOAuthServerAdressForTokenEndpoint | 10.15.0.6:44301 | Transaction SAML2  | check transaction /nSMICM | 


## Add Policy
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=1115s)

# Power Platform
## Test from Power App
[Video](https://www.youtube.com/watch?v=NSE--fVLdUg&t=1185s)



