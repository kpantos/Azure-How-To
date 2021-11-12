# Expose a Kerberos secured on prem API with Azure APIM

Application Proxy is a feature of Azure AD that enables users to access on-premises web applications from a remote client. Application Proxy includes both the Application Proxy service which runs in the cloud, and the Application Proxy connector which runs on an on-premises server. Azure AD, the Application Proxy service, and the Application Proxy connector work together to securely pass the user sign-on token from Azure AD to the web application.

Application Proxy works with:

* Web applications that use [Integrated Windows authentication](https://docs.microsoft.com/en-us/azure/active-directory/app-proxy/application-proxy-configure-single-sign-on-with-kcd) for authentication
* Web applications that use form-based or [header-based](https://docs.microsoft.com/en-us/azure/active-directory/app-proxy/application-proxy-configure-single-sign-on-with-headers) access
* Web APIs that you want to expose to rich applications on different devices
* Applications hosted behind a [Remote Desktop Gateway](https://docs.microsoft.com/en-us/azure/active-directory/app-proxy/application-proxy-integrate-with-remote-desktop-services)
* Rich client apps that are integrated with the Microsoft Authentication Library (MSAL)

Application Proxy supports single sign-on. For more information on supported methods, see [Choosing a single sign-on method](https://docs.microsoft.com/en-us/azure/active-directory/manage-apps/sso-options#choosing-a-single-sign-on-method).

Application Proxy is recommended for giving remote users access to internal resources. Application Proxy replaces the need for a VPN or reverse proxy. It is not intended for internal users on the corporate network.  These users who unnecessarily use Application Proxy can introduce unexpected and undesirable performance issues.

You can read more about Application Proxy here [https://docs.microsoft.com/en-us/azure/active-directory/app-proxy/application-proxy](https://docs.microsoft.com/en-us/azure/active-directory/app-proxy/application-proxy)

In this walkthrough you will see how you can use Azure API Manager to safely expose Kerberos secured on premises APIs with the use of Azure AD Application Proxy.

## Pre-Requisites ##
- An Azure subscription with contributor rights in a resource group 
- Access to an Azure AD tenant with global admin rights
- Azure CLI version 2.0.81 or later

## Deploy on-prem infrastructure (optional)
You are going to simulate the on premise runing infrastructure in Azure. So you are going to deploy a new AD and a sample Web application and use Kerberos authentication to access it.

> **NOTE**
>
> You will need to follow the next steps of this section if you don't already have an on prem domain and a web application or API using Kerberos authentication.

1. Deploy AD controllers
   
   There are multiple different ways to provision a domain for your organization, probably the easiest one though, since this walkthrough is using Azure to simulate the on-premise environment, is to use one of the available ARM templates from the [Azure Quickstart Templates](https://azure.microsoft.com/en-us/resources/templates/?term=Domain+) Gallery.

   For the purposese of this guide you will be using the [Create a new AD Domain with 2 DCs using Availability Zones](https://azure.microsoft.com/en-us/resources/templates/active-directory-new-domain-ha-2-dc-zones/) one, which creates 2 VMs in separate Availability Zones to be AD DCs (primary and backup) for a new Forest and Domain.

   > **NOTE**
   >
   > This Azure Resource Manager template was created by a member of the community and not by Microsoft. Each Resource Manager template is licensed to you under a license agreement by its owner, not Microsoft. Microsoft is not responsible for Resource Manager templates provided and licensed by community members and does not screen for security, compatibility, or performance. Community Resource Manager templates are not supported under any Microsoft support program or service, and are made available AS IS without warranty of any kind.

   Using your WSL terminal after [installing the Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt) or the Azure portal Cloud Shell run the following commands to provision your domain.

   ```bash
   az group create --name <resource-group-name> --location <resource-group-location> 
   az group deployment create --resource-group <my-resource-group> --template-uri https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/application-workloads/active-directory/active-directory-new-domain-ha-2-dc-zones/azuredeploy.json

   ```
   You will have to provide values for the following parameters (bold are required) either through a parameters file or through command line arguments
      
   | Parameter Name | Description |
   | -------------- | ----------- |
   | **adminUsername**	| The name of the Administrator of the new VM and Domain |
   | location       | Location for the VM, only certain regions support zones during preview. |
   | **adminPassword**  | The password for the Administrator account of the new VM and Domain |
   | **domainName**     | The FQDN of the AD Domain created |
   | **dnsPrefix**      | The DNS prefix for the public IP address used by the Load Balancer |
   | vmSize         | Size of the VM for the controller |


2. Deploy Web Server

    Same as with the Domain controllers you will be using an available ARM template from the [Azure Quickstart Templates](https://azure.microsoft.com/en-us/resources/templates/?term=Domain+) Gallery.

    For the purposese of this guide you will be using the [IIS Server using DSC extension on a Windows VM](https://azure.microsoft.com/en-us/resources/templates/dsc-extension-iis-server-windows-vm/) one, which creates a Windows VM and sets up an IIS server using the DSC extension.

    ```bash
    az group deployment create --resource-group <my-resource-group> --template-uri https://github.com/kopantos/Azure-How-To/blob/main/kerberos-onprem-api-with-apim/arm/ContosoWebsite.ps1.zip 
    ```
    
    You will have to provide values for the following parameters (bold are required) either through a parameters file or through command line arguments
      
   | Parameter Name | Description |
   | -------------- | ----------- |
   | diskType       | Type of the Storage for disks |
   | vmName         | Name of the VM |
   | vmSize         | Size of the VM |
   | imageSKU       | Image SKU |
   | **adminUsername**  | Admin username |
   | **adminPassword**  | Admin password |
   | **virtualNetworkName** | Virtual network name to deploy the VM to.
   | **subnetName** | The subnet name to deploy the VM to.
   | location       | Location for all resources. |

   Once the web server has been provisioned you will have to domain join it to the Domain you created in the previous step. Use RDP to connect to the web server and start a new powershell terminal. Use the Add-Computer cmdlet to join the computer to your domain and then restart the VM. When prompted, enter the domain admin credentials you specified at the domain provisioning step:

   ```powershell
   Add-Computer -DomainName CONTOSO.local -Restart
   ```

3. Deploy Web Application
   
   1. Use RDP to connect to the web server, download the following sample app https://github.com/kopantos/Azure-How-To/blob/main/kerberos-onprem-api-with-apim/deployment/SampleAPI.zip and unzip the contentes in the directory ```C:\inetpub\wwwroot\contoso-api```

   2. From IIS Manager console right click on sites and create a new site 
      ![New Web Site wizard](images/new-website.png)

   3. Configure site authentication by disabling all authentication methods except windows. Select providers from the right hand side menu and make sure Negotiate (for kerberos) and NTLM providers are selected
      ![Site authentication settings](images/site-authentication.png)

   4. (Optional) From the authorization settings configure the users that should have access to the web application and API
      ![Site authorization settings](images/site-authorization.png)

   Tring to access the application at http://web-server.contoso.local/ from the browser now should challenge you for a domain account and providing the right credentials should allow you to browse to the default web page of the .Net sample application.

   ![Web site sign in](images/sign-in.png)

## Setup AD Sync
The first step you'll have to take in order to efectively use application proxy for your on premise application is sync your on premise AD with your Azure AD.
For instruction on how to connect your on premise AD to your Azure AD use the https://docs.microsoft.com/en-us/azure/active-directory/hybrid/how-to-connect-install-express.

### Configure SSO for different identities
Configure Azure AD Connect settings so the main identity is the email address (mail). This is done as part of the customize process, by changing the **User Principal Name** field in the sync settings. These settings also determine how users log in to Office365, Windows10 devices, and other applications that use Azure AD as their identity store.  
   ![Identifying users screenshot - User Principal Name dropdown](https://docs.microsoft.com/en-us/azure/active-directory/app-proxy/media/application-proxy-configure-single-sign-on-with-kcd/app_proxy_sso_diff_id_connect_settings.png)  

## Configure the Service Principal Name (SPN)
A **service principal name** (SPN) is a unique identifier of a service instance. SPNs are used by Kerberos authentication to associate a service instance with a service logon account. This allows a client application to request that the service authenticate an account even if the client does not have the account name.

If your web site uses a different host name than the default server name ```web-server.contoso.local```, you will have to add the custom web site host name as an SPN alias for this server/AD object. 

[Read more on how to setup an SPN](spn.md)

For more information on SPN have a look at [TechNet](https://social.technet.microsoft.com/wiki/contents/articles/717.service-principal-names-spn-setspn-syntax.aspx)


## Setup Azure Application Proxy

1. **Install the connector**: Download the service from the Application Proxy configuration page top bar of your Azure AD. Run the connector installer on a server which has network access to the services you wish to publish through the application proxy – it doesn’t have to be physically in the same location, it can even be on an Azure virtual machine so long as you have a VPN set up. The connector is supported on Windows Server 2012 R2 and newer editions and you can have multiple connectors deployed in groups to achieve HA and scale.

      > **NOTES**
      >
      > The server running the Connector and the server running the app are domain joined and part of the same domain or trusting domains. For more information on domain join, see Join a Computer to a Domain.
      >
      > The server running the Connector has access to read the ```TokenGroupsGlobalAndUniversal``` attribute for users. This default setting might have been impacted by security hardening the environment.

      #### Configure AD for Connector and application server in the same domain
      1. In Active Directory, go to **Tools** > **Users and Computers**.
      2. Select the server running the connector.
      3. Right-click and select **Properties** > **Delegation**.
      4. Select **Trust this computer for delegation to specified services only**. 
      5. Select **Use any authentication protocol**.
      6. Under **Services to which this account can present delegated credentials** add the value for the SPN identity of the application server. This enables the Application Proxy Connector to impersonate users in AD against the applications defined in the list.

         ![Connector-SVR Properties window screenshot](https://docs.microsoft.com/en-us/azure/active-directory/app-proxy/media/application-proxy-configure-single-sign-on-with-kcd/properties.jpg)

      More information on [Kerberos Constrained Delegation for single sign-on (SSO) to your apps with Application Proxy](https://docs.microsoft.com/en-us/azure/active-directory/app-proxy/application-proxy-configure-single-sign-on-with-kcd)

2. **Enable the connector**: Once the setup is complete you’ll need to go back to the Azure portal and the Application Proxy page, if you’ve still got it open then give it a refresh. You’ll see that there’s now a connector group called Default, and inside it is a connector which should show its status as Active. Click on Enable application proxy in the toolbar at the top.

   ![Application Proxy View](images/application-proxy.png)

3. **Create an Application**: Now you will need to create the app you want to access through the App Proxy. Click on Configure an app at the top.

   1. Work your way through the fields, the basic information you’ll need for this is the **internal URL** for the service you want to publish through the proxy, 
   
   2. Next enter the **external URL** for your application, 
   
      > **NOTE**: You’ll have to configure DNS and upload an SSL certificate if you use anything but the default msappproxy.net domain. A PFX file containing the certificate and private key works here. This address is what your end users will be using to access the internal service.
   
   3. Select the **authentication** you want to use. For this walkthrough you will use Azure Active Directory to define who has permission to access this application in Azure, and those permitted access will be allowed through. 
   
      > **NOTE**: If your application doesn’t know how to interface with this then you may end up with double authentication prompts for your users – one from Azure, and one from the internal application. The alternative is to use passthrough, where the authentication is handled by the internal application rather than by Azure.

   4. Select the **Default Connector Group** that you have provisioned

      Accept the default values for the other settings and then click Add at the top toolbar

      ![Add AD proxy application](images/add-application.png)

   Once the application is provisioned you can find it by navigating to Azure AD > Enterprise Applications > All applications, and searching for it. You will first need to assign users and/or groups to access it. From the Getting Started icons click on the first one and **add a user that is synced from the on premise AD**.

   ![Proxy application getting started](images/getting-started.png)

   Next you will need to enable Single Sign On. Click the **Set up single sign on** button from the getting started screen to define that you will be using **Windows Integrated Authentication** for this application which allows the Application Proxy Connectors permission in Active Directory to impersonate users to the published application.
   
   ![Single Sign On configuration](images/SingleSignOn.png)

   Enter the **SPN your on premise application is using** see section [SPN](##Configure-the-Service-Principal-Name-(SPN)) for more information.

   Finally select to **use the On-Premises user principal name** part of the Token as the delegated Login Identity for the on premise AD 

   ![Service Principal Name configuration](images/spn-setup.png)

## Deploy and configure Azure APIM
Azure API Management (APIM) helps organizations publish APIs to external, partner, and internal developers to unlock the potential of their data and services. API Management provides the core competencies to ensure a successful API program through developer engagement, business insights, analytics, security, and protection. APIM enables you to create and manage modern API gateways for existing backend services hosted anywhere.

 ### 1. Follow [this quickstart](https://docs.microsoft.com/en-us/azure/api-management/get-started-create-service-instance) to create a new API Management instance using the Azure portal. ###

 ### 2. Create a new Application registration ###
This app registration is going to be used by APIM to access the proxied application (Enterprise Application). To create the app registration 

For details about app registration, see [Quickstart: Configure an application to expose a web API](https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/active-directory/develop/quickstart-configure-app-expose-web-apis.md).

1. In the Azure portal, navigate to Azure AD and search for and select App registrations.

2. Select New registration.

3. When the **Register an application** page appears, enter your application's registration information:

   - In the Name section, enter a meaningful application name that will be displayed to users of the app, such as backend-app.
   - In the Supported account types section, leave the default Web option.

4. Leave the **Redirect URI** section empty.

5. Select **Register** to create the application.

   ![New Application registration](images/application-registration.png)

6. On the app Overview page, find the **Application (client) ID** value and record it for later.

7. Under the **Manage** section of the side menu, select **Expose an API** and set the **Application ID URI** with the default value. Record this value for later.

   ![Expose API in application registration](images/expose-api-appregistration.png)

8. Select the **API permitions** from the side menu and click **Add a permition**. From the blade that opens up select the **APIs my Organisation uses** tab and search for and select the Enterprise application that has been created as part of the proxying of the on premises application.

   ![https://link](images/api-permition-appregistration.png)

9. Select to **impersonate** the user that accesses the Enterprise Application (proxied application) through this application registration. Click **Add permisssions** to complete the wizard.

   ![https://link](images/impersonation-appregistration.png)

10. Click the **Grant Admin Concent** button at the top bar to complete the api permissions assignement.

11. Create a **client secret** for this application to use in a subsequent step.

   - Under the **Manage** section of the side menu, select **Certificates & secrets**.
   - Under **Client secrets**, select **New client secret**.
   - Under **Add a client secret**, provide a **Description** and choose when the key should expire.
   - Select **Add**.

   When the secret is created, **note the key value** for use in a subsequent step.

 ### 3. Configure OAuth authentication for APIM
At this point, you have created your applications in Azure AD, and have granted proper permissions to allow any client-app to call the backend-app.

Next you will need to enable OAuth 2.0 user authorization

1. In the Azure portal, find the **Authorization endpoint URL** and **Token endpoint URL** and save them for later. 
    1. Open the **App registrations** page. 
    1. Select **Endpoints**.
    1. Copy the **OAuth 2.0 Authorization Endpoint** and the **OAuth 2.0 Token Endpoint**. 

1. Browse to your API Management instance.

1. Under the **Developer portal** section in the side menu, select **OAuth 2.0 + OpenID Connect**. 

1. Under the **OAuth 2.0** tab, select **Add**.

1. Provide a **Display name** and **Description**.

1. For the **Client registration page URL**, enter a placeholder value, such as `http://localhost`. 
    * The **Client registration page URL** points to a page where users create and configure their own accounts supported by OAuth 2.0 providers. 
    * We use a placeholder, since, in this example, users do not create and configure their own accounts.

1. For **Authorization grant types**, select **Authorization code**.

1. Specify the **Authorization endpoint URL** and **Token endpoint URL** you saved earlier: 
    1. Copy and paste the **OAuth 2.0 Authorization Endpoint** into the **Authorization endpoint URL** text box. 
    1. Select **POST** under Authorization request method.
    1. Enter the **OAuth 2.0 Token Endpoint**, and paste it into the **Token endpoint URL** text box. 
        * If you use the **v1** endpoint:
          * Add a body parameter named **resource**.
          * Enter the back-end app **Application ID** for the value.
        * If you use the **v2** endpoint:
          * Use the back-end app scope you created in the **Default scope** field.
          * Set the value for the [`accessTokenAcceptedVersion`](../active-directory/develop/reference-app-manifest.md#accesstokenacceptedversion-attribute) property to `2` in your [application manifest](../active-directory/develop/reference-app-manifest.md).      

   >[!IMPORTANT]
   >
   > While you can use either **v1** or **v2** endpoints, we recommend using v2 endpoints. 

1. Specify the client app credentials:
    * For **Client ID**, use the **Application ID** of the client-app.
    * For **Client secret**, use the key you created for the client-app earlier. 

1. Make note of the **Redirect URI** for the authorization code grant type.

1. Select **Create**.

1. Return to your client-app registration. 
 
1. Under **Manage**, select **Authentication**.

1. Under **Platform configurations**:
    * Select the  **Web** platform you created eaarlier. 
    * Paste the redirect URI you saved earlier under **Redirect URIs**.
    * Click on **Configure** button to save.

Now that the Developer Console can obtain access tokens from Azure AD via your OAuth 2.0 authorization server, enable OAuth 2.0 user authorization for your API. This enables the Developer Console to know that it needs to obtain an access token on behalf of the user, before making calls to your API.

### 4. Add your first API
Time to expose your on-premise application through APIM. 

1. Browse to your API Management instance.

1. Under the **APIs** section in the side menu, select **APIs**

1. Click on **HTTP** to manually define an API since you don't have an API definition

1. Select **Full** to enter all values for the **new API definition**
   * Enter a **Display name** for the API
   * Enter the programmatic name of the API
   * Set the **Web Service Url** the APIM will be calling. This should be the proxy application dns name with the api path e.g. ```https://webapi-kpanlab.msappproxy.net/api/```
   * Select **HTTPS** for the URL scheme
   * Select all available **Products**
   * Click **Create**

   ![New API Definition wizard](images/new-api-definition.png)

1. Click on the **Settings** tab of the newly created API and under the **Security** Section select the OAuth server you created in the previous step.
   ![Select OAuth for API](images/API-OAuth.png)

1. Add an **Operation** for the newly created API
   ![Add operation to API](images/add-operation.png)
### 5. Configure a JWT validation policy to pre-authorize requests
You will need to Pre-authorize requests in API Management with the Validate JWT policy, by validating the access tokens of each incoming request. If a request does not have a valid token, API Management blocks it.

Click on the design tab of the newly created API and edit the inbound policy with the policy editor

![Policy editor](images/policy-editor.png)

The following example policy, when added to the <inbound> policy section, checks the value of the audience claim in an access token obtained from Azure AD, and returns an error message if the token is not valid.

   ```xml
   <validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Unauthorized. Access token is missing or invalid.">
      <openid-config url="https://login.microsoftonline.com/<YOUR-AAD-DOMAIN>/.well-known/openid-configuration" />
      <audiences>
            <audience>[Enter the application ID of your Enterprise application here]</audience>
      </audiences>
      <required-claims>
            <claim name="appid" match="all">
               <value>[Enter the application id of your APIM application registration here]</value>
            </claim>
      </required-claims>
   </validate-jwt>
   ```

## Testing the application
1. Get an **Auth Token**

   - Issue a GET request at:
      ```
      https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/authorize?client_id=<CLIENT_ID>&response_type=code&redirect_uri=http://localhost&response_mode=query&scope=openid%20offline_access&state=12345
      ```
      > **NOTE**: This will prompt you to login. Use a user that has access to the enterprise application that you configured as well as the on premise API.

   - Copy the code returned as a querystring argument in the response
      > Example
      >
      >```http://localhost/?code=```
      ***0.AUgAUxhucWtI3k6VKgHihM2XA3VzVzjkTsNHp-1Blvlll99IAGM.AQABAAIAAAD--DLA3VO7QrddgJg7WevrO7yBuIEfJH7xNEC_k__B0nb7AYz-RTWmr19aZ7vmss5HN8OyJ7tdEo0KaIDn1Oe_MBPGKPhEAHAdZQB3Dbsa2ZqmblcqFYH9m3dAtZoWFa7kVjtaDadhtd2Qhdy-G8VUZUhQlmcNV0diF6UdYhxgqsvFaH75_CZ_HdlXwf0uvVNeuCphGFD1UyNl6p2keP59w4wkLnUeQ9ZoYhqpMWUjEUDacagvWA4WCSoPwJnk_kqkmxK6MoLkKJUVFKi__s0n738d4SmVW7ruUybXmvz3tEYCSTn9dWafaBBlrUiXTanrKw2cQ0cKvXSzEzzqUVeMG9YGGqwCjxQ1lFXOhPSDaHKHoGW0YzkW3SyIwiGxSllSbCHCoMBNuNH7o45NvG5CrScE6LGWt91ovJL_I1vGYTR0QAlgCoZk5LXXTyAjHO5kAjZItBhlH3YbuRRRHOokazolQx-3a6n06tvsbHPmIszNZlm2uGcS-1CmLAaI1ZgCKhg1npwgp_22CusDfOUCrbmEN7bTnIYOgbHjoxRmjTk27IMH3JuRbFZipVCM5SZqzeg-ru-VG40f5yyUs8uJd3Tl6S6CWtffxBMD49RM1SAA***
      ```&state=12345&session_state=a8372839-965a-4f7b-a63c-c9deaedac9f9#```
   
1. Get a **Token** by sending a post request to:
      ```https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token```
      
      Passing the following as x-www-form-urlencoded arguments :

      ```
      client_id:<CLIENT_APPLICATION_REGISTRATION_ID>
      scope:<PROXIED_ENTERPRISE_APPLICATION_ID>/.default
      redirect_uri:http://localhost
      grant_type:authorization_code
      client_secret:<APPLICATION_REGISTRATION_SECRET>
      code:<THE CODE RETURNED BY THE PREVIOUS REQUEST>      
      ```
      This should return a json document with the Bearer tokens you can use to access the APIM management APIs

      ```json
      {
         "token_type": "Bearer",
         "scope": "0c446fa9-b468-40a6-8893-196dabb00118/user_impersonation 0c446fa9-b468-40a6-8893-196dabb00118/.default",
         "expires_in": 3937,
         "ext_expires_in": 3937,
         "access_token": "...",
         "refresh_token": "...",
         "id_token": "..."
      }
      ```
1. Access the API that you configured by sending a GET request at the endpoint that you configured earlier passing the bearer token returned.


   

