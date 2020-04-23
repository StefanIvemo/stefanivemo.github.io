---
layout: post
title: Deploy APIM like a champ - Part 1 - Prerequisites
---

I've been working a lot with Azure API Management supporting developers with the surrounding infrastructure like VNets (for APIM in VNet mode), Application Gateways, certificates, backup, custom RBAC etc. Over the years I've learned a lot of things regarding APIM, some of them undocumented or poorly documented. I want to share some of the tips and tricks I've learned in a series of posts on the topic "Deploy APIM like a champ" and this is the first part of the series.

Prerequisites
------

Let's start with the prerequisites for API Management. They're not really prerequisites to deploy the service itself, but they're needed to simplify the administration of APIM when using custom domain names.

The things we will do in Part 1 are:

1. Deploy a Managed Identity (User Assigned) - Used by APIM to list and get certificates from Azure Key Vault
2. Deploy an Azure Key Vault - Used to store our SSL certificates for APIM Custom Domain Names
3. Define our custom domain names and get some SSL Certificates for them

This is our starting point to be able to deploy the APIM service with custom domain names. During the blog series we'll add more and more features to APIM until we have an awesome setup.

Deploy Managed Identity
-----

The Managed Identity (User Assigned) will be used by APIM to GET Certificates from a Key Vault. The reason for using a User Assigned Managed Identity is that it can be reused by other Azure services as well. Later on in another part of this blog series we will use it to manage certificates for an Application Gateway using the same Managed Identity.

The template to deploy the management is simple, I only provide a location and name during deployment.

<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/stefanivemo/APIM/generate" data-size="large" aria-label="Use this template stefanivemo/APIM on GitHub">Use this template</a>

{% highlight JSON %}
{
    "$schema": "http://schema.management.azure.com/schemas/2014-04-01-preview/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "name": {
            "type": "string"
        },
        "location": {
            "type": "string"
        }
    },
    "resources": [
        {
            "apiVersion": "2018-11-30",
            "name": "[parameters('name')]",
            "location": "[parameters('location')]",
            "type": "Microsoft.ManagedIdentity/userAssignedIdentities",
            "properties": {}
        }
    ]
}
{% endhighlight %}

Deploy Azure Key Vault
-----

Now that we have our Managed Identity we can go ahead and create the Azure Key Vault. During deployment an Access Policy is assigned giving the Managed Identity we created Get and List permissions to both Secrets and Certificates. Certificates will be used for the Custom Domain names and secrets used by API Policys to retrieve secrets used by our APIs. Pass the ObjectID of you managed identity and your Azure AD Tenant ID to the parameters ObjectID and TenantID.

{% highlight JSON %}
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "name": {
            "type": "string"
        },
        "location": {
            "type": "string"
        },
         "ObjectID": {
            "type": "string"        
        },  
        "TenantID": {
            "type": "string"        
        }   
      
    },
    "variables": {},
    "resources": [
        {
            "apiVersion": "2018-02-14",
            "name": "[parameters('name')]",
            "location": "[parameters('location')]",
            "type": "Microsoft.KeyVault/vaults",
            "properties": {
                "enabledForDeployment": false,
                "enabledForTemplateDeployment": true,
                "enabledForDiskEncryption": false,
                "enableRbacAuthorization": false,
                "accessPolicies": [
 {
                    "objectId": "[parameters('ObjectID')]",
                    "tenantId": "[parameters('TenantID')]",
                    "permissions": {
                        "keys": [],
                        "secrets": [
                            "Get",
                            "List"
                        ],
                        "certificates": [
                            "Get",
                            "List"
                        ]
                    }
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
                    "ipRules": [],
                    "virtualNetworkRules": []     
                }
            },
            "tags": {},
            "dependsOn": []
        }
    ],
    "outputs": {}
}
{% endhighlight %}

Certificates
-----

To continue with deployment we also need to get some SSL Certifiacates for our Custom Domain names. APIM supports certificates from any known public CA. I prefer to use Let's Encrypt because it's free, and I've setup automated issuing and renewal of Let's Encrypt SSL/TLS certificates using [Key Vault Acmebot](https://github.com/shibayan/keyvault-acmebot). I'm not going to go through the actual steps of configuring it since a deployment guide can be found in the Key Vault Acmebot repository.

The custom domain names I will use for my deployment are:

| Domain name   | Endpoint      |
| :------------- |:-------------|
| api.doublerdiner.dev     | Gateway |
| developerportal.dobulerdiner.dev   | Developer Portal      |
| legacydeveloperportal.doublerdiner.dev | Developer Portal (Legacy)     |
| apimmanagement.doublerdiner.dev | Management    |

Summary
-----
That's it! Now all prerequsites are inplace to get started with the APIM deployment, stay tuned for Part 2.