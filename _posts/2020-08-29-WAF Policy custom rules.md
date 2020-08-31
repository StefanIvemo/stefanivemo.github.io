---
layout: post
title: Bypassing custom rules using the RequestHeaders match variable in WAF v2
---

I had a case the other day where a custom rule in a Web Application Firewall v2 policy attached to an Application Gateway behaved kind of funky. The rule was setup to deny traffic if a specific request header in the HTTP request was not present. At first everything looked good but after a while I still noticed that some unwanted traffic was hitting my backend service. After some testing and investigation, I came up with the following. Thanks [@SimonWahlin](https://twitter.com/SimonWahlin) for the support!


The custom rule gotcha!
-----
If you are new to custom rules for Web Application Firewall v2, read [this](https://docs.microsoft.com/en-us/azure/web-application-firewall/ag/custom-waf-rules-overview) article on docs.microsoft.com, it will give you a good introduction on how they work.  


Let's say we have a web application protected by an Application Gateway with WAF v2 enabled. The web application has one job, process a web request and send a response. The request header for a valid request contains the header `Content-Type: application/secret-request`. To only allow valid traffic to my backend I want to create a custom rule that denies all traffic without this specific header in the request. It should work with a single block rule denying all traffic that does not contain the `Content-Type: application/secret-request` header, at least that's my impression after reading the documentation and the [custom rule examples](https://docs.microsoft.com/en-us/azure/web-application-firewall/ag/create-custom-waf-rules), but lets try it out.  

In order to test this I have setup an Application Gateway with the WAF v2 SKU, created a WAF policy and attached it to the App GW, configured the App GW to publish a Web Application running on an App Service over port 80 on the DNS name `secret.ivemo.se`.  
<br/>
<img src="https://github.com/StefanIvemo/stefanivemo.github.io/blob/master/images/waf-gotcha/appgw-overview.png?raw=true">
<br/><br/>
Lets create a custom rule with the following settings:

- **Name** - DenyNonSecretRequests
- **Priority** - 10
- **Rule type** - MatchRule
- **Match variable** - RequestHeaders
- **Selector** - Content-Type
- **Operator** - Equal
- **Negate condition** - Is not
- **Transform** - Lowercase
- **Match values** - application/secret-request
- **Action** - Block

Below is how the rule is created using PowerShell:

{% highlight powershell %}
 $variable = New-AzApplicationGatewayFirewallMatchVariable `
   -VariableName RequestHeaders `
   -Selector Content-Type

$condition = New-AzApplicationGatewayFirewallCondition `
   -MatchVariable $variable `
   -Operator Equal `
   -MatchValue "application/secret-request" `
   -Transform Lowercase `
   -NegationCondition $true

$rule = New-AzApplicationGatewayFirewallCustomRule `
   -Name DenyNonSecretRequests `
   -Priority 10 `
   -RuleType MatchRule `
   -MatchCondition $condition `
   -Action Block
{% endhighlight %}


With the rule in-place lets perform some tests to see it in action. 

Sending a request with the header `Content-Type: application/secret-request` to my web services looks good traffic is allowed. 

<img src="https://github.com/StefanIvemo/stefanivemo.github.io/blob/master/images/waf-gotcha/1CurlCorrectHeader.PNG?raw=true">


Now let's test again by sending the header `Content-Type: application/public-request`, all looks good again and the traffic is blocked by the WAF!
 
<img src="https://github.com/StefanIvemo/stefanivemo.github.io/blob/master/images/waf-gotcha/2CurlWrongHeader.PNG?raw=true">

Awesome! The rule seems to work as planned, traffic without the correct request header `Content-Type: application/secret-request` is not allowed through the WAF. But here is where it all gets a little bit funky! It might be that I have misunderstood how it's supposed to work, or it might be a bug in the rule processing.  

Lets send one more request to my web service, this time I will send the request with no `Content-Type:` header at all.

<img src="https://github.com/StefanIvemo/stefanivemo.github.io/blob/master/images/waf-gotcha/3CurlNoHeader.PNG?raw=true">


Oops! My custom rule has been bypassed. I have tried this with other request headers as well like User-Agent with the exact same result. If the header is not present at all the traffic will go through the WAF even though we have a rule to block traffic that does not meet our rule criteria!

The workaround
-----

In order to reach my goal, I had to change my approach a bit. Once a custom rule in a WAF is matched, the corresponding action defined in the rule is applied to the request. Once such a match is processed, rules with lower priorities are not processed further. This logic means that if I create one allow rule with a high priority to allow traffic with the request header `Content-Type: application/secret-request` and one block rule that in some way is catching all other traffic with a lower priority it should work better.  

Here are the new rules I created:

{% highlight powershell %}
$variable1 = New-AzApplicationGatewayFirewallMatchVariable `
   -VariableName RequestHeaders `
   -Selector Content-Type

$variable2 = New-AzApplicationGatewayFirewallMatchVariable `
   -VariableName RemoteAddr
 
$condition1 = New-AzApplicationGatewayFirewallCondition `
   -MatchVariable $variable1 `
   -Operator Equal `
   -MatchValue "application/secret-request" `
   -Transform Lowercase `
   -NegationCondition $false

$condition2 = New-AzApplicationGatewayFirewallCondition `
   -MatchVariable $variable2 `
   -Operator IPMatch `
   -MatchValue "127.0.0.1" `
   -NegationCondition $true

$rule1 = New-AzApplicationGatewayFirewallCustomRule `
   -Name AllowSecretRequests `
   -Priority 10 `
   -RuleType MatchRule `
   -MatchCondition $condition1 `
   -Action Allow
   
$rule2 = New-AzApplicationGatewayFirewallCustomRule `
   -Name DenyAllTraffic `
   -Priority 20 `
   -RuleType MatchRule `
   -MatchCondition $condition2 `
   -Action Block   
{% endhighlight %}


The best way I could come up with to block all other traffic was to create a deny rule for traffic that is not originating from IP address 127.0.0.1. In an externally facing Application Gateway the source IP will never be 127.0.0.1 which makes it a valid way to deny traffic in my opinion. 

Let’s test again by sending a request with no `Content-Type:` header again.

<img src="https://github.com/StefanIvemo/stefanivemo.github.io/blob/master/images/waf-gotcha/4CurlNoHeaderBlocked.PNG?raw=true">

This time the traffic is being blocked by the WAF! And my valid traffic is still allowed as well.

<img src="https://github.com/StefanIvemo/stefanivemo.github.io/blob/master/images/waf-gotcha/1CurlCorrectHeader.PNG?raw=true">


Summary
-----
That's it! In my world my first approach should have been enough to control the inbound traffic. No header should be treated the same ways as a non matching header. Anyway I was able to create a workaround, hope it will help you someday!



<script src="https://utteranc.es/client.js"
        repo="StefanIvemo/stefanivemo.github.io"
        issue-term="pathname"
        label="Comment"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>

<script async defer src="https://buttons.github.io/buttons.js"></script>