---
layout: post
title: Introducing Bicep PowerShell Module
author: Stefan Ivemo
summary: An introduction to the Bicep PowerShell module.
---

As you may know I'm completely sold on [Bicep](https://github.com/Azure/bicep), I absolutely love using it! I use it all the time to create my templates for Azure Deployments and I'm super excited for the upcoming release of v0.3 that will give us the missing piece of the puzzle, copy loops!

Lately I've been spending a lot of time creating a bunch of Bicep modules that I re-use in my templates over and over again. When I've spent a couple of hours coding I usually end up with a folder containing multiple `.bicep` files that I need to compile running `bicep build`. Since `bicep build` doesn't have a `--all` switch or support wildcard characters on Windows (on OSX/Linux you can run `bicep build *.bicep`) I started using a simple PowerShell function to compile all `.bicep` files in a folder instead of compiling them one by one. After a couple of days using the function it escalated with more features and I decided to put together a PowerShell module that wraps Bicep CLI and share it with the community. And that's when the [Bicep PowerShell Module](https://github.com/StefanIvemo/BicepPowerShell) was born.

## Commands implemented

When writing this post the following commands have been implemented. For a updated list of commands see the [Bicep - PowerShell Module repository](https://github.com/StefanIvemo/BicepPowerShell).

- `Invoke-BicepBuild` - An equivalent to `bicep build` with some additional features.
  - Use the `-Path` switch to specify a folder with `.bicep` files in order to compile them all.
  - If you have work in progress in the folder that you've specified, you can use the `-ExcludeFile` parameter to exclude a file from compilation.
  - Use the `-GenerateParameterFile` switch to generate ARM Template parameter files for the compiled `.bicep` file(s). 
- `ConvertTo-Bicep` - An equivalent to `bicep decompile`.
  - Use the `-Path` switch to specify a folder with ARM Templates in `.json` format in order to decompile them all.
- `Get-BicepVersion` - Compares the installed version of Bicep CLI with the latest release.
- `Install-BicepCLI` - Install the latest Bicep CLI release available from the Azure/Bicep repo.
- `Update-BicepCLI` - Updates Bicep CLI if there is a newer version available.

## Installing the module

The Bicep module is available at [PowerShell Gallery](https://www.powershellgallery.com/packages/Bicep/).

```powershell
Install-Module -Name Bicep
```

## Invoke-BicepBuild

Lets take a look at some of the features available in `Invoke-BicepBuild`!

### Compile multiple files
Here´s how you can compile all `.bicep` files in a directory.

1. In the directory `C:\Bicep\Modules` I have a couple of Bicep modules. 

    ```powershell
    PS C:\> Get-ChildItem -Path C:\Bicep\Modules
    
        Directory: C:\Bicep\Modules
    
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    -a---          2021-01-15    12:47          11872 appgw.bicep
    -a---          2021-01-15    13:23            849 keyvault.bicep
    -a---          2021-01-08    14:01            346 nsg.bicep
    -a---          2021-01-15    14:14           1020 publicip.bicep
    -a---          2021-01-08    14:01            464 routetable.bicep
    -a---          2021-01-08    14:01           1491 subnet.bicep
    -a---          2021-01-08    14:01            819 vnet.bicep
    ```

2. To compile all the files I can simply run `Invoke-BicepBuild` if the working directory is the same directory as where my bicep modules are located. I have all my bicep modules in a different directory than my working directory and I want to exclude `appgw.bicep` from compilation because it's still a work in progress and I know it will just generate a lot of build errors. I can then run `Invoke-BicepBuild -Path C:\Bicep\Modules -ExcludeFile appgw.bicep` to compile the files in the directory.

    ```powershell
    PS C:\Bicep\Modules> Invoke-BicepBuild -Path C:\Bicep\Modules -ExcludeFile appgw.bicep
    ```

3. If we take a look inside the directory again we'll see that ARM templates have been created for each `.bicep` file except `appgw.bicep`.

    ```powershell
    PS C:\> Get-ChildItem -Path C:\Bicep\Modules
    
        Directory: C:\Bicep\Modules
    
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    -a---          2021-01-15    12:47          11872 appgw.bicep
    -a---          2021-01-15    13:23            849 keyvault.bicep
    -a---          2021-01-16    21:37           1702 keyvault.json
    -a---          2021-01-08    14:01            346 nsg.bicep
    -a---          2021-01-16    21:38           1034 nsg.json
    -a---          2021-01-15    14:14           1020 publicip.bicep
    -a---          2021-01-16    21:38           2089 publicip.json
    -a---          2021-01-08    14:01            464 routetable.bicep
    -a---          2021-01-16    21:38           1249 routetable.json
    -a---          2021-01-08    14:01           1491 subnet.bicep
    -a---          2021-01-16    21:38           3062 subnet.json
    -a---          2021-01-08    14:01            819 vnet.bicep
    -a---          2021-01-16    21:38           1861 vnet.json
    ```

### Generate ARM Template parameter files

When you run `bicep build` to compile your `.bicep` files to ARM Templates you will only get a template file. But you´ll probably end up creating a parameter file before you deploy the template. I've created the `-GenerateParameterFile` switch to save you some time. It will generate a parameter file containing all the parameters defined in the `.bicep` file and add any default values defined as value in the parameter file. Lets have a look at how it works.

1. If we look at the `vnet.bicep` file we compiled earlier. It has the following parameters defined:

    ```bicep
    param location string = resourceGroup().location
    param vnetname string = 'super-duper-vnet'
    param addressprefix string = '10.0.0.0/24'
    param dnsservers array
    param enableDdosProtection bool = false
    param ddosProtectionPlanID string = ''
    ```

2. If we compile `vnet.bicep` again using the `-GenerateParameterFile` switch we will get parameter file called `vnet.parameters.json`.

    ```powershell
    PS C:\> Invoke-BicepBuild -Path C:\Bicep\Modules\vnet.bicep -GenerateParameterFile
    PS C:\> Get-ChildItem -Path C:\Bicep\Modules\ vnet*
    
        Directory: C:\Bicep\Modules
    
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    -a---          2021-01-16    21:49            851 vnet.bicep
    -a---          2021-01-16    21:49           1915 vnet.json
    -a---          2021-01-16    21:49            499 vnet.parameters.json
    ```

3. And if we look inside `vnet.parameters.json` it's a valid parameter file.

    > NOTE: All default values have been added to the parameter file except for `resourceGroup().location` used as default value for the `vnetname` parameter. Since ARM template functions can´t be used in parameter files they are replace with empty strings instead.

    ```json
    {
      "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
      "contentVersion": "1.0.0.0",
      "parameters": {
        "location": {
          "value": ""
        },
        "vnetname": {
          "value": "super-duper-vnet"
        },
        "addressprefix": {
          "value": "10.0.0.0/24"
        },
        "dnsservers": {
          "value": []
        },
        "enableDdosProtection": {
          "value": false
        },
        "ddosProtectionPlanID": {
          "value": ""
        }
      }
    }
    ```

## ConvertTo-Bicep

The only feature added to `ConvertTo-Bicep` is the possibility to decompile multiple ARM Templates to `.bicep` files at the same time.

### Decompile multiple ARM Templates

Here´s how you can decompile all `.json` files in a directory.

1. You have a directory with multiple ARM Templates.

    ```powershell
    PS C:\> Get-ChildItem -Path C:\ARMTemplates\
    
        Directory: C:\ARMTemplates
    
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    -a---          2021-01-16    21:37           1702 keyvault.json
    -a---          2021-01-16    21:38           1034 nsg.json
    -a---          2021-01-16    21:38           2089 publicip.json
    -a---          2021-01-16    21:38           1249 routetable.json
    -a---          2021-01-16    21:38           3062 subnet.json
    -a---          2021-01-16    21:49           1915 vnet.json
    ```

2. You can now decompile them all to `.bicep` files using `ConvertTo-Bicep -Path C:\ARMTemplates`

    ```powershell
    PS C:\> ConvertTo-Bicep -Path C:\ARMTemplates
    ```

3. When we look in the folder again we can see that `.bicep` files have been generated for each ARM Template.

    ```powershell
    PS C:\> Get-ChildItem -Path C:\ARMTemplates\
    
        Directory: C:\ARMTemplates
    
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    -a---          2021-01-16    22:03            860 keyvault.bicep
    -a---          2021-01-16    21:37           1702 keyvault.json
    -a---          2021-01-16    22:03            450 nsg.bicep
    -a---          2021-01-16    21:38           1034 nsg.json
    -a---          2021-01-16    22:03           1026 publicip.bicep
    -a---          2021-01-16    21:38           2089 publicip.json
    -a---          2021-01-16    22:03            574 routetable.bicep
    -a---          2021-01-16    21:38           1249 routetable.json
    -a---          2021-01-16    22:03           1678 subnet.bicep
    -a---          2021-01-16    21:38           3062 subnet.json
    -a---          2021-01-16    22:03            887 vnet.bicep
    -a---          2021-01-16    21:49           1915 vnet.json
    ```

## Get-BicepVersion

`Get-BicepVersion` is a simple cmdlet to that outputs the installed version of Bicep CLI and the version number of the latest release in the [Azure/Bicep repo](https://github.com/Azure/bicep/releases)

### Check versions

1. To check the versions just run `Get-BicepVersion`

    ```powershell
    PS C:\> Get-BicepVersion
    
    InstalledVersion LatestVersion
    ---------------- -------------
    0.2.212          0.2.212
    ```

## Install-BicepCLI

`Install-BicepCLI` installs the latest release of Bicep CLI.

### Install Bicep CLI

1. To install Bicep CLI run `Install-BicepCLI`. If Bicep CLI is already installed a message will be outputted in the terminal.

    ```powershell
    PS C:\> Install-BicepCLI
    The latest Bicep CLI Version is already installed.
    ```
    Or:
    ```powershell
    PS C:\> Install-BicepCLI
    Bicep CLI is already installed, but there is a newer release available. Use Update-BicepCLI or Install-BicepCLI -Force to updated to the latest release
    ```

## Update-BicepCLI

`Update-BicepCLI` updates Bicep CLI to the latest release.

### Update Bicep CLI

1. To update Bicep CLI run `Update-BicepCLI`. If the latest version of Bicep CLI is already installed a message will be outputted in the terminal.

    ```powershell
    PS C:\> Update-BicepCLI
    You are already running the latest version of Bicep CLI.
    ```

## Bug report and feature request

If you find a bug or have an idea for a new feature create an issue in the [BicepPowerShell repo](https://github.com/StefanIvemo/BicepPowerShell/issues). This is also the place where you can see any planned features.

## Contribution

If you like the Bicep PowerShell module and want to contribute you are very much welcome to do so. Please create an issue before you start working with a brand new feature to make sure that it's not already in the works or that the idea has been dismissed already.

## Summary

That's it! Hope you like it!


<script src="https://utteranc.es/client.js"
        repo="StefanIvemo/stefanivemo.github.io"
        issue-term="pathname"
        label="Comment"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>

<script async defer src="https://buttons.github.io/buttons.js"></script>