---
layout: post
title: Introducing Bicep PowerShell Module
author: Stefan Ivemo
summary: An introduction to the Bicep PowerShell module.
---

As you may know I'm completely sold on [Bicep](https://github.com/Azure/bicep), I absolutely love using it! Since a while back I use Bicep to create all my templates for Azure Deployments (as long the features I need is supported in Bicep) and I'm super excited for the upcoming release of v0.3 that will give us the missing piece of the puzzle, copy loops!

Lately I've been spending a lot of time creating a bunch of Bicep modules that I re-use in my templates over and over again. When I've spent a couple of hours coding I usually end up with a folder containing multiple `.bicep` files that I need to compile running `bicep build`. But since `bicep build` doesn't have a `--all` switch or support wildcard characters on Windows (on OSX/Linux you can run `bicep build *.bicep`), I started using a simple PowerShell function to compile all `.bicep` files in a folder instead of compiling them one by one. After a couple of days using the function it escalated with more features and I decided to put together a PowerShell module that wraps Bicep CLI and share it with the community. And that's how the [Bicep PowerShell Module](https://github.com/StefanIvemo/BicepPowerShell) was born.

## Commands implemented

When writing this post the following commands have been implemented in the module. For a updated list of commands see the [Bicep - PowerShell Module repository](https://github.com/StefanIvemo/BicepPowerShell).

- `Invoke-BicepBuild` - An equivalent to `bicep build` with some additional features.
  - Use the `-Path` switch to specify a folder with `.bicep` files in order to compile them all at the same time.
  - If you have work in progress in the folder that you've specified, you can use the `-ExcludeFile` parameter to exclude a file from compilation.
  - Use the `-GenerateParameterFile` switch to generate ARM Template parameter files for the compiled `.bicep` file(s).
- `ConvertTo-Bicep` - An equivalent to `bicep decompile`.
  - Use the `-Path` switch to specify a folder with ARM Templates in `.json` format in order to decompile them all.
- `Get-BicepVersion` - Compares the installed version of Bicep CLI with the latest release.
- `Install-BicepCLI` - Install the latest Bicep CLI release available from the Azure/Bicep repo.
- `Update-BicepCLI` - Updates Bicep CLI if there is a newer version available.

## Installing the module

The Bicep module is available at [PowerShell Gallery](https://www.powershellgallery.com/packages/Bicep/).

{% highlight powershell %}
Install-Module -Name Bicep
{% endhighlight %}

## Invoke-BicepBuild

Lets take a look at some of the features available in `Invoke-BicepBuild`!

### Compile multiple files
Here´s how you can compile all `.bicep` files in a directory.

If we take a look in the directory `C:\Bicep\Modules` we can see that I have a couple of Bicep files in it.

{% highlight powershell %}
Get-ChildItem -Path 'C:\Bicep\Modules'
        
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
{% endhighlight %}

To compile all the files I can simply run `Invoke-BicepBuild` if the working directory is the same directory as where my bicep modules are located in. But I have all my bicep modules in a different directory than my working directory and I also want to exclude `appgw.bicep` from compilation, because it's still a work in progress and I know it will just generate a lot of build errors. I can then run `Invoke-BicepBuild -Path C:\Bicep\Modules -ExcludeFile appgw.bicep` to compile the files in the directory.

{% highlight powershell %}
Invoke-BicepBuild -Path 'C:\Bicep\Modules' -ExcludeFile 'appgw.bicep'
{% endhighlight %}

If we take a look inside the directory again we'll see that all `.bicep` files have been compiled to ARM templates except `appgw.bicep`.

{% highlight powershell %}
Get-ChildItem -Path 'C:\Bicep\Modules'
        
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
{% endhighlight %}

### Generate ARM Template parameter files

When you run `bicep build` to compile your `.bicep` files to ARM Templates you will only get a template file. But you´ll probably end up creating a parameter file before you deploy the template. I've created the `-GenerateParameterFile` switch to save you some time here. It will generate a parameter file containing all the parameters defined in the `.bicep` file and add any default values defined as value in the parameter file. Lets have a look at how it works.

If we look at the `vnet.bicep` file we compiled earlier. It has the following parameters defined:

{% highlight console %}
param location string = resourceGroup().location
param vnetname string = 'super-duper-vnet'
param addressprefix string = '10.0.0.0/24'
param dnsservers array
param enableDdosProtection bool = false
param ddosProtectionPlanID string = ''
{% endhighlight %}

If we compile `vnet.bicep` again using the `-GenerateParameterFile` switch we will now get parameter file called `vnet.parameters.json` as well.

{% highlight powershell %}
Invoke-BicepBuild -Path 'C:\Bicep\Modules\vnet.bicep' -GenerateParameterFile
Get-ChildItem -Path 'C:\Bicep\Modules\' vnet*
        
    Directory: C:\Bicep\Modules
        
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a---          2021-01-16    21:49            851 vnet.bicep
-a---          2021-01-16    21:49           1915 vnet.json
-a---          2021-01-16    21:49            499 vnet.parameters.json
{% endhighlight %}

And if we look inside `vnet.parameters.json` it's a valid parameter file.

> NOTE: All default values have been added to the parameter file except for `resourceGroup().location` used as default value for the `vnetname` parameter. Since ARM template functions can´t be used in parameter files they are replace with empty strings instead.

{% highlight JSON %}
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
{% endhighlight %}

## ConvertTo-Bicep

The only feature added to `ConvertTo-Bicep` is the possibility to decompile multiple ARM Templates to `.bicep` files at the same time.

### Decompile multiple ARM Templates

Here´s how you can decompile all `.json` files in a directory.

We have a directory with multiple ARM Templates in it.
        
{% highlight powershell %}
Get-ChildItem -Path 'C:\ARMTemplates\'
            
    Directory: C:\ARMTemplates
            
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a---          2021-01-16    21:37           1702 keyvault.json
-a---          2021-01-16    21:38           1034 nsg.json
-a---          2021-01-16    21:38           2089 publicip.json
-a---          2021-01-16    21:38           1249 routetable.json
-a---          2021-01-16    21:38           3062 subnet.json
-a---          2021-01-16    21:49           1915 vnet.json
{% endhighlight %}

We can now decompile them all to `.bicep` files using `ConvertTo-Bicep -Path C:\ARMTemplates`

{% highlight powershell %}
ConvertTo-Bicep -Path 'C:\ARMTemplates'
{% endhighlight %}

When we look in the folder again we can see that `.bicep` files have been generated for each ARM Template in the directory.

{% highlight powershell %}
Get-ChildItem -Path 'C:\ARMTemplates\'
        
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
{% endhighlight %}
    
## Get-BicepVersion

`Get-BicepVersion` is a simple cmdlet to that outputs the installed version of Bicep CLI and the version number of the latest release in the [Azure/Bicep repo](https://github.com/Azure/bicep/releases)

### Check versions

To check the versions just run `Get-BicepVersion`

{% highlight powershell %}
Get-BicepVersion
        
InstalledVersion LatestVersion
---------------- -------------
0.2.212          0.2.212
{% endhighlight %}

## Install-BicepCLI

`Install-BicepCLI` installs the latest release of Bicep CLI.

### Install Bicep CLI

To install Bicep CLI run `Install-BicepCLI`. If Bicep CLI is already installed a message will be outputted in the terminal.

{% highlight powershell %}
Install-BicepCLI
The latest Bicep CLI Version is already installed.
{% endhighlight %}

Or:

{% highlight powershell %}
Install-BicepCLI
Bicep CLI is already installed, but there is a newer release available. Use Update-BicepCLI  
or Install-BicepCLI -Force to updated to the latest release
{% endhighlight %}

## Update-BicepCLI

`Update-BicepCLI` updates Bicep CLI to the latest release.

### Update Bicep CLI

To update Bicep CLI run `Update-BicepCLI`. If the latest version of Bicep CLI is already installed a message will be outputted in the terminal.

{% highlight powershell %}
Update-BicepCLI
You are already running the latest version of Bicep CLI.
{% endhighlight %}

## Bug report and feature request

If you find a bug or have an idea for a new feature create an issue in the [BicepPowerShell repo](https://github.com/StefanIvemo/BicepPowerShell/issues). This is also the place where you can see any planned features.

## Contribution

If you like the Bicep PowerShell module and want to contribute you are very much welcome to do so. Please create an issue before you start working with a brand new feature to make sure that it's not already in the works or that the idea has been dismissed already.

## Summary

That's it! Hope you like it!

<br>
<br>
<div class="commenttext">
    To write a comment click "Sign in to comment" and use your GitHub account, a GitHub issue will be created for this post.
</div>
<script src="https://utteranc.es/client.js"
        repo="StefanIvemo/stefanivemo.github.io"
        issue-term="pathname"
        label="Comment"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>

<script async defer src="https://buttons.github.io/buttons.js"></script>