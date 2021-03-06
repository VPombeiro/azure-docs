---
title: Create identity for Azure app with PowerShell | Microsoft Docs
description: Describes how to use Azure PowerShell to create an Active Directory application and service principal, and grant it access to resources through role-based access control. It shows how to authenticate application with a password or certificate.
services: azure-resource-manager
documentationcenter: na
author: tfitzmac
manager: timlt
editor: tysonn

ms.assetid: d2caf121-9fbe-4f00-bf9d-8f3d1f00a6ff
ms.service: azure-resource-manager
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: multiple
ms.workload: na
ms.date: 01/17/2017
ms.author: tomfitz

---
# Use Azure PowerShell to create a service principal to access resources
> [!div class="op_single_selector"]
> * [PowerShell](resource-group-authenticate-service-principal.md)
> * [Azure CLI](resource-group-authenticate-service-principal-cli.md)
> * [Portal](resource-group-create-service-principal-portal.md)
> 
> 

When you have an app or script that needs to access resources, you can set up an identity for the app and authenticate it with its own credentials. This approach is preferable to running the app under your own credentials because:

* You can assign permissions to the app identity that are different than your own permissions. Typically, these permissions are restricted to exactly what the app needs to do.
* You do not have to change the app's credentials if your responsibilities change. 
* You can use a certificate to automate authentication when executing an unattended script.

This topic shows you how to use [Azure PowerShell](/powershell/azureps-cmdlets-docs) to set up everything you need for an application to run under its own credentials and identity.

With PowerShell, you have two options for authenticating your AD application:

* password
* certificate

This topic shows how to use both options in PowerShell. If you intend to log in to Azure from a programming framework (such Python, Ruby, or Node.js), password authentication might be your best option. Before deciding whether to use a password or certificate, see the [Sample applications](#sample-applications) section for examples of authenticating in the different frameworks.

## Active Directory concepts
In this article, you create two objects - the Active Directory (AD) application and the service principal. The AD application is the global representation of your application. It contains the credentials (an application id and either a password or certificate). The service principal is the local representation of your application in an Active Directory. It contains the role assignment. This topic focuses on a single-tenant application where the application is intended to run within only one organization. You typically use single-tenant applications for line-of-business applications that run within your organization. In a single-tenant application, you have one AD app and one service principal.

You may be wondering - why do I need both objects? This approach makes more sense when you consider multi-tenant applications. You typically use multi-tenant applications for software-as-a-service (SaaS) applications, where your application runs in many different subscriptions. For multi-tenant applications, you have one AD app and multiple service principals (one in each Active Directory that grants access to the app). To set up a multi-tenant application, see [Developer's guide to authorization with the Azure Resource Manager API](resource-manager-api-authentication.md).

## Required permissions
To complete this topic, you must have sufficient permissions in both your Azure Active Directory and your Azure subscription. Specifically, you must be able to create an app in the Active Directory, and assign the service principal to a role. 

The easiest way to check whether your account has adequate permissions is through the portal. See [Check required permission](resource-group-create-service-principal-portal.md#required-permissions).

Now, proceed to a section for either [password](#create-service-principal-with-password) or [certificate](#create-service-principal-with-certificate) authentication.

## Create service principal with password
In this section, you perform the steps to:

* create the AD application with a password
* create the service principal
* assign the Reader role to the service principal

To quickly perform these steps, see the following cmdlets:

```powershell
$app = New-AzureRmADApplication -DisplayName "{app-name}" -HomePage "https://{your-domain}/{app-name}" -IdentifierUris "https://{your-domain}/{app-name}" -Password "{your-password}"
New-AzureRmADServicePrincipal -ApplicationId $app.ApplicationId
Start-Sleep 15
New-AzureRmRoleAssignment -RoleDefinitionName Reader -ServicePrincipalName $app.ApplicationId
```

The script sleeps for 15 seconds to allow some time for the new service principal to propagate throughout Active Directory. If your script does not wait long enough, you see an error stating: "PrincipalNotFound: Principal {id} does not exist in the directory." If you receive this error, you can rerun the cmdlet to assign it to a role.

Let's go through these steps more carefully to make sure you understand the process.

1. Sign in to your account.
   
   ```powershell
   Add-AzureRmAccount
   ```

2. Create a new Active Directory application by providing a display name, the URI that describes your application, the URIs that identify your application, and the password for your application identity.

   ```powershell   
   $app = New-AzureRmADApplication -DisplayName "exampleapp" -HomePage "https://www.contoso.org/exampleapp" -IdentifierUris "https://www.contoso.org/exampleapp" -Password "{Your_Password}"
   ```

     For single-tenant applications, the URIs are not validated.
   
     If your account does not have the [required permissions](#required-permissions) on the Active Directory, you see an error message indicating "Authentication_Unauthorized" or "No subscription found in the context."
3. Examine the new application object. 
   
   ```powershell
   $app
   ```
   
     Note in particular the `ApplicationId` property, which is needed for creating service principals, role assignments, and acquiring the access token.
   
   ```powershell
   DisplayName             : exampleapp
   ObjectId                : c95e67a3-403c-40ac-9377-115fa48f8f39
   IdentifierUris          : {https://www.contoso.org/example}
   HomePage                : https://www.contoso.org
   Type                    : Application
   ApplicationId           : 8bc80782-a916-47c8-a47e-4d76ed755275
   AvailableToOtherTenants : False
   AppPermissions          : 
   ReplyUrls               : {}
   ```
4. Create a service principal for your application by passing in the application id of the Active Directory application.
   
   ```powershell
   New-AzureRmADServicePrincipal -ApplicationId $app.ApplicationId
   ```

5. Grant the service principal permissions on your subscription. In this example, you add the service principal to the Reader role, which grants permission to read all resources in the subscription. For other roles, see [RBAC: Built-in roles](../active-directory/role-based-access-built-in-roles.md). For the `ServicePrincipalName` parameter, provide the `ApplicationId` that you used when creating the application. Before running this cmdlet, you must allow some time for the new service principal to propagate throughout Active Directory. When you run these cmdlets manually, usually enough time has elapsed between cmdlets. In a script, you should add a step to sleep between the cmdlets (like `Start-Sleep 15`). If you see an error stating: "PrincipalNotFound: Principal {id} does not exist in the directory", rerun the cmdlet.

   ```powershell   
   New-AzureRmRoleAssignment -RoleDefinitionName Reader -ServicePrincipalName $app.ApplicationId
   ```

    If your account does not have sufficient permissions to assign a role, you see an error message. The message states your account "does not have authorization to perform action 'Microsoft.Authorization/roleAssignments/write' over scope '/subscriptions/{guid}'." 

That's it! Your AD application and service principal are set up. The next section shows you how to log in with the credential through PowerShell. If you want to use the credential in your code application, you can jump to the [Sample applications](#sample-applications). 

### Provide credentials through PowerShell
Now, you need to log in as the application to perform operations.

1. Create a `PSCredential` object that contains your credentials by running the `Get-Credential` command. You need the `ApplicationId` before running this command so make sure you have that available to paste.

   ```powershell   
   $creds = Get-Credential
   ```

2. You are prompted you to enter your credentials. For the user name, use the `ApplicationId` that you used when creating the application. For the password, use the one you specified when creating the account.
   
     ![enter credentials](./media/resource-group-authenticate-service-principal/arm-get-credential.png)
3. Whenever you sign in as a service principal, you need to provide the tenant id of the directory for your AD app. A tenant is an instance of Active Directory. If you only have one subscription, you can use:

   ```powershell   
   $tenant = (Get-AzureRmSubscription).TenantId
   ```
   
     If you have more than one subscription, specify the subscription where your Active Directory resides. For more information, see [How Azure subscriptions are associated with Azure Active Directory](../active-directory/active-directory-how-subscriptions-associated-directory.md).

   ```powershell
   $tenant = (Get-AzureRmSubscription -SubscriptionName "Contoso Default").TenantId
   ```

4. Log in as the service principal by specifying that this account is a service principal and by providing the credentials object. 
   
   ```powershell
   Add-AzureRmAccount -Credential $creds -ServicePrincipal -TenantId $tenant
   ```
   
     You are now authenticated as the service principal for the Active Directory application that you created.

### Save access token to simplify log in
To avoid providing the service principal credentials every time it needs to log in, you can save the access token.

1. To use the current access token in a later session, save the profile.
   
   ```powershell
   Save-AzureRmProfile -Path c:\Users\exampleuser\profile\exampleSP.json
   ```
   
     Open the profile and examine its contents. Notice that it contains an access token. 
2. Instead of manually logging in again, simply load the profile.
   
   ```powershell
   Select-AzureRmProfile -Path c:\Users\exampleuser\profile\exampleSP.json
   ```

  > [!NOTE]
  > The access token expires, so using a saved profile only works for as long as the token is valid.
  >  

Alternatively, you can invoke REST operations from PowerShell to log in. From the authentication response, you can retrieve the access token for use with other operations. For an example of retrieving the access token by invoking REST operations, see [Generating an Access Token](resource-manager-rest-api.md#generating-an-access-token).

## Create service principal with certificate
In this section, you perform the steps to:

* create a self-signed certificate
* create the AD application with the certificate
* create the service principal
* assign the Reader role to the service principal

To quickly perform these steps with Azure PowerShell 2.0 on Windows 10 or Windows Server 2016 Technical Preview, see the following cmdlets:

```powershell
$cert = New-SelfSignedCertificate -CertStoreLocation "cert:\CurrentUser\My" -Subject "CN=exampleapp" -KeySpec KeyExchange
$keyValue = [System.Convert]::ToBase64String($cert.GetRawCertData())
$app = New-AzureRmADApplication -DisplayName "exampleapp" -HomePage "https://www.contoso.org" -IdentifierUris "https://www.contoso.org/example" -CertValue $keyValue -EndDate $cert.NotAfter -StartDate $cert.NotBefore
New-AzureRmADServicePrincipal -ApplicationId $app.ApplicationId
Start-Sleep 15
New-AzureRmRoleAssignment -RoleDefinitionName Reader -ServicePrincipalName $app.ApplicationId
```

The script sleeps for 15 seconds to allow some time for the new service principal to propagate throughout Active Directory. If your script does not wait long enough, you see an error stating: "PrincipalNotFound: Principal {id} does not exist in the directory." If you receive this error, you can rerun the cmdlet to assign it to a role.

Let's go through these steps more carefully to make sure you understand the process. This article also shows how to accomplish the tasks when using earlier versions of Azure PowerShell or operating systems.

### Create the self-signed certificate
The version of PowerShell available with Windows 10 and Windows Server 2016 Technical Preview has an updated `New-SelfSignedCertificate` cmdlet for generating a self-signed certificate. Earlier operating systems have the New-SelfSignedCertificate cmdlet but it does not offer the parameters needed for this topic. Instead, you need to import a module to generate the certificate. This topic shows both approaches for generating the certificate based on the operating system you have. 

* If you have **Windows 10 or Windows Server 2016 Technical Preview**, run the following command to create a self-signed certificate: 
   
  ```powershell
  $cert = New-SelfSignedCertificate -CertStoreLocation "cert:\CurrentUser\My" -Subject "CN=exampleapp" -KeySpec KeyExchange
  ```
* If you **do not have Windows 10 or Windows Server 2016 Technical Preview**, you need to download the [Self-signed certificate generator](https://gallery.technet.microsoft.com/scriptcenter/Self-signed-certificate-5920a7c6/) from Microsoft Script Center. Extract its contents and import the cmdlet you need.

  ```powershell  
  # Only run if you could not use New-SelfSignedCertificate
  Import-Module -Name c:\ExtractedModule\New-SelfSignedCertificateEx.ps1
  ```
  
     Then, generate the certificate.
  
  ```powershell
  $cert = New-SelfSignedCertificateEx -Subject "CN=exampleapp" -KeySpec "Exchange" -FriendlyName "exampleapp"
  ```

You have your certificate and can proceed with creating your AD app.

### Create the Active Directory app and service principal
1. Retrieve the key value from the certificate.
   
   ```powershell
   $keyValue = [System.Convert]::ToBase64String($cert.GetRawCertData())
   ```
2. Sign in to your Azure account.
   
   ```powershell
   Add-AzureRmAccount
   ```
3. Create a new Active Directory application by providing a display name, the URI that describes your application, the URIs that identify your application, and the password for your application identity.
   
     If you have Azure PowerShell 2.0 (August 2016 or later), use the following cmdlet:

   ```powershell   
   $app = New-AzureRmADApplication -DisplayName "exampleapp" -HomePage "https://www.contoso.org" -IdentifierUris "https://www.contoso.org/example" -CertValue $keyValue -EndDate $cert.NotAfter -StartDate $cert.NotBefore      
   ```
   
    If you have Azure PowerShell 1.0, use the following cmdlet:

   ```powershell
   $app = New-AzureRmADApplication -DisplayName "exampleapp" -HomePage "https://www.contoso.org" -IdentifierUris "https://www.contoso.org/example" -KeyValue $keyValue -KeyType AsymmetricX509Cert  -EndDate $cert.NotAfter -StartDate $cert.NotBefore
   ```
   
    For single-tenant applications, the URIs are not validated.
   
    If your account does not have the [required permissions](#required-permissions) on the Active Directory, you see an error message indicating "Authentication_Unauthorized" or "No subscription found in the context."
   
    Examine the new application object. 
   
   ```powershell
   $app
   ```
   
    Notice the **ApplicationId** property, which is needed for creating service principals, role assignments, and acquiring access tokens.

   ```powershell
   DisplayName             : exampleapp
   ObjectId                : c95e67a3-403c-40ac-9377-115fa48f8f39
   IdentifierUris          : {https://www.contoso.org/example}
   HomePage                : https://www.contoso.org
   Type                    : Application
   ApplicationId           : 8bc80782-a916-47c8-a47e-4d76ed755275
   AvailableToOtherTenants : False
   AppPermissions          : 
   ReplyUrls               : {}
   ```
4. Create a service principal for your application by passing in the application id of the Active Directory application.
   
   ```powershell
   New-AzureRmADServicePrincipal -ApplicationId $app.ApplicationId
   ```
5. Grant the service principal permissions on your subscription. In this example, you add the service principal to the Reader role, which grants permission to read all resources in the subscription. For other roles, see [RBAC: Built-in roles](../active-directory/role-based-access-built-in-roles.md). For the `ServicePrincipalName` parameter, provide the `ApplicationId` that you used when creating the application. Before running this cmdlet, you must allow some time for the new service principal to propagate throughout Active Directory. When you run these cmdlets manually, usually enough time has elapsed between cmdlets. In a script, you should add a step to sleep between the cmdlets (like `Start-Sleep 15`). If you see an error stating: "PrincipalNotFound: Principal {id} does not exist in the directory", rerun the cmdlet.
   
   ```powershell
   New-AzureRmRoleAssignment -RoleDefinitionName Reader -ServicePrincipalName $app.ApplicationId
   ```
   
    If your account does not have sufficient permissions to assign a role, you see an error message. The message states your account "does not have authorization to perform action 'Microsoft.Authorization/roleAssignments/write' over scope '/subscriptions/{guid}'."

That's it! Your AD application and service principal are set up. The next section shows you how to log in with certificate through PowerShell.

### Provide certificate through automated PowerShell script
Whenever you sign in as a service principal, you need to provide the tenant id of the directory for your AD app. A tenant is an instance of Active Directory. If you only have one subscription, you can use:

```powershell
$tenant = (Get-AzureRmSubscription).TenantId
```

If you have more than one subscription, specify the subscription where your Active Directory resides. For more information, see [Administer your Azure AD directory](../active-directory/active-directory-administer.md).

```powershell
$tenant = (Get-AzureRmSubscription -SubscriptionName "Contoso Default").TenantId
```

To authenticate in your script, specify the account is a service principal and provide the certificate thumbprint, the application id, and tenant id. To automate your script, you can store these values as environment variables and retrieve them during execution, or you can include them in your script.

```powershell
Add-AzureRmAccount -ServicePrincipal -CertificateThumbprint $cert.Thumbprint -ApplicationId $app.ApplicationId -TenantId $tenant
```

You are now authenticated as the service principal for the Active Directory application that you created.

## Change credentials

To change the credentials for an AD app, either because of a security compromise or a credential expiration, use the [Remove-AzureRmADAppCredential](/powershell/resourcemanager/azurerm.resources/v3.3.0/remove-azurermadappcredential) and [New-AzureRmADAppCredential](/powershell/resourcemanager/azurerm.resources/v3.3.0/new-azurermadappcredential) cmdlets.

To remove all the credentials for an application, use:

```powershell
Remove-AzureRmADAppCredential -ApplicationId 8bc80782-a916-47c8-a47e-4d76ed755275 -All
```

To add a password, use:

```powershell
New-AzureRmADAppCredential -ApplicationId 8bc80782-a916-47c8-a47e-4d76ed755275 -Password p@ssword!
```

To add a certificate value, create a self-signed certificate as shown in this topic. Then, use:

```powershell
New-AzureRmADAppCredential -ApplicationId 8bc80782-a916-47c8-a47e-4d76ed755275 -CertValue $keyValue -EndDate $cert.NotAfter -StartDate $cert.NotBefore
```

## Sample applications
The following sample applications show how to log in as the service principal.

**.NET**

* [Deploy an SSH Enabled VM with a Template with .NET](https://azure.microsoft.com/documentation/samples/resource-manager-dotnet-template-deployment/)
* [Manage Azure resources and resource groups with .NET](https://azure.microsoft.com/documentation/samples/resource-manager-dotnet-resources-and-groups/)

**Java**

* [Getting Started with Resources - Deploy Using Azure Resource Manager Template - in Java](https://azure.microsoft.com/documentation/samples/resources-java-deploy-using-arm-template/)
* [Getting Started with Resources - Manage Resource Group - in Java](https://azure.microsoft.com/documentation/samples/resources-java-manage-resource-group//)

**Python**

* [Deploy an SSH Enabled VM with a Template in Python](https://azure.microsoft.com/documentation/samples/resource-manager-python-template-deployment/)
* [Managing Azure Resource and Resource Groups with Python](https://azure.microsoft.com/documentation/samples/resource-manager-python-resources-and-groups/)

**Node.js**

* [Deploy an SSH Enabled VM with a Template in Node.js](https://azure.microsoft.com/documentation/samples/resource-manager-node-template-deployment/)
* [Manage Azure resources and resource groups with Node.js](https://azure.microsoft.com/documentation/samples/resource-manager-node-resources-and-groups/)

**Ruby**

* [Deploy an SSH Enabled VM with a Template in Ruby](https://azure.microsoft.com/documentation/samples/resource-manager-ruby-template-deployment/)
* [Managing Azure Resource and Resource Groups with Ruby](https://azure.microsoft.com/documentation/samples/resource-manager-ruby-resources-and-groups/)

## Next Steps
* For detailed steps on integrating an application into Azure for managing resources, see [Developer's guide to authorization with the Azure Resource Manager API](resource-manager-api-authentication.md).
* For a more detailed explanation of applications and service principals, see [Application Objects and Service Principal Objects](../active-directory/active-directory-application-objects.md). 
* For more information about Active Directory authentication, see [Authentication Scenarios for Azure AD](../active-directory/active-directory-authentication-scenarios.md).

