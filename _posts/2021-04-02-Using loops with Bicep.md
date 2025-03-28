---
layout: post
title: Using loops with Bicep! 💪
author: Stefan Ivemo
summary: In this post I cover multiple ways to use copy loops with Bicep!
---

With 💪[Bicep](https://github.com/Azure/bicep) v0.3 we finally got loop support and now we can start to do some really cool stuff using Bicep. I must say I really like the new loop syntax, it's clean, easy to read and removes a lot of the complexity we did see in ARM template copy loops. This post is dedicated to copy loops where I will show some different examples on how you can use loops in Bicep. The Bicep Loops specification can be [found here](https://github.com/Azure/bicep/blob/main/docs/spec/loops.md).

# Loops

With loops we can iterate over an array of objects to create multiple resources based on the objects in our array. We can also use loops to set an array property, loop outputs and variables. It's even possible use loops with module deployments.

A loop is declared using the for-expression and we can iterate over arrays declared by a parameter, variable or using the `range()` function.

For-expression example:
<script src="https://gist.github.com/StefanIvemo/1e8c849b5752991a0ebc165b96695832.js"></script>

Inside the loop body a new scope is created. Inside this inner-scope we can access any identifier declared outside of this scope, but any identifier declared in the inner-scope will not be available outside of it. Any resource, variable or parameter declared at the outer-scope may be used within a loop body. We can even use nested loops and use filtering via the `if` keyword.

## 1. Resource and module loops

Lets start with a simple and common way to use loops in templates and create some Virtual Networks. We have an array variable called `vnets`, it has two objects, both with a `name` and `addressPrefix` key/value pair. We are iterating over the `vnets` variable array and for each loop iteration we create a Virtual Network with the `name` and `addressPrefix` from the current array item. Pretty simple and straight forward.

<script src="https://gist.github.com/StefanIvemo/d2c6ac6113e8c279406fb816717837c5.js"></script>

It is also possible to use loops with modules. In this example we are iterating over the same `vnets` variable array as above, but instead of passing the values from the current array index to the properties of the resource, they are used in the `params` section of the module.

<script src="https://gist.github.com/StefanIvemo/582a147399e776c4d2a7ce84221820e8.js"></script>

## 2. Filtered resource loops

We can also use filters for our loops if we want to exclude objects that doesn't meet our conditions by using the `if` keyword. Filters must be expressions that evaluate to a boolean value (more info on expression can be [found here](https://github.com/Azure/bicep/blob/main/docs/spec/expressions.md)). In this example I've added a property to the objects in my array named `enabled` with the values `true` and `false`. This example will create a single Virtual Network named `landingzone-1-vnet` since it's the only one that meet my condition.

<script src="https://gist.github.com/StefanIvemo/beebb7ef5ffec548fa6ba797acafd1b0.js"></script>

## 3. Loop index

We have multiple ways of working with the loop index. The first way to write an index-based loop is to use the `range()` function. The range function creates an array of integers from a starting integer, and containing the number of items we declare for the `count` parameter (`range(startIndex, count)`). The `startIndex` parameter defines the first integer in the array, and `count` the number of integers in the array. In the example below we´re using `range(1,4)`, this means that the first integer in the array will be 1 and we want to count to 4 integers. This example will create 4 Virtual Networks where we use the current loop index (`i`) in both the name and addressPrefix.

<script src="https://gist.github.com/StefanIvemo/7e520153821f7b95368321718ed3314e.js"></script>

You can also get the index when iterating over an array of objects. Looking at the example below there is an array variable called `vnets` with three objects. Within the loop body, `vnet` stores the current element from the array and `i` stores the 0-based index of the current array element. Both are then referenced from within the loop body. Since the index is 0-based when iterating over an array of objects I've just added `+1` to the index (`${i+1}`) to make sure that my first VNet name gets the sequence number 1 instead of 0.

<script src="https://gist.github.com/StefanIvemo/87b7c53b58f0e2991a4abc2e6d135347.js"></script>

## 4. Array Property Loop

Array properties inside a resource declaration can be defined using loops, and this can be very useful in some cases. Subnets inside a Virtual Network is one of them. Because of how the `virtualNetworks` API work we get an issue if we deploy our subnets as a child resources and want our template to be re-deployable. If I redeploy a Virtual Network template where I have declared the subnets as child resources, it will first try to delete the subnets because my Virtual Network is deployed with an empty `subnets` property. If I have a resource using the subnet of the VNet it will fail to delete it, but if the subnet is empty it will be removed and re-created. This is not the behavior we want. Instead we can use a property loop to declare multiple subnets (or variable loops which are covered below).

> NOTE: I do believe that the `virtualNetworks` API will be updated in future API releases to behave differently with the `subnets` property. You should be able to re-deploy the VNet excluding the `subnets` property without having the subnets deleted. But only time will tell.

So, how do we use property loops then? In this example we create one Virtual Network and three subnets using a property loop. On each loop iteration, the `subnets` property of the Virtual Network is set to the current element of the array.

<script src="https://gist.github.com/StefanIvemo/8846cee7863fd3e898d773cb0443eea6.js"></script>

## 5. Nested Loops

It is also possible to use copy loops inside another loop. In the example below we are iterating over the array parameter `vnets` to create multiple Virtual Networks, and then for each Virtual Network we are also creating the subnets with another nested copy loop.

<script src="https://gist.github.com/StefanIvemo/1f6b1737fd86ecacf9f6cf485cc08186.js"></script>

## 6. Batch size

Batch size can be used to control the order that resources are created when using loops. By default resources will be deployed concurrently in a non-deterministic order at runtime. By using the @batchSize decorator the resources or modules in the for-expression will be deployed sequentially in batches of the specified size. Each batch will be deployed concurrently. Set the @batchSize to 1 if you want to make sure that the deployment is sequential.

The following example deploys 9 Virtual Networks, 3 at a time:

<script src="https://gist.github.com/StefanIvemo/766adfda4ec353b9f6eee69de6063b32.js"></script>

## 7. Output Loops

It's not only for resources and properties that we want to use copy loops, sometimes we need to loop the output as well. Directly referencing a resource module or module collection is not currently supported in output loops. In order to loop outputs we need to apply an array indexer to the expression (like we did in the index examples above).

In the example below I will create 2 Virtual Networks and output `name`, `resourceId` and `addressPrefixes` for each deployed VNet.

<script src="https://gist.github.com/StefanIvemo/d5c4b2733e1b499eb1d03c87738c4844.js"></script>

## 8. Variable loops

This is my favorite kind of loop in templates, with variable loops we can do a lot of cool stuff. I find them very useful when I want to construct an array of objects that I can use for a resource property. I prefer doing this over property loops that we looked at above, because I like that the template becomes a lot cleaner when I don't stick to much logic into the resource declaration it self.

Example variable loop:
<script src="https://gist.github.com/StefanIvemo/c6a6f53434a0b65ead5aef2f1d0265f3.js"></script>

Let's look at an Azure Firewall deployment where the number of public IPs is based on the integer parameter `publicIpCount`. The `ipConfigurations` array property in an Azure Firewall consists of one object per IP configuration, where the first IP configuration is referencing both the resourceId to the `AzureFirewallSubnet` in our Virtual Network, and also the resourceId to the first public IP address assigned to the Firewall. For all additional public IPs I want to add to my Firewall I only specify the public IP resourceId in the array object. The `ipConfigurations` array will look like this:

<script src="https://gist.github.com/StefanIvemo/4b31f965de9086dd6d4a77109c2428f6.js"></script>

Here's the full example of an Azure Firewall deployment using variable loops (existing VNet):
<script src="https://gist.github.com/StefanIvemo/0d6635bc21256aedb300981c0a7eacdd.js"></script>

# Summary

That was all I had about loops for now, hope you liked it! Who knows, another post with a massive Bicep example on the loop topic might show up here in a couple of weeks.

All examples in this post can be found here: 
<a class="github-button" href="https://github.com/StefanIvemo/BicepThings/tree/main/Loops" aria-label="StefanIvemo/BicepThings on GitHub">All examples</a>


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