---
layout: post
title: Draft - Deploy Azure Virtual WAN using templates.
---

Deploying Azure Virtual WAN using ARM templates can be frustrating and it takes some time before you get a hang on it. There are a lot of dependencies between resources and you need to make sure that everything is deployed in the correct order and allow the Virtual Hub and routing to reach succeeded state before throwing more things at it. In this post I will guide you through how to build a Virtual WAN template!

## API Version
For ARM templates, you must always specify an API version for every resource type you will deploy. The API version corresponds to a version of REST API operations that are released by the resource provider e.g `Microsoft.Network`. When a resource provider enables new features a new API Version is released. Sometimes a new API Version just enables a couple of new features, and you can just change it to the most recent version and your template will still function properly without any other modifications. But that's not always the case, in some cases a new API Version completely changes how some resources are deployed. When it comes to Azure Virtual WAN the change from API Version `2020-04-01` to `2020-05-01` equals a lot of changes. If you see an example template using API Version `2020-04-01` you can't just update the API Version and expect a successful deployment.

The following things are good to know whe it comes to `Microsoft.Network` API Versions and Virtual WAN:

- `2020-04-01` - Do not use this version if you´re building a new template, you will have to do a lot of modifications when upgrading to a newer version.
- `2020-05-01` - The API Version I use for Virtual WAN Playground. Some of the major changes are:
    - `Microsoft.Network/virtualHubs` has changed a lot. Properties that in earlier versions was declared as properties inside the Virtual Hub are now child resources:
        - `Microsoft.Network/virtualHubs/hubVirtualNetworkConnections`
        - `Microsoft.Network/firewallPolicies`
        - 
- `2020-06-01` - Released but undocumented API Version.

# Breaking down the template - Resource by resource
I have a full Virtual WAN lab envrionment 




<script src="https://utteranc.es/client.js"
        repo="StefanIvemo/stefanivemo.github.io"
        issue-term="pathname"
        label="Comment"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>

<script async defer src="https://buttons.github.io/buttons.js"></script>