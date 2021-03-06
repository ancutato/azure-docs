---
title: Tutorial - Use Azure Key Vault with a Windows virtual machine in .NET | Microsoft Docs
description: In this tutorial, you configure an ASP.NET core application to read a secret from your key vault.
services: key-vault
author: msmbaldwin
manager: rajvijan

ms.service: key-vault
ms.subservice: general
ms.topic: tutorial
ms.date: 01/02/2019
ms.author: mbaldwin
ms.custom: mvc
#Customer intent: As a developer I want to use Azure Key Vault to store secrets for my app, so that they are kept secure.
---
# Tutorial: Use Azure Key Vault with a Windows virtual machine in .NET

Azure Key Vault helps you to protect secrets such as API keys, the database connection strings you need to access your applications, services, and IT resources.

In this tutorial, you learn how to get a console application to read information from Azure Key Vault. Application would use virtual machine managed identity to authenticate to Key Vault. 

The tutorial shows you how to:

> [!div class="checklist"]
> * Create a resource group.
> * Create a key vault.
> * Add a secret to the key vault.
> * Retrieve a secret from the key vault.
> * Create an Azure virtual machine.
> * Enable a [managed identity](../../active-directory/managed-identities-azure-resources/overview.md) for the Virtual Machine.
> * Assign permissions to the VM identity.

Before you begin, read [Key Vault basic concepts](basic-concepts.md). 

If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F).

## Prerequisites

For Windows, Mac, and Linux:
  * [Git](https://git-scm.com/downloads)
  * The [.NET Core 3.1 SDK or later](https://dotnet.microsoft.com/download/dotnet-core/3.1).
  * [Azure CLI](/cli/azure/install-azure-cli?view=azure-cli-latest).

## Create resources and assign permissions

Before you start coding you need to create some resources, put a secret into your key vault, and assign permissions.

### Sign in to Azure

To sign in to Azure by using the Azure CLI, enter:

```azurecli
az login
```

### Create a resource group

An Azure resource group is a logical container into which Azure resources are deployed and managed. Create a resource group by using the [az group create](/cli/azure/group#az-group-create) command. 

This example creates a resource group in the West US location:

```azurecli
# To list locations: az account list-locations --output table
az group create --name "<YourResourceGroupName>" --location "West US"
```

Your newly created resource group will be used throughout this tutorial.

### Create a key vault and populate it with a secret

Create a key vault in your resource group by providing the [az keyvault create](/cli/azure/keyvault?view=azure-cli-latest#az-keyvault-create) command with the following information:

* Key vault name: a string of 3 to 24 characters that can contain only numbers (0-9), letters (a-z, A-Z), and hyphens (-)
* Resource group name
* Location: **West US**

```azurecli
az keyvault create --name "<YourKeyVaultName>" --resource-group "<YourResourceGroupName>" --location "West US"
```
At this point, your Azure account is the only one that's authorized to perform operations on this new key vault.

Now add a secret to your key vault using the [az keyvault secret set](/cli/azure/keyvault/secret?view=azure-cli-latest#az-keyvault-secret-set) command


To create a secret in the key vault called **AppSecret**, enter the following command:

```azurecli
az keyvault secret set --vault-name "<YourKeyVaultName>" --name "AppSecret" --value "MySecret"
```

This secret stores the value **MySecret**.

### Create a virtual machine
Create a virtual machine by using one of the following methods:

* [The Azure CLI](../../virtual-machines/windows/quick-create-cli.md)
* [PowerShell](../../virtual-machines/windows/quick-create-powershell.md)
* [The Azure portal](../../virtual-machines/windows/quick-create-portal.md)

### Assign an identity to the VM
Create a system-assigned identity for the virtual machine with the [az vm identity assign](/cli/azure/vm/identity?view=azure-cli-latest#az-vm-identity-assign) command:

```azurecli
az vm identity assign --name <NameOfYourVirtualMachine> --resource-group <YourResourceGroupName>
```

Note the system-assigned identity that's displayed in the following code. The output of the preceding command would be: 

```output
{
  "systemAssignedIdentity": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "userAssignedIdentities": {}
}
```

### Assign permissions to the VM identity
Assign the previously created identity permissions to your key vault with the [az keyvault set-policy](/cli/azure/keyvault?view=azure-cli-latest#az-keyvault-set-policy) command:

```azurecli
az keyvault set-policy --name '<YourKeyVaultName>' --object-id <VMSystemAssignedIdentity> --secret-permissions get list
```

### Sign in to the virtual machine

To sign in to the virtual machine, follow the instructions in [Connect and sign in to an Azure virtual machine running Windows](../../virtual-machines/windows/connect-logon.md).

## Set up the console app

Create a console app and install the required packages using the `dotnet` command.

### Install .NET Core

To install .NET Core, go to the [.NET downloads](https://www.microsoft.com/net/download) page.

### Create and run a sample .NET app

Open a command prompt.

You can print "Hello World" to the console by running the following commands:

```console
dotnet new console -n keyvault-console-app
cd keyvault-console-app
dotnet run
```

### Install the package

From the console window, install the Azure Key Vault Secrets client library for .NET:

```console
dotnet add package Azure.Security.KeyVault.Secrets
```

For this quickstart, you will need to install the following identity package to authenticate to Azure Key Vault:

```console
dotnet add package Azure.Identity
```

## Edit the console app

Open the *Program.cs* file and add these packages:

```csharp
using System;
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
```

Add these lines, updating the URI to reflect the `vaultUri` of your key vault. Below code is using  ['DefaultAzureCredential()'](/dotnet/api/azure.identity.defaultazurecredential?view=azure-dotnet) for authentication to key vault, which is using token from application managed identity to authenticate. It is also using exponential backoff for retries in case of key vault is being throttled.

```csharp
  class Program
    {
        static void Main(string[] args)
        {
            string secretName = "mySecret";

            var kvUri = "https://<your-key-vault-name>.vault.azure.net";
            SecretClientOptions options = new SecretClientOptions()
            {
                Retry =
                {
                    Delay= TimeSpan.FromSeconds(2),
                    MaxDelay = TimeSpan.FromSeconds(16),
                    MaxRetries = 5,
                    Mode = RetryMode.Exponential
                 }
            };

            var client = new SecretClient(new Uri(kvUri), new DefaultAzureCredential(),options);

            Console.Write("Input the value of your secret > ");
            string secretValue = Console.ReadLine();

            Console.Write("Creating a secret in " + keyVaultName + " called '" + secretName + "' with the value '" + secretValue + "` ...");

            client.SetSecret(secretName, secretValue);

            Console.WriteLine(" done.");

            Console.WriteLine("Forgetting your secret.");
            secretValue = "";
            Console.WriteLine("Your secret is '" + secretValue + "'.");

            Console.WriteLine("Retrieving your secret from " + keyVaultName + ".");

            KeyVaultSecret secret = client.GetSecret(secretName);

            Console.WriteLine("Your secret is '" + secret.Value + "'.");

            Console.Write("Deleting your secret from " + keyVaultName + " ...");

            client.StartDeleteSecret(secretName);

            System.Threading.Thread.Sleep(5000);
            Console.WriteLine(" done.");

        }
    }
```

The preceding code shows you how to do operations with Azure Key Vault in a Windows virtual machine.

## Clean up resources

When they are no longer needed, delete the virtual machine and your key vault.

## Next steps

> [!div class="nextstepaction"]
> [Azure Key Vault REST API](https://docs.microsoft.com/rest/api/keyvault/)
