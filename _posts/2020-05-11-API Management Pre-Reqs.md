---
layout: post
title: Deploy APIM like a champ! - Part 1 - Prerequisites
---

I've been working a lot with Azure API Management supporting developers with the surrounding infrastructure like VNets (for APIM in VNet mode), Application Gateways, ARM Templates, certificates, identity providers, backup, custom RBAC etc. Over the years I've learned a lot of things regarding APIM, some of them undocumented, sometimes poorly documented. I want to share some of the tips and tricks I've learned in a series of posts on the topic "Deploy APIM like a champ" and this is the first part of the series.

The blog series
------
So, what is this blog series really all about? I'm going to share my experience around APIM deployment and configuration, both basic stuff and more advanced things. I will start by deploying a simple APIM Service and then add more and more configuration along the way until we have a really good APIM infrastructure up and running. I will use ARM Templates wherever possible to make sure that we can manage APIM using CI/CD and Azure Pipelines.

All templates, scripts and code used in the series can be found in my GitHub repo StefanIvemo/APIM.  

<a class="github-button" href="https://github.com/StefanIvemo/APIM/" aria-label="Use this template StefanIvemo/APIM on GitHub">StefanIvemo/APIM</a>

Prerequisites
------

Let's start with the prerequisites for API Management. They're not really prerequisites to deploy the service itself, but they're needed to simplify the administration of APIM when using custom domain names which we want to do.

The things we will do in Part 1 are:

1. Deploy an Azure Key Vault - Used to store our SSL certificates for APIM Custom Domain Names.
2. Define our custom domain names for APIM
3. Setup automated issuing and renewal of certificates using Let's Encrypt and Key Vault Acmebot.

With these things in-place we'll be able to deploy the APIM service with custom domain names which we will do in Part 2.

Deploy Azure Key Vault
-----
First thing we need to do is to create an Azure Key Vault. The template I use is a quite simple template that basically just deploys the Key Vault, but there are some minor settings that are important later on.

1. Azure Key Vault is enabled for ARM template deployment. This is needed to access the Key Vault during template deployments, and we'll do that when we deploy the APIM Service.
2. Soft Delete is enabled on the Key Vault. This is a requirement when using certificates from an Azure Key Vault in an Azure Application Gateway that will be used in-front of APIM configured in internal VNet mode later on (and it's a good idea to always enable soft delete for your Key Vaults).
3. Assign an access policy with full permissions to my AAD account for administration.

<a class="github-button" href="https://github.com/StefanIvemo/APIM/blob/master/Misc/KeyVault.json" aria-label="Use this template StefanIvemo/APIM on GitHub">Use this template</a>

{% highlight JSON %}
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "kvname": {
            "type": "string"
        },
        "location": {
            "type": "string"
        },
        "tenantID": {
            "type": "string"
        },
        "adminID": {
            "type": "string",
            "metadata": {
                "description": "ObjectID of Key Vault Admin account"
            }
        }
    },
    "variables": {
    },
    "resources": [
        {
            "apiVersion": "2018-02-14",
            "name": "[parameters('kvname')]",
            "location": "[parameters('location')]",
            "type": "Microsoft.KeyVault/vaults",
            "properties": {
                "enabledForDeployment": false,
                "enabledForTemplateDeployment": true,
                "enabledForDiskEncryption": false,
                "enableRbacAuthorization": false,
                "accessPolicies": [
                    {
                        "objectId": "[parameters('adminID')]",
                        "tenantId": "[parameters('tenantID')]",
                        "permissions": {
                            "keys": [
                                "Get",
                                "List",
                                "Update",
                                "Create",
                                "Import",
                                "Delete",
                                "Recover",
                                "Backup",
                                "Restore"
                            ],
                            "secrets": [
                                "Get",
                                "List",
                                "Set",
                                "Delete",
                                "Recover",
                                "Backup",
                                "Restore"
                            ],
                            "certificates": [
                                "Get",
                                "List",
                                "Update",
                                "Create",
                                "Import",
                                "Delete",
                                "Recover",
                                "Backup",
                                "Restore",
                                "ManageContacts",
                                "ManageIssuers",
                                "GetIssuers",
                                "ListIssuers",
                                "SetIssuers",
                                "DeleteIssuers"
                            ]
                        },
                        "applicationId": null
                    }
                ],
                "tenantId": "[parameters('tenantID')]",
                "sku": {
                    "name": "Standard",
                    "family": "A"
                },
                "enableSoftDelete": true,
                "softDeleteRetentionInDays": 30,
                "networkAcls": {
                    "defaultAction": "allow",
                    "bypass": "AzureServices",
                    "ipRules": [
                    ],
                    "virtualNetworkRules": [
                    ]
                }
            },
            "tags": {
            },
            "dependsOn": [
            ]
        }
    ],
    "outputs": {
    }
}
{% endhighlight %}

Custom Domain names
-----
I use custom domain names with APIM instead of the default *.azure-api.net names. I will use the following custom domain names for my APIM endpoints, use domain names that makes sense for you.  

| Domain name   | Endpoint      |
| :------------- |:-------------|
| api.doublerdiner.dev     | Gateway |
| developerportal.dobulerdiner.dev   | Developer Portal      |
| legacydeveloperportal.doublerdiner.dev | Developer Portal (Legacy)     |
| apimmanagement.doublerdiner.dev | Management    |
| apimscm.doublerdiner.dev | SCM    |  

Key Vault Acmebot
-----
The last preparation is to get hold of some SSL Certificates for our Custom Domain Names. APIM supports certificates from any known public CA, so pick one that suits your environment/organization. In my lab I use Let's Encrypt for all my certificates and I've configured automated issuing and renewal of Let's Encrypt SSL/TLS certificates using [Key Vault Acmebot](https://github.com/shibayan/keyvault-acmebot). Key Vault Acmebot is built on Azure Functions and requires your DNS Zone to be hosted in Azure DNS. The reason for this is because the Let's Encrypt validation method used for the Key Vault Acmebot is [DNS-01 Challenge](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge). Since I have my DNS Zone for my domain doublerdiner.dev hosted in Azure I'm good to go.  
  
I'm not going to go through the actual steps of deploying the solution since a deployment guide can be found in the Key Vault Acmebot repository. But I will show you what it looks like when it's deployed and configured and how the issuing process works.  

The steps below are a simplified explanation of the Key Vault Acmebot issuing and how Let’s Encrypt works:

1. Navigate to https://YOUR-FUNCTIONS.azurewebsites.net/add-certificate and add all the domain names that you need certificates for and click on submit.

<img src="https://github.com/StefanIvemo/stefanivemo.github.io/blob/master/images/apim-part1/Part1-acmebotissue.png?raw=true">  

2. The first thing that needs to happen before the certificates can be issued is that the Key Vault Acmebot Function has to prove to the CA that it controls the domain, in my case doublerdiner.dev.  
    1. The Key Vault Acmebot Function asks the Let’s Encrypt CA what it needs to do in order to prove that it controls the domain (doublerdiner.dev).  
    2. Let’s Encrypt CA will look at the domain name being requested and issue one or more sets of challenges (HTTP-01 or DNS-01). These are different ways that the agent can prove control of the domain.  
    3. Along with the challenges, the Let’s Encrypt CA also provides a nonce that the agent must sign with its private key pair to prove that it controls the key pair.
    4. Since Key Vault Acmebot is built to use the DNS-O1 challenge the Azure Function creates a couple of TXT DNS Records in the DNS Zone for doublerdiner.dev in the _acme-challenge.<YOUR_DOMAIN> format with the value set to tokens from Let’s Encrypt CA.  
    <img src="https://github.com/StefanIvemo/stefanivemo.github.io/blob/master/images/apim-part1/part1-dnsverification.png?raw=true">  
    5. When the Key Vault Acmebot have completed the DNS Record Creation it notifies the Let’s Encrypt CA that it's ready to complete validation.
    6. The CA verifies the signature on the nonce, and it attempts to resolve the DNS Names and make sure it has the expected content.
    7. If the signature over the nonce is valid, and the challenges check out, then the agent identified by the public key is authorized to do certificate management for doublerdiner.dev

3. When the domain validation steps are done it's time to get the certificates issued.  
    1. The Key Vault Acmebot Function constructs CSRs for the requested domain names and asks the Let’s Encrypt CA to issue a certificates for doublerdiner.dev with a specified public key. The agent also signs the whole CSR with the authorized key for doublerdiner.dev so that the Let’s Encrypt CA knows it’s authorized.
    2. When the Let’s Encrypt CA receives the request, it verifies both signatures. If everything looks good, it issues a certificate for example.com with the public key from the CSR and returns it to the Key Vault Acmebot Function.  
    3. The Key Vault Acmebot Function adds the certificates to the Azure Key Vault.

4. And that's it, the certificates are now issued and can be found in my Key Vault, with automatic renewal in-place.  
<img src="https://github.com/StefanIvemo/stefanivemo.github.io/blob/master/images/apim-part1/part1-keyvaultcerts.png?raw=true">

If you're interesting in learning more about how Let's Encrypt works [read this](https://letsencrypt.org/how-it-works/).

Summary
-----
That's it! Now all prerequsites are inplace to get started with the APIM deployment, stay tuned for Part 2.

<script src="https://utteranc.es/client.js"
        repo="StefanIvemo/stefanivemo.github.io"
        issue-term="pathname"
        label="Comment"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>

<script async defer src="https://buttons.github.io/buttons.js"></script>