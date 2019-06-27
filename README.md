# Use Key Vault from App Service with Managed Service Identity

## Background
For Service-to-Azure-Service authentication, the approach so far involved creating an Azure AD application and associated credential, and using that credential to get a token. The sample [here](https://docs.microsoft.com/en-us/azure/key-vault/key-vault-use-from-web-application) shows how this approach is used to authenticate to Azure Key Vault from a Web App. While this approach works well, there are two shortcomings:
1. The Azure AD application credentials are typically hard coded in source code. Developers tend to push the code to source repositories as-is, which leads to credentials in source.
2. The Azure AD application credentials expire, and so need to be renewed, else can lead to application downtime.

With [Managed Service Identity (MSI)](https://docs.microsoft.com/en-us/azure/app-service/app-service-managed-service-identity), both these problems are solved. This sample shows how a Web App can authenticate to Azure Key Vault without the need to explicitly create an Azure AD application or manage its credentials. 

>Here's another sample that shows how to programatically deploy an ARM template from a .NET Console application running on an Azure VM with a Managed Service Identity (MSI) - [https://github.com/Azure-Samples/windowsvm-msi-arm-dotnet](https://github.com/Azure-Samples/windowsvm-msi-arm-dotnet)

>Here's another .NET Core sample that shows how to programmatically call Azure Services from an Azure Linux VM with a Managed Service Identity (MSI). - [https://github.com/Azure-Samples/linuxvm-msi-keyvault-arm-dotnet](https://github.com/Azure-Samples/linuxvm-msi-keyvault-arm-dotnet)

## Prerequisites
To run and deploy this sample, you need the following:
1. An Azure subscription to create an App Service and a Key Vault.

## Step 1: Create an App Service with a Managed Service Identity (MSI)
<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure-Samples%2Fapp-service-msi-keyvault-dotnet%2Fmaster%2Fazuredeploy.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>

Use the "Deploy to Azure" button to deploy an ARM template to create the following resources:
1. App Service with MSI.
2. Key Vault with a secret, and an access policy that grants the App Service access to **Get Secrets**.
>Note: When filling out the template you will see a textbox labelled 'Key Vault Secret'. Enter a secret value there. A secret with the name 'secret' and value from what you entered will be created in the Key Vault.

Review the resources created using the Azure portal. You should see an App Service and a Key Vault. View the access policies of the Key Vault to see that the App Service has access to it. 

## Step 2: Grant yourself data plane access to the Key Vault
Using the Azure Portal, go to the Key Vault's access policies, and grant yourself **Secret Management** access to the Key Vault. This will allow you to run the application on your local development machine. 

1.	Search for your Key Vault in “Search Resources dialog box” in Azure Portal.
2.	Select "Overview", and click on Access policies
3.	Click on "Add New", select "Secret Management" from the dropdown for "Configure from template"
4.	Click on "Select Principal", add your account 
5.	Click on "OK" to add the new Access Policy, then click "Save" to save the Access Policies

## Step 3: Clone the repo 
Clone the repo to your development machine. 

The project has two relevant Nuget packages:
1. Microsoft.Azure.Services.AppAuthentication - makes it easy to fetch access tokens for Service-to-Azure-Service authentication scenarios. 
2. Microsoft.Azure.KeyVault - contains methods for interacting with Key Vault. 

The relevant code is in WebAppKeyVault/WebAppKeyVault/Controllers/HomeController.cs file. The **AzureServiceTokenProvider** class (which is part of Microsoft.Azure.Services.AppAuthentication) tries the following methods to get an access token:-
1. Managed Service Identity (MSI) - for scenarios where the code is deployed to Azure, and the Azure resource supports MSI. 
2. [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) (for local development) - Azure CLI version 2.0.12 and above supports the **get-access-token** option. AzureServiceTokenProvider uses this option to get an access token for local development. 
3. Active Directory Integrated Authentication (for local development). To use integrated Windows authentication, your domain’s Active Directory must be federated with Azure Active Directory. Your application must be running on a domain-joined machine under a user’s domain credentials.

```csharp    
public async System.Threading.Tasks.Task<ActionResult> Index()
{
    AzureServiceTokenProvider azureServiceTokenProvider = new AzureServiceTokenProvider();

    try
    {
        var keyVaultClient = new KeyVaultClient(
            new KeyVaultClient.AuthenticationCallback(azureServiceTokenProvider.KeyVaultTokenCallback));

        var secret = await keyVaultClient.GetSecretAsync("https://keyvaultname.vault.azure.net/secrets/secret")
            .ConfigureAwait(false);

        ViewBag.Secret = $"Secret: {secret.Value}";
        
    }
    catch (Exception exp)
    {
        ViewBag.Error = $"Something went wrong: {exp.Message}";
    }

    ViewBag.Principal = azureServiceTokenProvider.PrincipalUsed != null ? $"Principal Used: {azureServiceTokenProvider.PrincipalUsed}" : string.Empty;

    return View();
}
```
## Step 4: Replace the key vault name
In the HomeController.cs file, change the Key Vault name to the one you just created. Replace **KeyVaultName** with the name of your Key Vault. 

## Step 5: Run the application on your local development machine
### Authenticating to Azure Services

Local machines do not support managed identities for Azure resources.  As a result, the `Microsoft.Azure.Services.AppAuthentication` library uses your developer credentials to run in your local development environment. When the solution is deployed to Azure, the library uses a managed identity to switch to an OAuth 2.0 client credential grant flow.  This means you can test the same code locally and remotely without worry.

For local development, `AzureServiceTokenProvider` fetches tokens using **Visual Studio**, **Azure command-line interface** (CLI), or **Azure AD Integrated Authentication**. Each option is tried sequentially and the library uses the first option that succeeds. If no option works, an `AzureServiceTokenProviderException` exception is thrown with detailed information.

### Authenticating with Visual Studio

Sign in to Visual Studio and use **Tools**&nbsp;>&nbsp;**Options**&nbsp;>&nbsp;**Azure Service Authentication** to select an account for local development. 

It may also be necessary to reauthenticate your developer token. To do so, go to **Tools**&nbsp;>&nbsp;**Options**>**Azure&nbsp;Service&nbsp;Authentication** and look for a **Re-authenticate** link under the selected account.  Select it to authenticate. 

**Run the application in Visual Studio** and the `AzureServiceTokenProvider` should use your developer token for authentication.

If you run into problems using Visual Studio, such as errors regarding the token provider file, make sure you have [Visual Studio 2017 v15.5](https://blogs.msdn.microsoft.com/visualstudio/2017/10/11/visual-studio-2017-version-15-5-preview/) or later installed.



### Authenticating with Azure CLI

To use Azure CLI for local development:

1. Install [Azure CLI v2.0.12](https://docs.microsoft.com/cli/azure/install-azure-cli) or later. Upgrade earlier versions. 

2. Use **az login** to sign in to Azure.

Use `az account get-access-token` to verify access.  If you receive an error, verify that Step 1 completed successfully. 

If Azure CLI is not installed to the default directory, you may receive an error reporting that `AzureServiceTokenProvider` cannot find the path for Azure CLI.  Use the **AzureCLIPath** environment variable to define the Azure CLI installation folder. `AzureServiceTokenProvider` adds the directory specified in the **AzureCLIPath** environment variable to the **Path** environment variable when necessary.

If you are signed in to Azure CLI using multiple accounts or your account has access to multiple subscriptions, you need to specify the specific subscription to be used.  To do so, use:

```
az account set --subscription <subscription-id>
```

This command generates output only on failure.  To verify the current account settings, use:

```
az account list
```



Since your developer account has access to the Key Vault, you should see the secret on the web page. Principal Used will show type "User" and your user account. 

You can also use a service principal to run the application on your local development machine. See the section "Running the application using a service principal" later in the tutorial on how to do this. 

## Step 6: Deploy the Web App to Azure
Use any of the methods outlined on [Deploy your app to Azure App Service](https://docs.microsoft.com/en-us/azure/app-service-web/web-sites-deploy) to publish the Web App to Azure. 
After you deploy it, browse to the web app. You should see the secret on the web page, and this time the Principal Used will show "App", since it ran under the context of the App Service. 
The AppId of the MSI will be displayed. 



## Automating key renewal
Create an azure **automation account**. Make sure "Create Azure Run As Account" is set to yes. Create the account and wait for it to finish. 

Go to **modules** and install the  **AzureRM.*KeyVault***  module.



Go to your **keyvault** into the "**Access policies**" settings.

Add your **automation principal** and grant **all permissions**.



Go back to the **automation account** go to runbooks and create a powershell runbook.


```powershell
<#
    .DESCRIPTION
        Updates password in database and keyvault

    .NOTES
        AUTHOR: Patrick van EK
        LASTEDIT: June 26, 2019
#>

param 
( 
    #[Parameter (Mandatory=$true)] 
    [int] $passwordLength = 30, 
    #[Parameter (Mandatory=$true)] 
    [string]$databaseResourceGroup = 'meetup-shared',
    #[Parameter (Mandatory=$true)] 
    [string]$databaseName = 'meetup-shared',
    #[Parameter (Mandatory=$true)] 
    [string]$keyvaultName = 'pvekeyvaulttest',
    #[Parameter (Mandatory=$true)] 
    [string]$secretName = 'DatabasePassword'
) 

$ErrorActionPreference = 'Stop'

$connectionName = "AzureRunAsConnection"
try
{
    # Get the connection "AzureRunAsConnection "
    $servicePrincipalConnection = Get-AutomationConnection -Name $connectionName         

    "Logging in to Azure..."
    Add-AzureRmAccount `
        -ServicePrincipal `
        -TenantId $servicePrincipalConnection.TenantId `
        -ApplicationId $servicePrincipalConnection.ApplicationId `
        -CertificateThumbprint $servicePrincipalConnection.CertificateThumbprint 

        Write-Output "Generating password..."

        # from azure gallery 'New-SecurePassword'

        $generatedPassword = ([char[]]([char]33..[char]95) + ([char[]]([char]97..[char]126)) | Sort-Object {Get-Random})[0..($passwordLength-2)] -join ''
        $newSecret = ConvertTo-SecureString -AsPlainText -Force $generatedPassword 

        Write-Output "Setting password in server..."
        Set-AzureRmSqlServer -ResourceGroupName $databaseResourceGroup -ServerName $databaseName -SqlAdministratorPassword $newSecret | out-null

        Write-Output "Setting password in KeyVault..."
        Set-AzureKeyVaultSecret -Name $secretName -SecretValue $newSecret -VaultName $keyvaultName | out-null


        Write-Output "New password set in Server and KeyVault!"
}
catch {
    if (!$servicePrincipalConnection)
    {
        $ErrorMessage = "Connection $connectionName not found."
        throw $ErrorMessage
    } else{
        Write-Error -Message $_.Exception
        throw $_.Exception
    }
}
```



You can now schedule this runbook to run regularly to rotate your keyvault passwords.


## Summary
The web app was successfully able to get a secret at runtime from Azure Key Vault using your developer account during development, and using MSI when deployed to Azure, without any code change between local development environment and Azure. 
As a result, you did not have to explicitly handle a service principal credential to authenticate to Azure AD to get a token to call Key Vault. You do not have to worry about renewing the service principal credential either, since MSI takes care of that.  