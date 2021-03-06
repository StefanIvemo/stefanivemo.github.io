---
layout: post
title: Get started with Virtual WAN deployments using ARM Templates
author: Stefan Ivemo
summary: Struggling with Virtual WAN Deployments? In this post I will guide you through Virtual WAN ARM Template Deployment.
---

Deploying Azure Virtual WAN using ARM templates can be frustrating and it takes some time before you get a hang on it. There are a lot of dependencies between resources and you need to make sure that everything is deployed in the correct order, and also allow  both the Virtual Hub and routing to reach succeeded state before throwing more things at it. In this post I will guide you through some of the different resource types to help you with your Virtual WAN deployment.

**Note:** *This post explains first time deployments only, to make the template reusable to be able to update the Virtual WAN you have to add some logic to it. But that is something we can cover in a separate post, let's focus on getting that VWAN deployed!*

## API Version

For ARM templates, you must always specify an API version for every resource type you define in your template. The API version corresponds to a version of REST API operations that are released by the resource provider e.g `Microsoft.Network`. When a resource provider enables new features a new API Version is released. Sometimes a new API Version just enables a couple of new features, and you can just change it to the most recent version and your template will still function properly without any other modifications. But that's not always the case, in some cases a new API Version completely changes how some resources are defined and deployed. When it comes to Azure Virtual WAN the change from API Version `2020-04-01` to `2020-05-01` equals a couple of breaking changes. If you see an example template using API Version `2020-04-01` you can't just update the API Version and expect a successful deployment.

The following things are good to know when it comes to `Microsoft.Network` API Versions and Virtual WAN:

- `2020-04-01` - Do not use this version (or earlier) if you´re building a new template, you will have to do a lot of modifications when upgrading to a newer version. And you can not use Virtual Hub Route tables.
- `2020-05-01` - With this API Version a couple of major changes where introduced:
  - `Microsoft.Network/virtualHubs` has changed a lot. Things that in earlier versions was declared as properties inside the Virtual Hub are now child resources, the most common of them `Microsoft.Network/virtualHubs/hubVirtualNetworkConnections`
  - `Microsoft.Network/firewallPolicies` Azure Firewall policies have been updated to support Rule Collection Groups `Microsoft.Network/firewallPolicies/ruleCollectionGroups`. This completely change how Firewall Policies are defined.
- `2020-06-01` - Latest API Version

## Breaking down the template - Resource by resource

Lets break down a Virtual WAN with a secure virtual hub deployment into pieces. A complete ARM template for reference can be found in my [Azure Virtual WAN Playground Repo](https://github.com/StefanIvemo/vwan-playground). The examples in this post are based on that repository, check it out if you want to skip this article and dive into the full deployment at once. This article will focus on the following resource types used in a Virtual WAN:

- Virtual WAN
- Virtual Hub
- Azure Firewall Policy
- Azure Firewall
- Virtual Hub Route Tables
  - Updating defaultRouteTable
  - Custom Route Table
- Hub Virtual Network Connections
- VPN Sites
- VPN Gateway
  - VPN Connection

**Note:** *The code snippets in this post are examples on how you can define the resources in your ARM template. If you create a template using copy/paste and these examples it might not work as expected. See the full example in the [VWAN Playground Repo](https://github.com/StefanIvemo/vwan-playground) to get the full picture (you can also look at a 💪Bicep verison of the template!). I'm also not covering all available properties for each resource type. For a full list of properties see [ARM Template Reference](https://docs.microsoft.com/en-us/azure/templates/microsoft.network/allversions). At last I've decided to remove the `dependsOn` property from the code snippets to reduce the code, and just added a simple dependsOn note above the examples to explain what other resources it depends on. For some resources I've just added an extra dependsOn just to allow the Virtual Hub to fully update and reach a succeeded state. Sometimes if you throw to much at it you will end up with conflict errors or badRequests because the Virtual Hub is not back in a succeeded state after a previous operation.*

## Microsoft.Network/virtualWans

There´s not much to say about the `Microsoft.Network/virtualWans` resource, it's pretty straight forward. You create the resource specifying just a couple of properties:

- `type` - Specifies the [Virtual WAN SKU](https://docs.microsoft.com/en-us/azure/virtual-wan/upgrade-virtual-wan), allowed values are `Standard` or `Basic`.
- `disableVpnEncryption` - Property to disable VPN Encryption.
- `allowBranchToBranchTraffic` - Specifies if [branches](https://docs.microsoft.com/en-us/azure/virtual-wan/virtual-wan-faq#is-branch-to-branch-connectivity-allowed-in-virtual-wan) should be allowed to communicate through the Virtual WAN. Important to remember that User VPN counts as a branch, if you want to allow users connected to VPN to reach an On-Premises site, this must be enabled.
- `office365LocalBreakoutCategory` - Specify Office 365 local breakout category - Allowed values `Optimize`, `OptimizeAndAllow`, `All`, `None`

`dependsOn: Nothing`

{% highlight JSON %}
{
    "type": "Microsoft.Network/virtualWans",
    "apiVersion": "2020-05-01",
    "name": "contoso-vwan",
    "location": "westeurope",
    "properties": {
        "type": "Standard",
        "disableVpnEncryption": false,
        "allowBranchToBranchTraffic": true,
        "office365LocalBreakoutCategory": "None"
    }
}
{% endhighlight %}

## Microsoft.Network/virtualHubs

Next up is the Virtual Hub, just like the Virtual WAN resource it's a simple one. The properties specified are:

- `name` - Name of the Virtual Hub
- `addressPrefix` - The address space used by the Virtual Hub, minimum address space is /24.
- `virtualWan` - Resource ID to the Virtual WAN.

`dependsOn: Virtual WAN`

{% highlight JSON %}
{
    "type": "Microsoft.Network/virtualHubs",
    "apiVersion": "2020-05-01",
    "name": "contoso-vhub-westeurope",
    "location": "westeurope",
    "properties": {
      "addressPrefix": "10.0.0.0/24",
      "virtualWan": {
        "id": "[resourceId('Microsoft.Network/virtualWans', 'contoso-fw-westeurope')]"
      }
    }
}
{% endhighlight %}

## Microsoft.Network/firewallPolicies

Time to prepare for Azure Firewall (Secure Virtual Hub) by creating an [Azure Firewall Policy](https://docs.microsoft.com/en-us/azure/firewall-manager/policy-overview). I like to specify the Azure Firewall Policy and Rule Collection Groups as separate resources in the template because it simplifies the template, especially when using multiple Rule Collection Groups. It keeps the Firewall properties separated from the Firewall Rules which is quite neat.

- `name` - Firewall Policy Name
- `threatIntelMode` - The operation mode for [Threat Intelligence](https://docs.microsoft.com/en-us/azure/firewall/threat-intel). - `Alert`, `Deny`, `Off`
- `threatIntelWhitelist` - Threat intelligence whitelist object
  - `ipAddresses` - List of IP Addresses to whitelist from Threat Intelligence
  - `fqdns` - List of FQDNS to whitelist from Threat Intelligence
- `dnsSettings` - [Azure Firewall DNS Settings](https://docs.microsoft.com/en-us/azure/firewall/dns-settings)
  - `servers`- List of Custom DNS Servers
  - `enableProxy` - Property to enable DNS Proxy
  - `requireProxyForNetworkRules` - Using FQDNs in Network Rules are supported when set to true

`dependsOn: Nothing`

{% highlight JSON %}
{
    "type": "Microsoft.Network/firewallPolicies",
    "apiVersion": "2020-05-01",
    "name": "contoso-fw-westeurope-policy",
    "location": "westeurope",
    "properties": {
        "threatIntelMode": "Alert",
        "threatIntelWhitelist": {
            "ipAddresses": [],
            "fqdns": []
        },
        "dnsSettings": {
            "servers": [],
            "enableProxy": false,
            "requireProxyForNetworkRules": false
        }
    }
}
{% endhighlight %}

## Microsoft.Network/firewallPolicies/ruleCollectionGroups

Time to add some rules to the Firewall Policy, I'll keep it simple for now with a single Rule Collection Group containing a single Application Rule. Note that the properties defined in the rules object will differ depending on the rule type you use (Network, Application or DNAT).

- `name` - Name of the rule collection group
- `priority` - Rule Collection Group priority.
- `ruleCollections` - Group of Firewall Policy rule collections
  - `ruleCollectionType` - Specify `FirewallPolicyFilterRuleCollection` for application and network collections or `FirewallPolicyNatRuleCollection` for DNAT collections.
  - `name` - Name of the Rule Collection
  - `priority` - Rule Collection Priority
  - `action` - Rule Collection Action
    - `type` -  Allow or Deny
  - `rules` - Group of Rules (example below shows an application rule)
    - `ruleType` - ApplicationRule, NetworkRule or NatRule
    - `name` - Name of the Rule (must be unique within a rule collection)
    - `sourceAddresses` - Source IP Address or range
    - `sourceIpGroups` - Source IP Group (resource ID)
    - `protocols` - Protocols used by the Application Rule
      - `port` - Port number
      - `protocolType` - Protocol type, http, https, mssql
    - `targetFqdns` - Target FQDNs Object
    - `fqdnTags` - FQDN Tags Object

`dependsOn: Azure Firewall Policy`

{% highlight JSON %}
    {
      "type": "Microsoft.Network/firewallPolicies/ruleCollectionGroups",
      "apiVersion": "2020-06-01",
      "name": "contoso-fw-westeurope-policy/DefaultApplicationRuleCollectionGroup",
      "location": "westeurope",
      "properties": {
        "priority": 100,
        "ruleCollections": [
          {
            "ruleCollectionType": "FirewallPolicyFilterRuleCollection",
            "name": "RC-01",
            "priority": 100,
            "action": {
              "type": "Allow"
            },
            "rules": [
              {
                "ruleType": "ApplicationRule",
                "name": "Allow-msft",
                "sourceAddresses": [
                  "*"
                ],
                "sourceIpGroups": [],
                "protocols": [
                  {
                    "port": "80",
                    "protocolType": "http"
                  },
                  {
                    "port": "443",
                    "protocolType": "https"
                  }
                ],
                "targetFqdns": [
                  "*.microsoft.com"
                ],
                "fqdnTags": []
              }
            ]
          }
        ]
      }
    }
{% endhighlight %}

## Microsoft.Network/azureFirewalls

The Azure Firewall resource deployed in a Virtual Hub is almost the same as when deployed in a Virtual Network. There are some key differences:

- The SKU property must be specified with the name `AZFW_Hub` instead of `AZFW_VNet`.
- In a VNet deployed Firewall there is a `ipConfigurations` property object where the IP Configurations are set. The first IP Configuration object in a VNet deployed Azure Firewall have two properties, `subnet` that contains the resource ID to the AzureFirewallSubnet and the `publicIPAddress` property that contains the resource ID to the Public IP address to use with the Azure Firewall. Additional Public IPs are assigned by adding additional `ipConfigurations` without the `subnet` property which is only allowed in the first IP Configuration declared.

- In a *Secure Virtual Hub* we don't have the `ipConfigurations` property available. Since the Virtual WAN Hub is a Microsoft Managed VNet we can't access the AzureFirewallSubnet. When it comes to the Public IP addresses we can't decide which ones to use, the only thing we can do is specify the Public IP Count to control the number of IP Addresses allocated to the Firewall. (I really hope that we will be able to allocate IPs from Public IP Prefixes to a Secure Virtual Hub Firewall in the future!). So what do we have instead of the `ipConfigurations` property?
  - We have a new property called `virtualHub` this is where we specify the resource ID of the Virtual WAN to associate the Azure Firewall to.
  - There is also a `hubIPAddresses` property where we can specify the number of Public IPs ot use by setting the `publicIPs` `count`.

Lets take a look at the Azure Firewall Resource:

- `name` - Name of the Azure Firewall resource
- `sku` - Azure Firewall SKU Object
  - `name` - Name of the Azure Firewall SKU - `AZFW_VNet` or `AZFW_Hub`
  - `tier` - Standard or Premium (not available to deploy as an ordinary user, you will get an error. I guess that a private preview is happening here)
- `virtualHub` - Virtual Hub Resource ID
- `hubIPAddresses` - Hub IP Addresses Object
  - `publicIPs` - Public IP Property
    - `count` - Public IP count, the number of IPs to associate with the Firewall
  - `firewallPolicy` - Firewall Policy Resource ID

`dependsOn: Virtual WAN, Virtual Hub, Azure Firewall Policy`

{% highlight JSON %}
{
      "type": "Microsoft.Network/azureFirewalls",
      "apiVersion": "2020-05-01",
      "name": "contoso-fw-westeurope",
      "location": "westeurope",
      "properties": {
        "sku": {
          "name": "AZFW_Hub",
          "tier": "Standard"
        },
        "virtualHub": {
          "id": "[resourceId('Microsoft.Network/virtualHubs', 'contoso-vhub-westeurope')]"
        },
        "hubIPAddresses": {
          "publicIPs": {
            "count": 2
          }
        },
        "firewallPolicy": {
          "id": "[resourceId('Microsoft.Network/firewallPolicies', 'contoso-fw-westeurope-policy')]"
        }
      }
    }
{% endhighlight %}

## Microsoft.Network/virtualHubs/hubRouteTables

Now that the Azure Firewall is deployed it's time to get some routing in-place. I want to route all traffic from On-Premises locations to my Azure VNets through Azure Firewall. And for my VNets in Azure I want to send outbound internet traffic and traffic towards On-Premises through Azure Firewall. In order to do that I need a new Custom Hub Route Table `RT_VNet` that I will associate with all VNet Connections and add a static route to the `defaultRouteTable` used by all branches. Custom Route Tables for branches are not available, all branches are associated with and are propagating routes to the `defaultRouteTable`.

- `name` - Name of the Route Table, since it's a child resource to the Virtual Hub make sure that the correct segments are used.
- `routes` - List of all routes to add
  - `name` - Route name
  - `destinationType` - The type of destination, `CIDR`, `ResourceId` or `Service`.
  - `destination` - List of destinations
  - `nextHopType` - Next hop type, `CIDR`, `ResourceId` or `Service`
  - `nextHop` - Next hop resource ID (Azure Firewall or VNet Connection)
- `labels` - List of labels associated with this route table.

### defaultRouteTable

The `defaultRouteTable` is created with the Virtual Hub and are used by all branch connections by default. I want to make sure that all traffic between On-Premises and Azure is routed through Azure Firewall and to achieve that a static route must be added. I like to reserve a CIDR block dedicated for a specific Azure region. I use this for the Virtual Hub (or Virtual Network Hub for a traditional topology) and all connected VNets in the specified region. This simplifies routing between regions and regional firewalls. In this example I add a static route with destination **10.0.0.0/16** and next hop Azure Firewall.

`dependsOn: Virtual Hub, Azure Firewall`

{% highlight JSON %}
       {
      "type": "Microsoft.Network/virtualHubs/hubRouteTables",
      "apiVersion": "2020-05-01",
      "name": "contoso-vhub-westeurope/defaultRouteTable",
      "location": "westeurope",
      "properties": {
        "routes": [
          {
            "name": "toFirewall",
            "destinationType": "CIDR",
            "destinations": [
              "10.0.0.0/16"
            ],
            "nextHopType": "ResourceId",
            "nextHop": "[resourceId('Microsoft.Network/azureFirewalls', 'contoso-fw-westeurope')]"
          }
        ],
        "labels": [
          "default"
        ]
      }
    }
{% endhighlight %}

### RT_VNet  

This is what the **RT_VNet** Hub Route Table looks like. One single static route for destination **0.0.0.0/0** has been added to send all traffic to Azure Firewall, the route table is also tagged with the label `VNet`. [Labels](https://docs.microsoft.com/en-us/azure/virtual-wan/about-virtual-hub-routing#static) can be used to logically group route tables. I've added a dependsOn to the defaultRouteTable to avoid a conflict during deployment, the Virtual WAN hub does not like to run multiple routing changes at the same time.

`dependsOn: Virtual Hub, Azure Firewall, defaultRouteTable`

{% highlight JSON %}
    {
      "type": "Microsoft.Network/virtualHubs/hubRouteTables",
      "apiVersion": "2020-05-01",
      "name": "contoso-vhub-westeurope/RT_VNet",
      "properties": {
        "routes": [
          {
            "name": "toFirewall",
            "destinationType": "CIDR",
            "destinations": [
              "0.0.0.0/0"
            ],
            "nextHopType": "ResourceId",
            "nextHop": "[resourceId('Microsoft.Network/azureFirewalls', 'contoso-fw-westeurope')]"
          }
        ],
        "labels": [
          "VNet"
        ]
      }
    }
{% endhighlight %}

## Microsoft.Network/virtualHubs/hubVirtualNetworkConnections

Virtual Network Connections are created as a child resource to the Virtual Hub. A Virtual Network connection is in fact just VNet peering between the Virtual Hub and a spoke VNet and its routing configuration.

- `name` - Name of the VNet Connection, since it's a child resource make sure that the correct segments are used.
  - `remoteVirtualNetwork` - Resource ID of the VNet to peer with the Virtual Hub
  - `enableInternetSecurity` - This is a very important property when using custom route tables sending all internet traffic to Azure Firewall. Without this property the 0.0.0.0/0 route will not show up as an effective route on resources in the peered VNet.
  - `routingConfiguration` - Object with all routing configuration for the VNet connection.
    - `associatedRouteTable` - Resource ID to the associated route table for the VNet Connection. This is the route table that the VNet will learn all its routes from.
    - `propagatedRouteTables` - Object with configuration on route tables where the VNet connection are propagating routes.
      - `labels` - This is a really cool feature. The VNet connection will propagate routes to all route tables in the Virtual WAN that have the same label as defined in this property.
      - `ids` - Resource IDs to all Route Tables that the VNet Connection should propagate routes.

`dependsOn: Virtual Hub, Azure Firewall, Hub Route Table (RT_VNet)`

{% highlight JSON %}
    {
      "type": "Microsoft.Network/virtualHubs/hubVirtualNetworkConnections",
      "apiVersion": "2020-05-01",
      "name": "contoso-vhub-westeurope/spokeConnectionName",
      "properties": {
        "remoteVirtualNetwork": {
          "id": "[resourceId('Microsoft.Network/virtualNetworks', 'spokeVnetName')]"
        },
        "enableInternetSecurity": true,
        "routingConfiguration": {
          "associatedRouteTable": {
            "id": "[resourceId('Microsoft.Network/virtualHubs/hubRouteTables', 'contoso-vhub-westeurope/RT_VNet')]"
          },
          "propagatedRouteTables": {
            "labels": [
              "VNet"
            ],
            "ids": [
              {
                "id": "[resourceId('Microsoft.Network/virtualHubs/hubRouteTables', 'contoso-vhub-westeurope/RT_VNet')]"
              }
            ]
          }
        }
      }
    }
{% endhighlight %}

## Microsoft.Network/vpnSites

Time to define all physical locations to where we want to connect using site-to-site VPN. VPN Sites are added to the Virtual WAN Resource. In this example BGP is being used. A VPN Site is very much like a Local Network Gateway in a traditional VNet topology.

- `name` - Name of VPN Site
- `addressSpace` - IP address space that is located on your on-premises site
  - `addressPrefixes` - Array with all IP Prefixes
- `bgpProperties` - BGP Properties object.
  - `asn` - AS Number for the physical location
  - `bgpPeeringAddress` - BGP Peer IP Address (this is not the public IP of the VPN Device)
  - `peerWeight` - Peer Weight
- `deviceProperties` - Device Properties Information. This is just pure metadata used by the Azure Team better understand your environment to add additional optimization possibilities in the future, or to help you troubleshoot.
  - `deviceVendor` - Name of the Device Vendor (for example: Citrix, Cisco, Barracuda)
  - `deviceModel` - Device model name
  - `linkSpeedInMbps` - Link speed in Mbps
- `ipAddress` - VPN Device public IP Address
- `virtualWan` - Virtual WAN Resource ID.

`dependsOn: Virtual WAN`

{% highlight JSON %}
    {
      "type": "Microsoft.Network/vpnSites",
      "apiVersion": "2020-05-01",
      "name": "onprem-vpnsite",
      "location": "westeurope",
      "properties": {
        "addressSpace": {
          "addressPrefixes": "10.20.0.0/24"
        },
        "bgpProperties": {
          "asn": 65010,
          "bgpPeeringAddress": "10.128.0.10",
          "peerWeight": 0
        },
        "deviceProperties": {
          "deviceVendor": "Microsoft",
          "deviceModel": "AzureVPNGateway",
          "linkSpeedInMbps": 10000
        },
        "ipAddress": "92.34.128.10",
        "virtualWan": {
          "id": "[resourceId('Microsoft.Network/virtualWans', 'contoso-vwan')]"
        }
      }
    }
{% endhighlight %}

## Microsoft.Network/vpnGateways

Time for the VPN Gateway, a quite simple resource. I don't define any VPN Connections in the VPN Gateway resource since it will be a mess with multiple tunnels.

- `name` - Name of VPN Gateway
- `virtualHub` - Resource ID of the Virtual Hub where the VPN Gateway should be created.
- `bgpSettings` - BGP Settings object
  - `asn` - VPN Gateway AS Number
- `vpnGatewayScaleUnit` - Number of VPN Gateway Scale Units

`dependsOn: Virtual WAN, Virtual Hub, Azure Firewall`

{% highlight JSON %}
    {
      "type": "Microsoft.Network/vpnGateways",
      "apiVersion": "2020-05-01",
      "name": "contoso-vhub-westeurope-vpngw",
      "location": "westeurope",
      "properties": {
        "virtualHub": {
          "id": "[resourceId('Microsoft.Network/virtualHubs', 'contoso-vhub-westeurope')]"
        },
        "bgpSettings": {
          "asn": 65515
        }
      }
    }
{% endhighlight %}

## Microsoft.Network/vpnGateways/vpnConnections

Time to connect the VPN Gateway with the VPN Site. This is usually a quite simple resource to declare, but there are a lot of other properties available that could be used. An example is `ipsecPolicies` that is used to configure a [custom IPsec policy](https://docs.microsoft.com/en-us/azure/virtual-wan/virtual-wan-custom-ipsec-portal).

- `name` - Name of VPN Connection. It's a child resource, make sure that the segments are correct.
- `connectionBandwidth` - Connected site bandwidth
- `enableBgp` - Property to enable BGP
- `sharedkey` - Pre-shared Key for the site-to-site connection. If no PSK is provided a key will automatically be generated by Azure.
- `remoteVpnSite` - Resource ID to the VPN Site to connect to.

`dependsOn: Virtual WAN, Virtual Hub, Azure Firewall, VPN Gateway`

{% highlight JSON %}
{
      "type": "Microsoft.Network/vpnGateways/vpnConnections",
      "apiVersion": "2020-05-01",
      "name": "contoso-vhub-westeurope-vpngw/HubToOnPremConnection",
      "properties": {
        "connectionBandwidth": 10,
        "enableBgp": true,
        "sharedKey": "Ox4ZCYPn8hCgnZjHDPxkllNgVFE/poTS",
        "remoteVpnSite": {
          "id": "[resourceId('Microsoft.Network/vpnSites', 'onpremvpnsitename')]"
        }
      }
    }
{% endhighlight %}

## Summary

That's all for this post! There are some other resources available for Virtual WAN as well like Express Route Gateway, User VPN Gateway, User VPN Configuration etc. but that´s something we can save for another time. I hope that the information in this post is enough to get you started with your Azure Virtual WAN deployments.

<a class="github-button" href="https://github.com/StefanIvemo/vwan-playground/blob/main/playground/main.json" aria-label="Full ARM Template Example">Full ARM Template Example</a>

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