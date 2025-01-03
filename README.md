# Single-Sign-On-with-Power-Platform-and-SAP
Single Sign-On with Power Platform and SAP 

This repo is intended to support the videos on configuring Single Sign-On (SSO) for the Power Platform (including Copilot Studio) with SAP. We will concentrate on SSO for the SAP OData Connector, which allows a SAML2 / OAuth flow.

There are lot of great (official) documentations already available, and this guide is supposed to support, extend, summarize and supplement all of them. Let me know if you found additional useful information. 

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
| /n/IWFND/maint_service | Find and activate SAP OData services |
| /n/IWFND/GW_CLIENT | Test SAP OData services directly in the SAP system |
| /nSMICF ??? | Verify if services are active in your SAP system |


## Testing the OData Service

# Setup trust relationship between SAP and Microsoft Entra ID using SAML 2.0
## Configuring the SAP System - Setup SAML Provider
![SAML Configuration - Provider](images/SAP-SAML2-Providername.jpg)

## Create a Microsoft Entra ID enterprise application to re-present the SAP System
![SAML Configuration](images/EA-SAMLConfiguration1.jpg)
Logout URL: https://microsoftintegrationdemo.com:44301/sap/saml2/sp/slo/400
![SAML Configuration](images/EA-SAML-Overview.jpg)
![User and Groups assignemnt](images/EA-UsersAndGroups.jpg)
## Set up SAP to trust Microsoft Entra ID
![SAML Configuration - List of Trusted Providers](images/SAP-SAML2-OAuth.jpg)

# Configure SAP OAuth
## Create the User
## OAuth 2.0 Administration
![OAuth 2.0 Administration](images/SAP-OAuth-1.jpg)
![OAuth 2.0 Administration - Scope Assignment](images/SAP-OAuth-2.jpg)


# Set up a Microsoft Entra ID application that grants access to the Microsoft Power Platform SAP OData app registration
![App Registration Overview](images/AR-Overview.jpg)
## Authentication
Select Authentication > Add a platform > Web.
Set Redirect URIs to https://localhost:44326/signin-oidc.

## Certificate and Secrets
![App Certificate & Secrets](images/AR-Certificate+Secrets.jpg)

## API Permissions
![App API Permissions](images/AR-APIPermissions.jpg)
### openid
### Enterprise App 
## Expose an API
![App Expose API](images/AR-ExposeAPI.jpg)
### Add Application ID URI
### Add Scope
### Add Enterprise App
### Add Power Platform App
### Grant Consent

# Usermapping via Email

# Azure API Management
## Create OData API
## Add Named Values
### Collect all required properties
| Properties | Example | How to get there? | Screenshot | 
|----------|----------|----------|----------|
| HBR-APIMAADRegisteredAppClientId | 481cc3b6-1eec-474d-82b6-d675a1f925e3 | [Registered App](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps) | ![HBR-APIMAADRegisteredAppClientId]() | 
| HBR-APIMAADRegisteredAppClientSecret | 45W8Q~0PxMdOWs3oMlEdXIOY1uHYcb6-L0y2WbPO | [Registered App](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps)  |![HBR-APIMAADRegisteredAppClientSecret]() | 
| HBR-AADSAPResource | https://pm4400 | Transaction SAML2 | ![HBR-AADSAPResource]() | 
| HBR-AADTenantId | bf806c73-21b2-493e-842a-bb596373bf0b | [Azure Portal Overview](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/Overview) | ![HBR-AADTenantId]() | 
| HBR-SAPOAuthClientID | OAUTH-HOBRUC | Transaction SOAUTH2 | ![HBR-SAPOAuthClientID]() | 
| HBR-SAPOAuthClientSecret | SetupSSO! | Transaction SU01 | ![HBR-SAPOAuthClientSecret]() | 
| HBRSAPOAuthRefreshExpiry | 3600 | Transaction SAML2 | ![HBRSAPOAuthRefreshExpiry]() | 
| HBR-SAPOAuthScope | ZAPI_BANKACCOUNT_SRV_0001 ZAPI_MATERIAL_STOCK_SRV_0001 | Transaction SAML2 | ![HBR-SAPOAuthScope]() | 
| HBR-SAPOAuthServerAdressForTokenEndpoint | 10.15.0.6:44301 | Transaction SAML2  | ![HBR-SAPOAuthServerAdressForTokenEndpoint]() | 

        
        		
                
                
                        
                
            
        
        
        
                

## Add Policy

# Power Platform
## Create Power Automate Flow
## Add SAP OData Action
## Test from Power App




