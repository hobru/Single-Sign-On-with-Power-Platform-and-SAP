# Configuring and testing SSO Setup Step-By-Step
This is a step-by-step configuration of the SSO flow outlined [here](README.md). 
You can find the HTTP REST calls in [this script](REST-Client-Scripts-For-SSO.http) and the full policy used [here](APIM-Policy.xml). 

The goal is not to end up with the super sophisticated Policy created by my colleague Martin Pankraz, but to show the creation of the policy and only focus on he main parts. The result is not meant to be used in production (again: here the policy from [Martin](https://github.com/Azure/api-management-policy-snippets/blob/master/examples/Request%20OAuth2%20access%20token%20from%20SAP%20using%20AAD%20JWT%20token.xml) is much better), but to help you understand the relevant steps. 

## Create a Simple API Proxy in Azure APIM
### Setup hard-coded SSO in the APIM Policy
The goal of this Step by step setup is to end-up with the most important parts of the APIM Policy that is also avaialble here, [AzureSAPODataReader](https://github.com/MartinPankraz/AzureSAPODataReader?tab=readme-ov-file). My colleauge Martin Pankraz has created this amazing policy which can do much more than just a simple SSO flow. However, when you look at it the first time, it can be overwhelming. That's why we set it up step by step.
On the way we will collect all the relevant datapoints to use the "real" policy in the end. 


In the inbound step of the policy add a step to set authentication basic:
```XML
        <authentication-basic username="pp-sbs" password="SSOSetup!" />
```

as a result the Policy should look like:
![Simple Policy](images/sbs/SimpleAPIMPolicy.jpg)

## Register an Application in Entra-ID to be used for SSO from Power Platform

In the Entra-ID tenant that also runs the Power Platform environment, register an Application called, hbr-sbs-power-to-apim
![Registred App](images/sbs/RegisteredApp.jpg)

Note down the first variables:
| Properties | Value |
|----------|----------|
| **HBR-AADTenantId** | bf806c73-21b2-493e-842a-bb596373bf0b |
| **HBR-APIMAADRegisteredAppClientId** | c24896d9-4975-40c4-8ad9-0bf7e3e08537 |
| HBR-APIMAADRegisteredAppClientSecret |  | 
| HBR-AADSAPResource |  | 
| HBR-SAPOAuthClientID |  | 
| HBR-SAPOAuthClientSecret |  | 
| HBRSAPOAuthRefreshExpiry |  | 
| HBR-SAPOAuthScope | | 
| HBR-SAPOAuthServerAdressForTokenEndpoint |  | 
| Power Platform OData Connector App ID URI | | 



In this application go to Manage -> Expose an API and select Add Application ID URI, 
![App ID URI](images/sbs/ExposeAPI-AppID.jpg)

### Add a scope
Click on "Add a scope" and add user_impersonation
![Add scope](images/sbs/AddScope.jpg)

Leave the deafults, like api://c24896d9-4975-40c4-8ad9-0bf7e3e08537 

| Properties | Value |
|----------|----------|
| HBR-AADTenantId | bf806c73-21b2-493e-842a-bb596373bf0b |
| HBR-APIMAADRegisteredAppClientId | c24896d9-4975-40c4-8ad9-0bf7e3e08537 |
| HBR-APIMAADRegisteredAppClientSecret |  | 
| HBR-AADSAPResource |  | 
| HBR-SAPOAuthClientID |  | 
| HBR-SAPOAuthClientSecret |  | 
| HBRSAPOAuthRefreshExpiry |  | 
| HBR-SAPOAuthScope | | 
| HBR-SAPOAuthServerAdressForTokenEndpoint |  | 
| **Power Platform OData Connector App ID URI** | api://c24896d9-4975-40c4-8ad9-0bf7e3e08537  | 



## Create a Power Automate Flow and add the SAP OData Action
![Create Connection in Power Automate flow](images/sbs/PowerCreateConnection.jpg)

This connection will not work yet, since the application we registered before does not allow the Power Automate flow to access it. 
![Error  Connection in Power Automate flow](images/sbs/PowerCreateConnection-Error.jpg)
```XML
The client has requested access to a resource which is not listed in the requested permissions in the client's application registration. Client app ID: 6bee4d13-fd19-43de-b82c-4b6401d174c3(PowerPlatform-SapOData-Connector).
```

## Add Power Platform OData Connector to registered app
In the hbr-sbs-power-to-apim registred app under Manage -> Expose API click on Add a client application and add the PowerPlatform-SapOData-Connector with App id: 6bee4d13-fd19-43de-b82c-4b6401d174c3
![Add Client Application](images/sbs/AddApplication-Connector.jpg)

Now we can sign-in from the Power Automate flow and select the Entity type from our SAP OData Service. Obviously this is not a single sign-on with principal propagation (we have hard-coded the SAP user in the APIM Policy), but it helps us to see the flow. After a successful authentication in Power Automate, the API and the policy in APIM is called and executed successfully. 
![APIM Logs](images/sbs/APIMLogs.jpg)

## Add external tracing to the policy

In order to trace what is actually happening we can perform a call-out from the APIM Policy to a Webserver that just shows us the content. This can be a site like https://webhook.site/ or https://pipedream.com/ or your own HTTP server running locally. 

> Note: I do not recommend to use public sites for productive environment / credentials!

In my demo-setup (all credentials / apps have been deleted after publishing this tutorial) I am using pipedream. 

After registerting with Pipedream and getting a "simple" endpoint, we update the Policy in Azure API Management. 

At first we create a new *Named Value* HBR-TracingURL with the URL to Pipedream, e.g. *https://eoziz0bkam8k6vo.m.pipedream.net*

With this we can update the policy to make a callout to this URL whenever the API is called, 

```xml
        <!-- Parameters: HBR-TracingURL - URL to a POST-Dump URL like pipedream, e.g. https://eoziz0bkam8k6vo.m.pipedream.net -->
<!-- Tracing 1 -->
        <send-request mode="new" response-variable-name="hbr-Trace-1" timeout="10" ignore-error="true">
            <set-url>{{HBR-TracingURL}}</set-url>
            <set-method>POST</set-method>
            <set-header name="Content-Type" exists-action="override">
                <value>plain/text</value>
            </set-header>
            <set-header name="X-Custom-HBR" exists-action="override">
                <value>Step 1 - Initial call from Power</value>
            </set-header>
            <set-body>Initial Call from Power Platform</set-body>
        </send-request>
```
![APIM Policy](images/sbs/APIM-Policy1.jpg)

Now each time the API is called, we can see a trace in pipedream. 
![Pipedream](images/sbs/Pipedream1.jpg)

## Get Assertion from Entra-ID
With the application that we have now registered, we can get an assertion (which we can later use in our authentication flow). 

Note: In order to test this also from a REST Client, we need to *Add a platform* under Authentication of type Web with the redirect URL https://localhost:44326/signin-oidc 
![Authentication - Web - Redirect](images/sbs/Authentication-Redirect.jpg)

This can be done manually, by calling this URL
```text
https://login.microsoftonline.com/<TENANT-ID>/oauth2/v2.0/authorize?client_id=<CLIENT-ID>&response_type=token&redirect_uri=https://localhost:44326/signin-oidc&scope=openid profile api://<CLIENT-ID>/user_impersonation&response_mode=fragment

```

e.g. 
```text
https://login.microsoftonline.com/bf806c73-21b2-493e-842a-bb596373bf0b/oauth2/v2.0/authorize?client_id=c24896d9-4975-40c4-8ad9-0bf7e3e08537&response_type=token&redirect_uri=https://localhost:44326/signin-oidc&scope=openid profile api://c24896d9-4975-40c4-8ad9-0bf7e3e08537/user_impersonation&response_mode=fragment
```

When we open this in a browser, the call -- after an authentication with the (Power Platform) end-user -- is redirected and in the URL we get an Access token. 
![Access Token](images/sbs/AccessToken.jpg)

This Access Token contains all the relevant user-information that we need to do the principal propagation and later on replace this with an access token for the SAP System (or any other backend system that you want to access). 

In the case of the Power Automate flow, this is actully done in the background when the Azure APIM Policy is called. Unlike in the browser, the user is already authenticated and as a result we immediately get the access token. 

Before we do that, lets take the access token and "look inside". For this copy the Access token from the URL. 
Note: Make sure to only copy the token. The reply URL continues with  additional query parameters, like token_type. 
```text
https://localhost:44326/signin-oidc#access_token=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6InoxcnNZSEhKOS04bWdndDRIc1p1OEJLa0JQdyIsImtpZCI6InoxcnNZSEhKOS04bWdndDRIc1p1OEJLa0JQdyJ9.eyJhdWQiOiJhcGk6Ly9jMjQ4OTZkOS00OTc1LTQwYzQtOGFkOS0wYmY3ZTNlMDg1MzciLCJpc3MiOiJodHRwczovL3N0cy53aW5kb3dzLm5ldC9iZjgwNmM3My0yMWIyLTQ5M2UtODQyYS1iYjU5NjM3M2JmMGIvIiwiaWF0IjoxNzM1NjYzNDU5LCJuYmYiOjE3MzU2NjM0NTksImV4cCI6MTczNTY2OTA3MiwiYWNyIjoiMSIsImFpbyI6IkFWUUFxLzhZQUFBQUFjYTZZRDJLclBzYjMxSG9XNU16YnI3TXIvQjJhMU5jNER0dy9JekUwdjU4THJ5NkZkMnBHZm5JSE1vUE5jZU1VZkd6VXE4WjZkWjB4djByNjV3MDRoNFI0WC9tQzVHRzQzTmNJUk5BVmJBPSIsImFtciI6WyJwd2QiLCJtZmEiXSwiYXBwaWQiOiJjMjQ4OTZkOS00OTc1LTQwYzQtOGFkOS0wYmY3ZTNlMDg1MzciLCJhcHBpZGFjciI6IjAiLCJmYW1pbHlfbmFtZSI6Ikxvd0NvZGUiLCJnaXZlbl9uYW1lIjoiMTB4RGF5cyBvZiIsImlwYWRkciI6IjE3Mi4yMDEuNzcuNDMiLCJuYW1lIjoiMTB4RGF5cyBvZiBMb3dDb2RlIiwib2lkIjoiZjkxODM5NTgtMzdkZi00NjQzLThlYjctNDkxYTNkNjlkYTgwIiwicmgiOiIxLkFYa0FjMnlBdjdJaFBrbUVLcnRaWTNPX0M5bVdTTUoxU2NSQWl0a0w5LVBnaFRlVUFFZDVBQS4iLCJzY3AiOiJ1c2VyX2ltcGVyc29uYXRpb24iLCJzdWIiOiJybUIwNk1CUnRSZUNEME0wazJHY3lQNzhOM1llbGNnYkt5X05rNEVGVHZrIiwidGlkIjoiYmY4MDZjNzMtMjFiMi00OTNlLTg0MmEtYmI1OTYzNzNiZjBiIiwidW5pcXVlX25hbWUiOiJIdW5kcmVkWERheXNPZkxvd0NvZGVAeDQya3Aub25taWNyb3NvZnQuY29tIiwidXBuIjoiSHVuZHJlZFhEYXlzT2ZMb3dDb2RlQHg0MmtwLm9ubWljcm9zb2Z0LmNvbSIsInV0aSI6IllqUnFPSm90N1VpM2hBc2xHRzgxQVEiLCJ2ZXIiOiIxLjAifQ.KdNkXcBUoMu0VATRwJZ93Xu9WogOHW9TeXkAoLAXD4O0WYALbD8tO3KjBcvzMpeLej88HAWu2MXlw3zlFs4zNxeFNVVcl_2COYbwfM--npLQ_VXs_nJg_YV5WbmLsIY-W1mW6GNRiyVXuc3b2_Qn-Hd01IQHJ-Phf2nzRYOzZnXXygYbMufPV6Uhs53_fdXrdnZQA6qL6ABZHR-pmRhJtbfDI99kMMQXoWPaMYWsTWXMTk7gd6Ul8Wc_F1-HWtLPdEtIOd1yms0YAbSwYoOp_IANqG92H3UbpvCJR6PRVAYtDUsaV2AYhoAmbk-Y_wxB2dK-FifgxX8GxacBvFEw-A&token_type=Bearer&expires_in=5312&scope=api%3a%2f%2fc24896d9-4975-40c4-8ad9-0bf7e3e08537%2fuser_impersonation&session_state=7df40107-42aa-45cd-9ba3-f79d7f0b1a53
```

You should only copy this:

```xml
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6InoxcnNZSEhKOS04bWdndDRIc1p1OEJLa0JQdyIsImtpZCI6InoxcnNZSEhKOS04bWdndDRIc1p1OEJLa0JQdyJ9.eyJhdWQiOiJhcGk6Ly9jMjQ4OTZkOS00OTc1LTQwYzQtOGFkOS0wYmY3ZTNlMDg1MzciLCJpc3MiOiJodHRwczovL3N0cy53aW5kb3dzLm5ldC9iZjgwNmM3My0yMWIyLTQ5M2UtODQyYS1iYjU5NjM3M2JmMGIvIiwiaWF0IjoxNzM1NjYzNDU5LCJuYmYiOjE3MzU2NjM0NTksImV4cCI6MTczNTY2OTA3MiwiYWNyIjoiMSIsImFpbyI6IkFWUUFxLzhZQUFBQUFjYTZZRDJLclBzYjMxSG9XNU16YnI3TXIvQjJhMU5jNER0dy9JekUwdjU4THJ5NkZkMnBHZm5JSE1vUE5jZU1VZkd6VXE4WjZkWjB4djByNjV3MDRoNFI0WC9tQzVHRzQzTmNJUk5BVmJBPSIsImFtciI6WyJwd2QiLCJtZmEiXSwiYXBwaWQiOiJjMjQ4OTZkOS00OTc1LTQwYzQtOGFkOS0wYmY3ZTNlMDg1MzciLCJhcHBpZGFjciI6IjAiLCJmYW1pbHlfbmFtZSI6Ikxvd0NvZGUiLCJnaXZlbl9uYW1lIjoiMTB4RGF5cyBvZiIsImlwYWRkciI6IjE3Mi4yMDEuNzcuNDMiLCJuYW1lIjoiMTB4RGF5cyBvZiBMb3dDb2RlIiwib2lkIjoiZjkxODM5NTgtMzdkZi00NjQzLThlYjctNDkxYTNkNjlkYTgwIiwicmgiOiIxLkFYa0FjMnlBdjdJaFBrbUVLcnRaWTNPX0M5bVdTTUoxU2NSQWl0a0w5LVBnaFRlVUFFZDVBQS4iLCJzY3AiOiJ1c2VyX2ltcGVyc29uYXRpb24iLCJzdWIiOiJybUIwNk1CUnRSZUNEME0wazJHY3lQNzhOM1llbGNnYkt5X05rNEVGVHZrIiwidGlkIjoiYmY4MDZjNzMtMjFiMi00OTNlLTg0MmEtYmI1OTYzNzNiZjBiIiwidW5pcXVlX25hbWUiOiJIdW5kcmVkWERheXNPZkxvd0NvZGVAeDQya3Aub25taWNyb3NvZnQuY29tIiwidXBuIjoiSHVuZHJlZFhEYXlzT2ZMb3dDb2RlQHg0MmtwLm9ubWljcm9zb2Z0LmNvbSIsInV0aSI6IllqUnFPSm90N1VpM2hBc2xHRzgxQVEiLCJ2ZXIiOiIxLjAifQ.KdNkXcBUoMu0VATRwJZ93Xu9WogOHW9TeXkAoLAXD4O0WYALbD8tO3KjBcvzMpeLej88HAWu2MXlw3zlFs4zNxeFNVVcl_2COYbwfM--npLQ_VXs_nJg_YV5WbmLsIY-W1mW6GNRiyVXuc3b2_Qn-Hd01IQHJ-Phf2nzRYOzZnXXygYbMufPV6Uhs53_fdXrdnZQA6qL6ABZHR-pmRhJtbfDI99kMMQXoWPaMYWsTWXMTk7gd6Ul8Wc_F1-HWtLPdEtIOd1yms0YAbSwYoOp_IANqG92H3UbpvCJR6PRVAYtDUsaV2AYhoAmbk-Y_wxB2dK-FifgxX8GxacBvFEw-A
```

You can use website like https://jwt.io/ or https://jwt.ms/ to analyse the token. I prefer [DevToys](https://devtoys.app/) which can be installed locally.  
![JWT Token in DevToys](images/sbs/DevToys.jpg)

```json
{
    "aud": "api://c24896d9-4975-40c4-8ad9-0bf7e3e08537",
  "iss": "https://sts.windows.net/bf806c73-21b2-493e-842a-bb596373bf0b/",
  "iat": 1735663459,
  "nbf": 1735663459,
  "exp": 1735669072,
  "acr": "1",
  "aio": "AVQAq/8YAAAAAca6YD2KrPsb31HoW5Mzbr7Mr/B2a1Nc4Dtw/IzE0v58Lry6Fd2pGfnIHMoPNceMUfGzUq8Z6dZ0xv0r65w04h4R4X/mC5GG43NcIRNAVbA=",
  "amr": [
      "pwd",
    "mfa"
  ],
  "appid": "c24896d9-4975-40c4-8ad9-0bf7e3e08537",
  "appidacr": "0",
  "family_name": "LowCode",
  "given_name": "10xDays of",
  "ipaddr": "172.201.77.43",
  "name": "10xDays of LowCode",
  "oid": "f9183958-37df-4643-8eb7-491a3d69da80",
  "rh": "1.AXkAc2yAv7IhPkmEKrtZY3O_C9mWSMJ1ScRAitkL9-PghTeUAEd5AA.",
  "scp": "user_impersonation",
  "sub": "rmB06MBRtReCD0M0k2GcyP78N3YelcgbKy_Nk4EFTvk",
  "tid": "bf806c73-21b2-493e-842a-bb596373bf0b",
  "unique_name": "HundredXDaysOfLowCode@x42kp.onmicrosoft.com",
  "upn": "HundredXDaysOfLowCode@x42kp.onmicrosoft.com",
  "uti": "YjRqOJot7Ui3hAslGG81AQ",
  "ver": "1.0"
}
```

Among other things we can see in this JWT token the *unique_name*. This could potentially already be good enough to have a simple mapping table in our API Management Policy and perform a hard-coded SSO to the SAP backend system. 

In our case we want take this token and request SAML assertion from Entra-ID with the On behalf of (ObO) flow v2. Via a REST client this can be done by calling the oauth2 token endpoint from login.microsoftonline.com. 

However, before doing that we need to Create a *Client Secret* for our registered app. Navigate to *Certificates & secrets* and click on *New Client Secret*. Provide a name for the description, e.g. CS-For-PowerToAPIM and click on add. 
Make sure to remember and write down the value of the newly created client secrect. 
![Client Secret](images/sbs/ClientSecrect.jpg)


| Properties | Value |
|----------|----------|
| HBR-AADTenantId | bf806c73-21b2-493e-842a-bb596373bf0b |
| HBR-APIMAADRegisteredAppClientId | c24896d9-4975-40c4-8ad9-0bf7e3e08537 |
| **HBR-APIMAADRegisteredAppClientSecret** | PFi8Q~YbxjNJTBIOdkLwB1IxlEeRRRplta3-0di2  | 
| HBR-AADSAPResource |  | 
| HBR-SAPOAuthClientID |  | 
| HBR-SAPOAuthClientSecret |  | 
| HBRSAPOAuthRefreshExpiry |  | 
| HBR-SAPOAuthScope | | 
| HBR-SAPOAuthServerAdressForTokenEndpoint |  | 
| Power Platform OData Connector App ID URI | api://c24896d9-4975-40c4-8ad9-0bf7e3e08537  | 

## Enterprise application to represent the SAP system
The ObO flow also needs a scope to create the SAML assertion for the SAP System. So the next step is to create an Enterprise Application that represents the SAP system. You could do the settings manually, but the easiest thing is to configure the backend / SAP System itself first and then download the required metadata and import it while creating the Enterprise App. 

# Configuring the SAP System
In our case we are using the SAP backend system as the local SAML provider. We could do the same for the SAP Identity Provider on BTP or any other backend system. This is again a situation where using the policy in the API Management system provides us with all the flexibility we need to establish a SSO flow from Power Automte to the SAP System. If you are using a different authentication provider for your SAP (or non-SAP system), you could configure and use this information for the Enterprise App in Entra ID later on. 

Open Transaction */nSAML2* in your SAP system, which opens the 
SAML 2.0 Configuration of ABAP System, e.g. https://10.15.0.6:44301/sap/bc/webdynpro/sap/saml2?sap-client=400&sap-language=EN# 
![SAML2](images/sbs/SAML2.jpg)

In case the Local provider is no yet configured, follow the wizard and make sure that the provider name is something like this, https://\<SAP-ID>\<SAP-Client>, e.g. https://pm4400

| Properties | Value |
|----------|----------|
| HBR-AADTenantId | bf806c73-21b2-493e-842a-bb596373bf0b |
| HBR-APIMAADRegisteredAppClientId | c24896d9-4975-40c4-8ad9-0bf7e3e08537 |
| HBR-APIMAADRegisteredAppClientSecret | PFi8Q~YbxjNJTBIOdkLwB1IxlEeRRRplta3-0di2  | 
| **HBR-AADSAPResource** | https://pm4400 | 
| HBR-SAPOAuthClientID |  | 
| HBR-SAPOAuthClientSecret |  | 
| HBRSAPOAuthRefreshExpiry |  | 
| HBR-SAPOAuthScope | | 
| HBR-SAPOAuthServerAdressForTokenEndpoint |  | 
| Power Platform OData Connector App ID URI | api://c24896d9-4975-40c4-8ad9-0bf7e3e08537  |

That's all for now. Click on *Metadata* and download the Metadata with all the entity descriptors included. 

## Create Enterprise application in Entra-ID
Now in Entra-ID under *Manage*, select *[Enterprise applications](https://portal.azure.com/#view/Microsoft_AAD_IAM/StartboardApplicationsMenuBlade/~/AppAppsPreview/menuId~/null) and click on *New Application*

Search and click on *SAP NetWeaver*
![Create Enterprise App - SAP NetWeaver](images/sbs/SAPNetWeaver.jpg)

Click on *Single sign-on*, select *SAML* and click on *Upload metadata file*
![Upload Metadata](images/sbs/UploadMetadata.jpg)

We will now need to adjust some of the properties. Depending on the configuration in your SAP system, the servername, port and path might vary. We will continue here with a "best guess" but later in a troubleshooting step come back and correct it. 

* The *Identifier (Entity ID)* has to be the name of the local provider in your SAP system (as configured in the previous step). 
* The *Reply URL (Assertion Consumer Service URL)* needs to be changed to the token end-point of your SAP system. Typically this is something int he form of https://\<SAP-Server-Name>:\<SAP-PORT>/sap/bc/sec/oauth2/token?sap-client=400
* The *Sign on URL* is not used in our scenario so we can change it to https://dummy.com or any other valid URI
* For the *Logout Url (Optional)* make sure that it is also an HTTPS enabled URL. 
![SAML Configuration](images/sbs/SAMLConfiguration.jpg)

### Test the ObO flow
Now we have all the properties to call our ObO flow. However, calling


```text
###
### Request SAML assertion from AAD with ObO flow v2
###
# Assertion from previous request
# Enter here the assertion from the previous request executed in the browser
@assertion = <Copied manuall from previous login-call in the browser>
# @name fetchAccessTokenForSAP

POST https://login.microsoftonline.com/{{HBR-AADTenantId}}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
&assertion={{assertion}}
&client_id={{HBR-APIMAADRegisteredAppClientId}}
&client_secret={{HBR-APIMAADRegisteredAppClientSecret}}
&scope={{HBR-AADSAPResource}}/.default
&requested_token_use=on_behalf_of
&requested_token_type=urn:ietf:params:oauth:token-type:saml2
```

Results in an error message 
```text
AADSTS50105: Your administrator has configured the application HBR - SAP NetWeaver - SBS - PM4  ('d4ad39a1-ca42-45b1-a3ca-c22244219a96') to block users unless they are specifically granted ('assigned') access to the application.  The signed in user '{EUII Hidden}' is blocked because they are not a direct member of a group with access, nor had access directly assigned by an administrator. Please contact your administrator to assign access to this application."
```
![Calling ObO - Part 1](images/sbs/CallingObO-1.jpg)

So we need to first allow users to call the enterprise app. Select *Users and Groups*, click on *+Add user/group*, *None selected* and select all the users that should be able to leverage the Single Sign-On flow. 
![Assigned Users](images/sbs/AssignedUsers.jpg)

Performing the previous ObO call results in another error:
```text
AADSTS65001: The user or administrator has not consented to use the application with ID 'c24896d9-4975-40c4-8ad9-0bf7e3e08537' named 'hbr-sbs-power-to-apim'. Send an interactive authorization request for this user and resource.
```

So next we need to go back to the previously [registered app](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps) (in our case hbr-sbs-power-to-apim) and under *API Permissions* select *Add a permission* and *APIs my organization uses*. Then search for the name of the Enterprise App, e.g. *HBR - SAP NetWeaver - SBS - PM4* and select *Add permissions* with the *user_impersonation* selected.  
![Add Permissions for Enterprise App](images/sbs/AddPermissions.jpg)

As a final step we need to click on *Grant admin consent for ...*
![Grant admin consent](images/sbs/GrantAdminConsent-1.jpg)

### Successfully test the ObO flow
Now the call of the ObO flow works successful and we get an access token as a result. 
![Successful ObO Call](images/sbs/SuccessfulOboCall.jpg)

This access token can be anaylized using a Base64 decoder. In my case decoding the token looked liked (you again might use tools like DevToys to )

```xml
<Assertion ID="_b311ea5b-8fab-4924-80f0-0c6abac42e00" IssueInstant="2025-01-01T15:13:26.743Z" Version="2.0" xmlns="urn:oasis:names:tc:SAML:2.0:assertion">
  <Issuer>https://sts.windows.net/bf806c73-21b2-493e-842a-bb596373bf0b/</Issuer>
  <Signature xmlns="http://www.w3.org/2000/09/xmldsig#">
    <SignedInfo>
      <CanonicalizationMethod Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#" />
      <SignatureMethod Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256" />
      <Reference URI="#_b311ea5b-8fab-4924-80f0-0c6abac42e00">
        <Transforms>
          <Transform Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature" />
          <Transform Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#" />
        </Transforms>
        <DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256" />
        <DigestValue>9dt1OYV/HkTgBqiT7l+7XvxVckShBbk2Icbw6i9WNlM=</DigestValue>
      </Reference>
    </SignedInfo>
    <SignatureValue>bXfh5j52QcndAU+npZtOYNeqWxPBl2ayPh91RsjywtMlCpd/FdMVonPbOwSAKZiMSMArnXtA2jx3KHOh1oScBhsmt9rTgcL68fqZ7+ZSG56B7UGdpQcgVAxDoprWtI5uhm40IB84GVG2ZeZIAK38mqNRjrpNgWs48SwHy+gLQiR4JG4CPm2hu8JmmKt1ChAKnDU3Z+p3vRhKCcjwFKTMw1anyhMXyIlCK5dQFPvZ2CNJmJpVku6mwdpLHJvbpXZy9HXSvmoS85kBKDh/Yi0hfOmQxVRliZ2Mp9q4r840ERADCDB8r0vg7rqyHjh0y9chI8Vbt2LU5ohqW2UUwjyUdA==</SignatureValue>
    <KeyInfo>
      <X509Data>
        <X509Certificate>MIIC8DCCAdigAwIBAgIQbAmwESG4nLZOa49hI0zxYjANBgkqhkiG9w0BAQsFADA0MTIwMAYDVQQDEylNaWNyb3NvZnQgQXp1cmUgRmVkZXJhdGVkIFNTTyBDZXJ0aWZpY2F0ZTAeFw0yNTAxMDExNDUyMTZaFw0yODAxMDExNDUyMTZaMDQxMjAwBgNVBAMTKU1pY3Jvc29mdCBBenVyZSBGZWRlcmF0ZWQgU1NPIENlcnRpZmljYXRlMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEArx96YNbZMrjHw3SCSsIuSYg8fHbDeFpIKbKgMEbINbAN4c9yms8jvarYnw6iHR5xIlVVGTxXeOGbF5A663spFzbf+oOE0tob4sEn90GoQw52LGjnu0+95nJZmKpk6NrMwnOZ+rjNrNv3r5W5x+J9BW9BWq04LdVN8LTgt27Nn4NSzkfBc6ojBPTYtgII+hhAig1iDm8Ct218jFf0N1LXhreopchoMCFq9Y60FD3/Ou9cHN3sjZwivZHhtOn0gCtHNsSmmyK4r4GCHVTu6Sfg/a24wQFoIm5dw/Gmjl7qEQLK5JVJuGZo4M0lSC0/SY8MBeJ2MlLUtsMi3m9OQMV/IQIDAQABMA0GCSqGSIb3DQEBCwUAA4IBAQCFA0nrGRTQsC/g6kzDmbcxnCUOvSrJJ+wCmfst/PyRdbi0BfxCtwOzCiePZymWyNbLryzIxeLmxhFpnD+pTrhpjlbvAqUHdYdxnBFYgML0lY0gfHCzfASxM7hnqCzyasvVEv8dKoXaVDUtUZ/GV8jZ4azhsWBuWT2qyPEZZyPXfdsq0/CEwig3Jw4gWHT/oGO3Yza39zFk127MUjm/3QwGfddcZouK64dBuj0vTIAH3Ug9eZxZ+wy2T7fu+ABLLVWMM9oaWXonNtFSxQxFQXguxAIYdEAdFRcaVhMGI19BhYGVZ3xthDh4u6/eQ8o6wWFjp/szOrMavEqpexvW1np+</X509Certificate>
      </X509Data>
    </KeyInfo>
  </Signature>
  <Subject>
    <NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">HundredXDaysOfLowCode@x42kp.onmicrosoft.com</NameID>
    <SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
      <SubjectConfirmationData NotOnOrAfter="2025-01-01T15:26:03.619Z" Recipient="https://microsoftintegrationdemo.com:44301/sap/bc/sec/oauth2/token?sap-client=400" />
    </SubjectConfirmation>
  </Subject>
  <Conditions NotBefore="2025-01-01T15:08:26.619Z" NotOnOrAfter="2025-01-01T15:26:03.619Z">
    <AudienceRestriction>
      <Audience>https://pm4400</Audience>
    </AudienceRestriction>
  </Conditions>
  <AttributeStatement>
    <Attribute Name="http://schemas.microsoft.com/identity/claims/tenantid">
      <AttributeValue>bf806c73-21b2-493e-842a-bb596373bf0b</AttributeValue>
    </Attribute>
    <Attribute Name="http://schemas.microsoft.com/identity/claims/objectidentifier">
      <AttributeValue>f9183958-37df-4643-8eb7-491a3d69da80</AttributeValue>
    </Attribute>
    <Attribute Name="http://schemas.microsoft.com/identity/claims/displayname">
      <AttributeValue>10xDays of LowCode</AttributeValue>
    </Attribute>
    <Attribute Name="http://schemas.microsoft.com/identity/claims/identityprovider">
      <AttributeValue>https://sts.windows.net/bf806c73-21b2-493e-842a-bb596373bf0b/</AttributeValue>
    </Attribute>
    <Attribute Name="http://schemas.microsoft.com/claims/authnmethodsreferences">
      <AttributeValue>http://schemas.microsoft.com/ws/2008/06/identity/authenticationmethod/password</AttributeValue>
      <AttributeValue>http://schemas.microsoft.com/claims/multipleauthn</AttributeValue>
    </Attribute>
    <Attribute Name="http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname">
      <AttributeValue>10xDays of</AttributeValue>
    </Attribute>
    <Attribute Name="http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname">
      <AttributeValue>LowCode</AttributeValue>
    </Attribute>
    <Attribute Name="http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress">
      <AttributeValue>HundredXDaysOfLowCode@x42kp.onmicrosoft.com</AttributeValue>
    </Attribute>
    <Attribute Name="http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name">
      <AttributeValue>HundredXDaysOfLowCode@x42kp.onmicrosoft.com</AttributeValue>
    </Attribute>
  </AttributeStatement>
  <AuthnStatement AuthnInstant="2024-11-07T13:42:45.144Z">
    <AuthnContext>
      <AuthnContextClassRef>urn:oasis:names:tc:SAML:2.0:ac:classes:Password</AuthnContextClassRef>
    </AuthnContext>
  </AuthnStatement>
</Assertion>
```

Most of the important attributes are already there. We can still see the email address under *identity/claims/name* and can also already see some relevant information from the SAP system, like the Recipient. 

Now that we know that this flow is also working, we can go back to API Management and enhance our policy. Instead of us calling the ObO flow manually, we want the policy to do this. 

> Note: We will still keep the hard-coded username and password in the policy. We are not yet ready to really have a principal propagation. Only once we have the full end-to-end flow will we remove the Basic authentication in the policy and replace it with an access token. 

For this we first add HBR_AADRegisteredAppClientId, HBR_AADRegisteredAppClientSecret and HBR_AADSAPResource as new *Named values* to the APIM configuration 
![New Named Values](images/sbs/NewNamedValues.jpg)

and then enhance the policy with these steps (right after the previous *</send-request>*) and before the line *        <set-backend-service base-url="https://10.15.0.6:44301/sap/opu/odata/sap/API_BUSINESS_PARTNER" />*

```xml
        <!-- Get Values from Azure API Management -->
        <!-- Variables for Exchange AAD Bearer token-->
        <set-variable name="HBR-APIMAADRegisteredAppClientId" value="{{HBR-APIMAADRegisteredAppClientId}}" />
        <set-variable name="HBR-APIMAADRegisteredAppClientSecret" value="{{HBR-APIMAADRegisteredAppClientSecret}}" />
        <set-variable name="HBR-AADSAPResource" value="{{HBR-AADSAPResource}}" />
        <!-- Tracing Start Exchange AAD Bearer token -->
        <send-request mode="new" response-variable-name="hbr-Trace-2" timeout="10" ignore-error="true">
            <set-url>{{HBR-TracingURL}}</set-url>
            <set-method>POST</set-method>
            <set-header name="Content-Type" exists-action="override">
                <value>application/json</value>
            </set-header>
            <set-header name="X-Custom-HBR" exists-action="override">
                <value>Step - Exchange AAD Bearer token</value>
            </set-header>
            <set-body>@{
                            var _AADRegisteredAppClientId = context.Variables["HBR-APIMAADRegisteredAppClientId"];
                            var _AADRegisteredAppClientSecret = context.Variables["HBR-APIMAADRegisteredAppClientSecret"];
                            var _AADSAPResource = context.Variables["HBR-AADSAPResource"];
                            var assertion = context.Request.Headers.GetValueOrDefault("Authorization","").Replace("Bearer ","");
                            return $"grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion={assertion}&client_id={_AADRegisteredAppClientId}&client_secret={_AADRegisteredAppClientSecret}&scope={_AADSAPResource}/.default&requested_token_use=on_behalf_of&requested_token_type=urn:ietf:params:oauth:token-type:saml2";
                        }</set-body>
        </send-request>
        <!-- Tracing End Exchange AAD Bearer token -->

        <!-- Exchange AAD Bearer token for AAD issued SAML token on behalf of logged in user -->
        <send-request mode="new" response-variable-name="fetchSAMLAssertion" timeout="10" ignore-error="false">
            <set-url>https://login.microsoftonline.com/{{HBR-AADTenantId}}/oauth2/v2.0/token</set-url>
            <set-method>POST</set-method>
            <set-header name="Content-Type" exists-action="override">
                <value>application/x-www-form-urlencoded</value>
            </set-header>
            <set-body>@{
                            var _AADRegisteredAppClientId = context.Variables["HBR-APIMAADRegisteredAppClientId"];
                            var _AADRegisteredAppClientSecret = context.Variables["HBR-APIMAADRegisteredAppClientSecret"];
                            var _AADSAPResource = context.Variables["HBR-AADSAPResource"];
                            var assertion = context.Request.Headers.GetValueOrDefault("Authorization","").Replace("Bearer ","");
                            return $"grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion={assertion}&client_id={_AADRegisteredAppClientId}&client_secret={_AADRegisteredAppClientSecret}&scope={_AADSAPResource}/.default&requested_token_use=on_behalf_of&requested_token_type=urn:ietf:params:oauth:token-type:saml2";
                        }</set-body>
        </send-request>
        <!-- Step 1a: Put the resulting access token in the variable accessToken -->
        <set-variable name="accessToken" value="@((string)((IResponse)context.Variables["fetchSAMLAssertion"]).Body.As<JObject>()["access_token"])" />
        <!-- Tracing Sart access token -->
        <send-request mode="new" response-variable-name="hbr-Trace-3" timeout="10" ignore-error="true">
            <set-url>{{HBR-TracingURL}}</set-url>
            <set-method>POST</set-method>
            <set-header name="Content-Type" exists-action="override">
                <value>plain/text</value>
            </set-header>
            <set-header name="X-Custom-HBR" exists-action="override">
                <value>Step - Access Token</value>
            </set-header>
            <set-body>@((string)context.Variables["accessToken"])</set-body>
        </send-request>
        <!-- Tracing End access token -->
```
![Enhance Policy - 1](images/sbs/EnhancePolicy2.jpg)
![Enhance Policy - 2](images/sbs/EnhancePolicy3.jpg)

>Note: Again, please do the tracing steps only if you trust the receiver (in my case pipedream.com). Never use this in production environments!

This enhancements perform the ObO call that we did previously manually in the policy, but also helps with tracing. Especially the step  *Tracing Exchange AAD Bearer token* can help to identify spaces / new-line / ... or other unwanted characters. As a result you should also be able to see the Access Token in pipedream which you can evaluate with a Base64 decoder as done before. 
![Pipedream - Access token](images/sbs/Pipedream-AccessToken.jpg)

## Request access token from SAP backend with SAML Bearer Grant Type
With the SAML assertion from Entra-ID for our SAP system, we are ready to call the Token end-point in the SAP system to finally get an access token from SAP to call our OData services. 

The first step here is to tell the SAP system to trust Entra-ID. In the steps before we had exported the metadata information from our SAP System and imported it for the configuration of the Enterprise App. Now open the Enterprise App again and export the *Certificate (Base64)* and *Federation Metadata XML* 
![Download Enterprise App Metadata](images/sbs/DownloadEnterpriseAppMetadata.jpg)

Then on the SAP system in transaction SAML2 / SAML 2.0 Configuration of ABAP System PM4/400 (/sap/bc/webdynpro/sap/saml) click on *Trusted Providers* and select *OAuth 2.0 Identity Providers* from the *Show* drop-down. THen click on *Upload Metadata file* from the *Add* menu.
![Upload Metadata](images/sbs/UploadMetadata-FromEA.jpg)

In the first step upload the metadata file
![Upload Metadata - Step 1](images/sbs/Step1-Metadata.jpg)

In the next step upload the certificate and continue through the wizard. 
![Upload Certificate - Step 2](images/sbs/Step2-Certificate.jpg)

As a last step, select E-Mail as the Supported NameID Formats
![Map Emails](images/sbs/Mapping-Email.jpg)



The next step is to configure an OAuth 2.0 Client ID. The first step is to create a system user via /nSU01. This user does not have any roles and permissions. Just a username and password. 
![SAP Client ID - SU01](images/sbs/ClientIDUser.jpg)

| Properties | Value |
|----------|----------|
| HBR-AADTenantId | bf806c73-21b2-493e-842a-bb596373bf0b |
| HBR-APIMAADRegisteredAppClientId | c24896d9-4975-40c4-8ad9-0bf7e3e08537 |
| HBR-APIMAADRegisteredAppClientSecret | PFi8Q~YbxjNJTBIOdkLwB1IxlEeRRRplta3-0di2  | 
| HBR-AADSAPResource | https://pm4400 | 
| **HBR-SAPOAuthClientID** | HOBRUCHE-SBS | 
| **HBR-SAPOAuthClientSecret** | SetupSSO! | 
| HBRSAPOAuthRefreshExpiry |  | 
| HBR-SAPOAuthScope | | 
| HBR-SAPOAuthServerAdressForTokenEndpoint |  | 
| Power Platform OData Connector App ID URI | api://c24896d9-4975-40c4-8ad9-0bf7e3e08537  |


Next start transaction /nSOAUTH2 which opens the *OAuth 2.0 Administration*, e.g. https://\<SAP-Server-Name>:\<SAP-Port>>/sap/bc/webdynpro/sap/oauth2_config?sap-client=400&sap-language=EN#

From there click on *Create* to create a new Client and select the user that we just created in SU01. You can also note down the Token lifetime. 
![Select SAP User](images/sbs/SelectUser.jpg)

| Properties | Value |
|----------|----------|
| HBR-AADTenantId | bf806c73-21b2-493e-842a-bb596373bf0b |
| HBR-APIMAADRegisteredAppClientId | c24896d9-4975-40c4-8ad9-0bf7e3e08537 |
| HBR-APIMAADRegisteredAppClientSecret | PFi8Q~YbxjNJTBIOdkLwB1IxlEeRRRplta3-0di2  | 
| HBR-AADSAPResource | https://pm4400 | 
| HBR-SAPOAuthClientID | HOBRUCHE-SBS | 
| HBR-SAPOAuthClientSecret | SetupSSO! | 
| **HBRSAPOAuthRefreshExpiry** | 3600 | 
| HBR-SAPOAuthScope | | 
| HBR-SAPOAuthServerAdressForTokenEndpoint | | 
| Power Platform OData Connector App ID URI | api://c24896d9-4975-40c4-8ad9-0bf7e3e08537  |

In the step *Client Authentication* click on Next and in the step *Grant Type Settings* select the *Trusted OAuth 2.0 IdP* from the previous step. Also make sure to check *Refresh Allowed*
![Select Trusted OAuth IdP](images/sbs/SelectTrustedOAuth.jpg)

In the last step *Scope Assignment* we can select the OData Services that can be accessed via this OAuth Client ID. In our case this is the Business Partner API. If you have not yet done so, go to Transcation /n/IWFND/MAINT_SERVICE and select the API. Then click on *OAuth* and save the service. Here you can also see the Scope. 
![Configure OData for OAuth](images/sbs/OAuthServiceConfig)

Once the OData services is enabled for OAuth, you can select it from the list, e.g. ZAPI_BUSINESS_PARTNER_0001 and finish the wizard. 
![Configure OData for OAuth](images/sbs/AddScopeID.jpg)

| Properties | Value |
|----------|----------|
| HBR-AADTenantId | bf806c73-21b2-493e-842a-bb596373bf0b |
| HBR-APIMAADRegisteredAppClientId | c24896d9-4975-40c4-8ad9-0bf7e3e08537 |
| HBR-APIMAADRegisteredAppClientSecret | PFi8Q~YbxjNJTBIOdkLwB1IxlEeRRRplta3-0di2  | 
| HBR-AADSAPResource | https://pm4400 | 
| HBR-SAPOAuthClientID | HOBRUCHE-SBS | 
| HBR-SAPOAuthClientSecret | SetupSSO! | 
| HBRSAPOAuthRefreshExpiry | 3600 | 
| **HBR-SAPOAuthScope** | ZAPI_BUSINESS_PARTNER_0001 | 
| HBR-SAPOAuthServerAdressForTokenEndpoint | | 
| Power Platform OData Connector App ID URI | api://c24896d9-4975-40c4-8ad9-0bf7e3e08537  |


### Call SAP Backend to request access token
Now it is time to test the service again and we can try to request an access token from the SAP Backend system. For this we call a call to the SAP Token service:

```text
# @name accessTokenFromSAP
POST https://<SAP-SYSTEM>>:<SAP-PORT>/sap/bc/sec/oauth2/token?sap-client=400
Content-Type: application/x-www-form-urlencoded;charset=UTF-8
Authorization: Basic {{HBR-SAPOAuthClientID}}:{{HBR-SAPOAuthClientSecret}}

grant_type=urn:ietf:params:oauth:grant-type:saml2-bearer
&assertion={{fetchAccessTokenForSAP.response.body.access_token}}
&client_id={{HBR-SAPOAuthClientID}}
&scope={{HBR-SAPOAuthScope}}
```
In my case I am getting an invalid grant error message. 
![Invalid Grant](images/sbs/InvalidGrant.jpg)

It says that *Exception was Attribute 'Recipient' of element 'SubjectConfirmationData' is invalid*. If we look at the Recipient attribute in the previous decoded token, it has something to do with this line:

```xml
   <SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
      <SubjectConfirmationData NotOnOrAfter="2025-01-01T15:26:03.619Z" Recipient="https://microsoftintegrationdemo.com:44301/sap/bc/sec/oauth2/token?sap-client=400" />
    </SubjectConfirmation>
```

In order to find out the correct Recipient, let's take a look at the security trace in the SAP system. For this open the sec_dig_tool, e.g. https://\<SAP-Server>:\<SAP-Port>>/sap/bc/webdynpro/sap/sec_diag_tool?sap-client=400&sap-language=EN#

and start the Debug trace and run the REST call tot he SAP token service again. 
![Troubelshooting](images/sbs/Troubleshooting.jpg)

Once done, open the log and you can hopefully see additional information (in my case, at the very end of the trace I can see the following entry):
![Debug Trace - Correct recipient](images/sbs/Recipient.jpg)

It looks like instead of the recipient that I had configured in my setup 
```text
Recipient from the assertion is: https://microsoftintegrationdemo.com:44301/sap/bc/sec/oauth2/token
```
 I should have used:  
```text
Requested recipient is: https://10.15.0.6:44301/sap/bc/sec/oauth2/token)
```

So lets go back to the [Enterprise App](https://portal.azure.com/#view/Microsoft_AAD_IAM/StartboardApplicationsMenuBlade/~/AppAppsPreview/menuId~/null) and change the Reply URL to the new value:
![Change Reply URL](images/sbs/ChangeReplyURL.jpg)

With this change, we can run through the Token dance again. If you check the access token you should now see the updated recipient
```xml
    <SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
      <SubjectConfirmationData NotOnOrAfter="2025-01-02T17:15:46.468Z" Recipient="https://10.15.0.6:44301/sap/bc/sec/oauth2/token?sap-client=400" />
    </SubjectConfirmation>
```


and then finally you should get an access token from the SAP system.
```json
{
    "access_token": "AA06j3aIHu-ypPU1d2li6oDoBB40-mkilSPiVrIYFinYsL0A",
  "token_type": "Bearer",
  "expires_in": "3600",
  "refresh_token": "AA06j3aIHu-ypPU1d2mC6qXb0UZsYX2jAYdV6xtTV7nhsD8L",
  "scope": "ZAPI_BUSINESS_PARTNER_0001"
}
```
![SAP Access Token](images/sbs/SuccessAccessToken.jpg)

### Call the SAP System
With this last piece we can finally take the access_token and call our SAP OData Service:
```text
### Call the SAP System with the Access Token from SAP
###
GET https://10.15.0.6:44301/sap/opu/odata/sap/API_BUSINESS_PARTNER
Authorization: Bearer {{accessTokenFromSAP.response.body.access_token}}
```
![Call SAP System](images/sbs/CallSAPSystem.jpg)

This token can then also be used to fetch for example the Top 15 Business Partners
```text
### Fetch top 15 Business parnterns from SAP
###
GET https://10.15.0.6:44301/sap/opu/odata/sap/API_BUSINESS_PARTNER/A_BusinessPartner?$top=15
Authorization: Bearer {{accessTokenFromSAP.response.body.access_token}}
```
![Business Partner Top 15](images/sbs/BusinessPartnerTop15.jpg)



## Update the APIM Policy to request access token from SAP
Now that we know that this step is also working, lets update the Policy in API Management with this new step. 
For this new step, we need additional variables: HBR-SAPOAuthClientID, HBR-SAPOAuthClientSecret & HBR-SAPOAuthScope
![Final Named Values](images/sbs/FinalNamedValues.jpg)

After that we can enhance the policy with this new call to get the SAP backend issued Bearer token using OAuth2SAMLBearerAssertion flow
```xml
        <!-- Tracing Sart 4 Get SAP backend issued Bearer token -->
        <send-request mode="new" response-variable-name="hbr-Trace-4" timeout="10" ignore-error="true">
            <set-url>{{HBR-TracingURL}}</set-url>
            <set-method>POST</set-method>
            <set-header name="Content-Type" exists-action="override">
                <value>plain/text</value>
            </set-header>
            <set-header name="X-Custom-HBR" exists-action="override">
                <value>Step 4 - Get SAP backend issued Bearer token</value>
            </set-header>
            <set-body>@{
                            var _SAPOAuthClientID = context.Variables["HBR-SAPOAuthClientID"];
                            var _SAPOAuthScope = context.Variables["HBR-SAPOAuthScope"];
                            var assertion2 = context.Variables["accessToken"];
                            return $"grant_type=urn:ietf:params:oauth:grant-type:saml2-bearer&assertion={assertion2}&client_id={_SAPOAuthClientID}&scope={_SAPOAuthScope}";
                        }</set-body>
        </send-request>
        <!-- Tracing End 4 Get SAP backend issued Bearer token -->
        <!-- Step 3: Get SAP backend issued Bearer token for presented AAD issued SAML token using OAuth2SAMLBearerAssertion flow. -->
        <send-request mode="new" response-variable-name="fetchSAPBearer" timeout="10" ignore-error="false">
            <set-url>https://{{HBR-SAPOAuthServerAdressForTokenEndpoint}}/sap/bc/sec/oauth2/token</set-url>
            <set-method>POST</set-method>
            <set-header name="Content-Type" exists-action="override">
                <value>application/x-www-form-urlencoded</value>
            </set-header>
            <!-- Provide Authentication to SAP OAuth server. Check SAP transaction code SOAUTH2 for your individual configuration -->
            <set-header name="Authorization" exists-action="override">
                <value>@{
                                var _SAPOAuthClientID = context.Variables["HBR-SAPOAuthClientID"];
                                var _SAPOAuthClientSecret = context.Variables["HBR-SAPOAuthClientSecret"];
                                return "Basic " + Convert.ToBase64String(Encoding.UTF8.GetBytes($"{_SAPOAuthClientID}:{_SAPOAuthClientSecret}"));
                            }</value>
            </set-header>
            <!--  Don't expose APIM subscription key to the backend. -->
            <set-header name="Ocp-Apim-Subscription-Key" exists-action="delete" />
            <set-body>@{
                            var _SAPOAuthClientID = context.Variables["HBR-SAPOAuthClientID"];
                            var _SAPOAuthScope = context.Variables["HBR-SAPOAuthScope"];
                            var assertion2 = context.Variables["accessToken"];
                            return $"grant_type=urn:ietf:params:oauth:grant-type:saml2-bearer&assertion={assertion2}&client_id={_SAPOAuthClientID}&scope={_SAPOAuthScope}";
                        }</set-body>
        </send-request>
        <!-- Store the access_token from the reply in SAPBearerToken  -->
        <set-variable name="SAPResponseObject" value="@(((IResponse)context.Variables["fetchSAPBearer"]).Body.As<JObject>())" />
        <set-variable name="SAPBearerToken" value="@(((JObject)context.Variables["SAPResponseObject"])["access_token"].ToString())" />
        <!-- Tracing Start 5 Now we have the Bearer Token to call the SAP OData Service -->
        <send-request mode="new" response-variable-name="hbr-Trace-5" timeout="10" ignore-error="true">
            <set-url>{{HBR-TracingURL}}</set-url>
            <set-method>POST</set-method>
            <set-header name="Content-Type" exists-action="override">
                <value>plain/text</value>
            </set-header>
            <set-header name="X-Custom-HBR" exists-action="override">
                <value>Step 5 - Final Bearer Token for SAP OData Service</value>
            </set-header>
            <set-body>@("Bearer " + (string)context.Variables["SAPBearerToken"])</set-body>
        </send-request>
        <!-- Tracing End 5 Now we have the Bearer Token to call the SAP OData Service -->
```

After this step we have the required token to call the SAP System stored in the *SAPBearerToken* variable. So the last step in the policy is to remove the previously hard-coded username and password
```xml
        <authentication-basic username="pp-sbs" password="SSOSetup!" />
```

and set the *Authentication* header and put the Bearer token there:
```xml
        <!-- Step 4: Use the now optained Bearer Token to call the actual SAP Backend OData Service  -->
        <set-header name="Authorization" exists-action="override">
            <value>@("Bearer " + (string)context.Variables["SAPBearerToken"])</value>
        </set-header>
        <!--  Don't expose APIM subscription key to the backend. -->
        <set-header name="Ocp-Apim-Subscription-Key" exists-action="delete" />
        <set-backend-service base-url="https://10.15.0.6:44301/sap/opu/odata/sap/API_BUSINESS_PARTNER" />
```

From now own each time the API is called, the token handed over from the Power Automate flow (remember Client ID: 6bee4d13-fd19-43de-b82c-4b6401d174c3), is exchanged to a SAML token on behalf of the logged in user in Power Automate, then exchanged with a access token from SAP backend with SAML Bearer Grant Type, and finally this Bearer token can be set in the Authorization header so that the call to the actual SAP OData Service succseeds.  

# Back to Power Automate - SAP OData Connector
That's it! With this final information we can trigger the Power Automate flow again. This time the logged on user is handed over to the API Management Policy. The token dance is executed and we finally call the SAP OData Service with the right mapped user. 

![Successful Power Automate FLow](images/sbs/SuccessfullPowerAutomateFlow.jpg)

# Final words
With these steps one very basic authentication flow from Power Automate to the SAP backend is covered. The real / official policy from Martin is much, much more elaborate. It is caching and refresh token handling in it, coveres aspects of load-balancing users that might request the refresh token all at once on a Monday morning and much more. You should be able to switch to the official policy without any changes to the *Named values* (just make sure to remove my "namespace" HBR-). 

Let me know how it goes!


## Additional thoughts
### OpenID
For some of the authentication flows it is also helpful to add additional configurations in the step. One is to add *openid* permissions -- this is helpful if not Power Automate (a registered app, but a human calls the service)

The reason for that is that we need to assign additional permissions to our registered app for openid. 

Go back to the registered app and select *API Permissions*. From there select *Add a permission*, select *Microsoft Graph*, *Delegated Permissions* and search and select *openid*.  
![Add OpenID Permissions](images/sbs/AddOpenID.jpg)

Afterwards click on *Grant admin consent* and test the token dance again. 


### Additional Client Permissions
Another topic is the registration of the Enterprise App. This can be helfup to reduce the pop-ups that a user will be presented with to accept and grant permissions. 

Go back to the previously [registered app](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps) (in our case hbr-sbs-power-to-apim) and under *Expose an API* add the enterprise App as a *Client Application*. 
Note: The Client Application ID d4ad39a1-ca42-45b1-a3ca-c22244219a96 in the screenshot is the Application ID of the Enterprise App that we created before. 
![Assign Enterprise App](images/sbs/AddClientApplicationID.jpg)














