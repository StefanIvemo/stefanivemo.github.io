---
layout: post
title: Inventory App Services enabled for VNet Integration
---

I've been working a lot with Azure Networking the last couple of months, re-building and implementing new networking designs all over the place. When the time comes to move services to the new VNets I've deployed it can sometimes be a bit difficult to find all the services connected to the old VNets(not all services are listed under connected devices in the VNet). One of them are App Services and Function Apps with VNet Integration enabled. Since VNet Integration was released for App Services and Function Apps (available for Function Apps running on App Service Plans only!) a couple of months ago it's been spreading like a disease across Azure Subscriptions with high privileged developers enabling it left to right in need of On-Premises connectivity😘. There can be hundreds of them and sometimes you need to find them to be able to get control of the network again.
  
Finding all App Services and Function Apps enabled for VNet Integration is a bit more difficult than you can imagine when you start looking at it.

1. The obvious start for me was to use PowerShell to list all the App Services. But it turns out there is no PowerShell Support to configure or view VNet Integration settings for App Services yet. 
2. Ok, so how about Azure Resource Graph then? After a bit of research I noticed that the VNet Integration config can be found in the resource type `microsoft.web/sites/config` and that is not a resource type yet [supported](https://docs.microsoft.com/en-us/azure/governance/resource-graph/reference/supported-tables-resources){:target="_blank"} by Azure Resource Graph, so we can't use that either.
3. Reading the docs for [VNet Integration](https://docs.microsoft.com/en-us/azure/app-service/web-sites-integrate-with-vnet#automation){:target="_blank"} there is a section regarding automation using Azure CLI. The two commands available in Preview are `az webapp vnet-integration` and `az appservice vnet-integration`, finally we have something to work with.


The Script
------
The script is a simple one where I just list all App Services and Function Apps in my subscription using the commands `az webapp list` and `az functionapp list`. Then I loop through the result looking for services with VNet Integration enabled and outputs some information regarding the App Services and Function Apps found that I find useful.

<a class="github-button" href="https://github.com/StefanIvemo/AzureNetworking/blob/master/Scripts/Get-AppServiceVNetIntegration.ps1" aria-label="Use this template StefanIvemo/APIM on GitHub">Use this script</a>

{% highlight powershell %}
#Login to your Azure Subscription using az login before running the script
$AppServices = @(
az webapp list | ConvertFrom-Json
az functionapp list | ConvertFrom-Json
)
$VNetAppServices = foreach ($App in $AppServices) {    
    $Apps = az webapp vnet-integration list --n $App.Name -g $App.ResourceGroup
    if ($Apps -ne "[]") {
        $ServerFarm = $App.appServicePlanId -split '/'
        $VNetIntData = $Apps | ConvertFrom-Json
        $VNet = $VNetIntData.VnetResourceID -split '/'
        [PSCustomObject]@{
            AppServiceName = $App.Name
            ServerFarm     = $ServerFarm[8]
            ResourceGroup  = $App.ResourceGroup
            Location       = $App.Location
            VNetName       = $VNet[8]
            SubnetName     = $VNetIntData.name  
            VNetRG         = $VNet[4]       
        }
    }    
}
$VNetAppServices | ft
{% endhighlight %}

The output will look like this:

<img src="https://github.com/StefanIvemo/stefanivemo.github.io/blob/master/images/vnet-integration/script-output.PNG?raw=true">

Azure CLI & PowerShell Color bug
------
When running this script, I had the problem that occurred every time I ran my script where the color on my output magically changed to black and staying like that until I restarted VSCode. If you test this script you might experience the same color bug I ran into, since it took me a while to figure out what happened and I wanted to share how to solve it with a workaround.

This is what happened:  

<img src="https://github.com/StefanIvemo/stefanivemo.github.io/blob/master/images/vnet-integration/script-output-black.PNG?raw=true">

It turns out there is a [bug](https://github.com/Azure/azure-cli/issues/12084){:target="_blank"} when running Azure CLI commands in PowerShell Scripts. There is a workaround available where you can add an environment variable to the [Azure CLI Config](https://docs.microsoft.com/en-us/cli/azure/azure-cli-configuration?view=azure-cli-latest){:target="_blank"} to disable color settings. The configuration file itself is located at `$AZURE_CONFIG_DIR/config`. The default value of `AZURE_CONFIG_DIR` is `$HOME/.azure` on Linux and macOS, and `%USERPROFILE%\.azure` on Windows. I've added the following environment variables to my config file to disable coloring and suppress warnings that commands are in preview.

{% highlight conf %}
[core]
only_show_errors=yes  
no_color=yes
{% endhighlight %}


Summary
------
Hope you find the script useful, happy inventory!

<script src="https://utteranc.es/client.js"
        repo="StefanIvemo/stefanivemo.github.io"
        issue-term="pathname"
        label="Comment"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>

<script async defer src="https://buttons.github.io/buttons.js"></script>