---
layout: post
title: Introducing Azure Virtual WAN Playground!
---
Last week I decided to share my Azure Virtual WAN lab environment by publishing it on [GitHub](https://github.com/StefanIvemo/vwan-playground), this blog post is an introduction to my so called **Azure Virtual WAN Playground!** 

## What is Azure Virtual WAN Playground?
If you've been using Azure Virtual WAN you know that it's quite expensive to run for lab purposes and not something you just leave running in your personal Azure Subscription. And deploying it using the Portal by enabling one feature at the time takes a lot of time and you'll probably spend a day just to get your lab environment up and running. I wanted to make sure I had a lab environment that looked the same every time I had to play around with Virtual WAN, was quick and easy to deploy and could be removed as soon as I was done to save me some Azure Credit. The outcome of this work is what I like to call **Azure Virtual WAN Playground**.

## How it's built
When I started working with Azure Virtual WAN a while back I was using ARM Templates for deployment. One of the challenges with Virtual WAN and writing the ARM template are all the references and dependencies between resources, it will drive you crazy! There are also some big differences between API Versions, for example moving from `2020-04-01` to `2020-05-01` completely changes the `Microsoft.Network/virtualHubs` resource. I ended up spending a lot of time maintaining and developing the template. 

Then 💪[Bicep language](https://github.com/Azure/bicep) was introduced and released in Alpha a month ago! **Bicep** is a Domain Specific Language (DSL) for deploying Azure resources declaratively. It aims to drastically simplify the authoring experience with a cleaner syntax and better support for modularity and code re-use. Sounds awesome, right!? I decided to give it a shot and see how it could improve my Virtual WAN Deployment. I know Bicep is still in Alpha but I love it already! I work with ARM templates almost every day and using **Bicep** is such an improvement. I use it all the time now when I have to build a new template, just write the **Bicep** code, run a build and an ARM Template gets generated. If I need to use a function that isn't supported yet like `copy` loops or `conditions` I add it to my complied template and I'm good to go. Enough about **Bicep** we'll cover that some other time!

## Virtual WAN Playground Topology
At the time of writing this blog post the Virtual WAN Playground contains the following resources (this will change over time 😊):

- Azure Virtual WAN
  - Virtual WAN Hub (Secured Virtual Hub)
  - Firewall Policy
  - Azure Firewall
  - Custom Hub Route Table (Associated with VNet Connection)
  - Virtual Network Connection (Spoke VNet <> Virtual Hub)
    - Using custom hub route table routing branch and internet traffic to Azure Firewall
  - VPN Gateway
  - VPN Site (On-Prem VNet)
- Spoke VNet
  - Azure Bastion Service
  - Virtual Machine
- On-Prem VNet
  - Azure Bastion Service
  - Virtual Machine
  - VPN Gateway
    - Connection (On-Prem to Virtual Hub)
    - Local Network Gateway
- Log Analytics Workspace (Firewall Diagnostics)

<img src="https://github.com/StefanIvemo/stefanivemo.github.io/blob/master/images/vwan-playground/vwan-playground-topology.png?raw=true"> 

## Improvements
Virtual WAN Playground is an ongoing project and new functionality will be added over time. Some of my planned improvements are:

- When Bicep language supports the `condition` property it will be possible to decide which features to deploy using parameters.
- A cleanup script will be added that removes all resources in the correct order for quick removal of the solution.
- User VPN Gateway
    - User VPN Configuration (AAD Auth).
    - User VPN Configuration (Cert Auth).
- ExpressRoute Gateway
- Additional Spoke VNets (when Bicep language supports `copy`)
- Azure Firewall default rule set added to Firewall Policy.
- Azure Firewall Workbook added to Log Analytics Workspace.
- Azure Sentinel.
- Ability to create additional VPN Sites

Summary
------
Hope you find this useful! Stay tuned for a deep dive into how to deploy Virtual WAN using templates and how to avoid your deployment from failing. In the meantime start deploying your Virtual WAN Playground!

<a class="github-button" href="https://github.com/StefanIvemo/vwan-playground" aria-label="Azure Virtual WAN Playground!">Azure Virtual Wan Playground!</a>

<script src="https://utteranc.es/client.js"
        repo="StefanIvemo/stefanivemo.github.io"
        issue-term="pathname"
        label="Comment"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>

<script async defer src="https://buttons.github.io/buttons.js"></script>