---
layout: post
title: Deploy Azure APIM like a champ - Part 1 - Prerequisites
---

I've been working a lot with Azure API Management supporting developers with the surrounding infrastructure like vNets (for APIM in vNet mode), certificates, Application Gateways, backup, custom RBAC etc. Over the years I've learned a lot of things regarding APIM, some of them undocumented or poorly documented. I want to share some of the tips and tricks I've learned in a series of posts on the topic "Deploy Azure API Management like a champ" and this is the first part of the series.

Prerequisites
------

Let's start with the prerequisites for API Management. They're not really prerequisites to deploy the service itself, but they're needed to simplify the administration of APIM.

The things we will configure in Part 1 are:

1. Deploy Managed Identity (User Assigned) - Used to list and get certificates from Azure Key Vault
2. Deploy Azure Key Vault - Used to store our SSL certificates for APIM Custom Domain Names
3. Get some SSL Certificates for our APIM Custom Domain Names

This is our starting point to deploy the APIM service with custom domain names, during this blog series we'll add more features and services around it.

Deploy Managed Identity
-----

The Managed Identity (User Assigned) will be used by APIM to GET Certificates from a Key Vault. The reason for using a User Assigned Managed Identity is that it can be reused by other Azure services as well. Later on in another part of this blog series we will use it to manage certificates for an Application Gateway as well using the same Managed Identity.

I'm using a simple template where I only provide a location and name during deployment.

````JSON
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
````

Deploy Azure Key Vault
-----

Now that we have our Managed Identity we can go ahead and create the Azure Key Vault. During deployment an Access Policy is assigned giving the Managed Identity Get and List permissions to both Secrets and Certificates. Certificates will be used for the Custom Domain names and secrets used by API Policys to retrieve secrets used by our APIs. 

````JSON
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
        "tenantID": {
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
````

Certificates
-----

To continue with deployment we also need to get some SSL Certifiacates for our Custom Domain names. APIM supports certificates from any known public CA. I prefer to use Let's Encrypt and I've setup automated issuing and renewal of Let's Encrypt SSL/TLS certificates using [Key Vault Acmebot](https://github.com/shibayan/keyvault-acmebot). I'm not going to go through the actual steps of configuring it since a deployment guide can be found in the Key Vault Acmebot repo.

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