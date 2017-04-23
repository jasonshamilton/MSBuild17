title: Managing Concurrent Connection Limits for Reverse Proxy | Microsoft Docs
description: Architecting against chatty function calls in Reverse Proxy
services: FabricApplicationGateway
documentationcenter: dev-center-name
author: GitHub-alias-of-only-one-author
manager: manager-alias
This is a test


ms.service: required
ms.devlang: may be required
ms.topic: article
ms.tgt_pltfrm: may be required
ms.workload: required
ms.date: 03/05/2017
ms.author: Your MSFT alias or your full email address;semicolon separates two or more aliases

---

#Introduction 
This article is about managing the concurrent connection limits for FabricApplicationGateway reverse proxy. It discusses and addresses how to configure and manages to eliminate exceptions from causing your application to denial of service (DDOS) itself in a self healing cluster such as Azure Service Fabric.

##Sceanrio
AppFoo has an excessive function call (for a few possible reasons including overly chatty code, aggressive retry mechanisms onsome error/exception, etc...).  This is causing a saturation/flooding scenario on the connection limit.

##Investigation Points
Issues to investigate and consider:

- Is the connection limit running out of sockets?  There is a default 16k winsocket port
- Is the default conncurrent connection limit being exceeded which is managed by service fabric reverse proxy

For the focus of this article, the assumption is sockets have been investigated and there are no issues.

##Service Fabric Concurrent Connection Limits

- Currently Reverse proxy handles 1000 concurrent requests by default  
- Beyond this limit, requests will be pushed into the http.sys queue
- This queue can affect response time from the target service 
- This increased response time will affect the topology chain as well within the Reverse Proxy.  In other words if you are using more than one service in your chain (e.g. AppCentral Tollbridge etc..), the delay will permeate across these requests.

##ARM Template for Increasing Reverse Proxy Defaults
In a scenario where there is need to increase the reverse proxy concurrent request defaults there are a few methods/approaches

###Using the Resource Mgr to Modify the Service Fabric Cluster

- Go to https://resources.azure.com 
- Navigate down to your subscriptions -- >  resourceGroups -- > providers -- > Microsoft.ServiceFabirc --- > your cluster
 
 
- Click "Read/Write" then "Edit"
 
1.	Scroll down to “fabricSettings” part, so you can add or modify.
 

Note: Please test the suggestion on the testing environment first before use it to any production cluster.



Also, We are able to find a way to increase 1000 concurrent requests to a bigger value in our reverse proxy layer today by involving the reverse proxy code component developer, basically, you will have to push a change through ARM template on fabricSettings for using a bigger ApplicationGateway/Http/NumberOfParallelOperations value.

The ARM template code snippet is as following.
"fabricSettings": [

      {
        "name": " ApplicationGateway/Http",
        "parameters": [
          {
            "name": "NumberOfParallelOperations",
            "value": "10000"
          }
        ]
      }
],

•	You can set the above setting by using an incremental ARM template deployment. 
•	Or you can use the https://resources.azure.com to modify the Service Fabric cluster under the “fabricSettings” part.

If ARM template is not possible for you, you can also push the upgrade change from https://resources.azure.com 
1.	Go to https://resources.azure.com
2.	

I hope the information helps.

Thanks

-hz

From: Hilke, William [mailto:Bill.Hilke@wolterskluwer.com] 
Sent: Wednesday, March 1, 2017 4:39 PM
To: Hui Zhu <huizhu@microsoft.com>; Mercer, Casey <Casey.Mercer@wolterskluwer.com>; Todd Foust <Todd.Foust@microsoft.com>; Haque, MD <MD.Haque@wolterskluwer.com>
Cc: MSSolve Case Email <casemail@microsoft.com>; Hamilton, Jason <Jason.Hamilton@wolterskluwer.com>; Keith Mayer <kmayer@microsoft.com>; Taylor, Sean <Sean.Taylor@wolterskluwer.com>
Subject: RE: [REG:117030115391423] Bill.Hilke@wolterskluwer.com Initial Response

All great info Hui, thanks.

We’ll look into tweak connection limits while dev team looks into how they are dosing ourselves.



FYI guys, we were able to resolve the slow reverse proxy issue but we need to know why this was happening.

Basically, for the past week or so we had a bad application in the cluster which was crashing in RunAsync, moving to secondary replicata, crashing, repeating, repeat, etc.

For whatever reason this app in a constant crash/recover loop was causing the reverse proxy to take 30-50 seconds to respond to requests for a different app in the cluster.

As soon as we deleted the bad app, waited a minute for anything to clear out, the reverse proxy started responding immediately.

How do we get to the root cause of the correlation here?

I suspect that:
Even we are talking about a totally different app. App1 is crashing and ReverseProxy is talking to App2. For the problem symptom you listed, the delay may happen at the either Service Fabric Naming service (report the correct endpoint to reverse proxy FabricApplicationGateway.exe) or HealthManager service level. With a bad app in crash loop (app1), you may have extra /busy network payload at the intra node communications (need network trace to confirm) when the Service Fabric HealthManager to keep pulling the bad app status, which may slow down the whole network. If you can still repro the issue, we will need to take the network trace at your cluster node to check the TCP communication.

The major concern is the following HttpApplicationGateway connection issue over and over again when the bad app is in place.

2017-2-28 23:19:43.384	HttpApplicationGateway.RequestHandler.SendResponseToClient	15404	460	b155070e-e6c9-4b86-b72f-15c12294f431 Sending body during reply failed with 0xd0000001
2017-2-28 23:19:43.584	HttpApplicationGateway.RequestHandler.SendResponseToClient	15404	460	6a7b88c1-4e47-4cf6-9459-d26453a31d69 Sending body during reply failed with 0xd0000001
2017-2-28 23:19:43.584	HttpApplicationGateway.RequestHandler	15404	460	6a7b88c1-4e47-4cf6-9459-d26453a31d69 SendResponse to client failed with 0xd0000001, winHttpError 0
2017-2-28 23:19:43.586	HttpApplicationGateway.RequestHandler.SendRequestToService	15404	460	8596b3ce-8d14-4081-8afc-8a912f39c947 SendRequestChunk failed with 0xd00000e5 WinHttpError - 12030

Basically, the HttpApplicationGateway to service failed with 12030, which means ERROR_WINHTTP_CONNECTION_ERROR. This error indicate some connection retry timeout. So, the endpoint seems to be messed up in the naming service when the bad app in place.

Question: how fabric:/TollbridgeApp/Tollbridge/v1.0/api relates to your fabric:/AppCentralApp application and bad fabric:/bpp app?


Ok. We saw the thousands of reverse proxy call over and over again to the following locations: 
http://10.234.151.133:8080/Tollbridge/v1.0/api/config/Business%20Process%20Platform:default@audit_queue_publish 
http://10.234.151.133:8080/Tollbridge/v1.0/api/config/message_platform:default@queue_publisher

Most of them are ending up with follow 2 type of errors:
either
2017-2-28 22:57:00.412	HttpApplicationGateway.RequestHandler.SendRequestToService	15404	46320	ea51752e-a195-4b03-833d-351f8d81d0d3 SendRequestChunk failed with 0xd00000e5 WinHttpError - 12002

Or
2017-2-28 22:57:00.412	HttpApplicationGateway.RequestHandler.SendRequestToService	15404	460	11e10ce3-130f-4bcc-b703-c32e211e7e5b SendRequestChunk failed with 0xd00000e5 WinHttpError - 12030

12002 means ERROR_WINHTTP_TIMEOUT
12030 means ERROR_WINHTTP_CONNECTION_ERROR

So, it looks like when in bpp retry mechanism it was creating a DOS (denial of service) traffic for sockets to the HttpApplicationGateway, and eventually make HttpApplicatonGateway not have ephemeral port/socket to do the normal request connection in timely fashion. The issue is not network traffic jam, but the HttpApplicationGateway process seemed to be running out of connection and ephemeral ports.

Just a simple “netstat -ano” output during the repro might give you the full details of the root cause.


The connection limit is not at the VM size level, such as D13 and D15. It is setting at winsock limit of OS level. 
You can query it by using “netsh int ipv4 show dynamicport tcp” from admin dos prompt. All sockets combined together by default has a limit 16k for windows 2012 R2, but you can increase the socket limit number by using netsh int <ipv4|ipv6> set to tweak it more than 16k. Also, Any a closed connection to be qualified for reuse has to be wait up to 240 seconds time elapse. https://technet.microsoft.com/en-us/library/cc938217.aspx . The minimum you can decrease this wait is to set 30 seconds in that registry of above link.

ReverseProxy is implement by using WinHttp library. The following winhttp errors can be either 1) no winsock resource on the client or 2) or the server side service is not accepting the connection. That is the reason why I mentioned a winhttp API trace or a netmon trace can help to clear out the story for us.

Error:
12002 means ERROR_WINHTTP_TIMEOUT
12030 means ERROR_WINHTTP_CONNECTION_ERROR

Definitely, those thousand of 12002 and 1203 error in reservice proxy HttpApplicationGateway is the root cause of symptom you saw. But without netstat -ano, I cannot pull the real time data for how many sockets has already been used at a given moment of yesterday.

Please let me know if you need to talk with me on the phone.

Thx.

-hz



title: Page title that displays in the browser tab and search results | Microsoft Docs
description: Article description that will be displayed on landing pages and in most search results
services: service-name
documentationcenter: dev-center-name
author: GitHub-alias-of-only-one-author
manager: manager-alias


ms.service: required
ms.devlang: may be required
ms.topic: article
ms.tgt_pltfrm: may be required
ms.workload: required
ms.date: mm/dd/yyyy
ms.author: Your MSFT alias or your full email address;semicolon separates two or more aliases

---
# Markdown template for Azure on Microsoft Docs

Your article should have only one H1 heading, which you create with a single # sign. The the H1 heading should always be followed by a descriptive paragraph that helps the customer understand what the article is about. It should contain keywords you think customers would use to search for this piece of content. Do not start the article with a note or tip - always start with an introductory paragraph.

## Headings 

Two ## signs create an H2 heading - if your article needs to be structured with headings below the H1, you need to have at least TWO H2 headings.

H2 headings are rendered on the page as an automatic on-page TOC. Do not hand-code article navigation in an article. Use the H2 headings to do that.

Within an H2 section, you can use three ### signs to create H3 headings. In our content, try to avoid going deeper than 3 heading layers - the headings are often hard to distinguish on the rendered page. 

## Images
You can use images throughout a technical article. Make sure you include alt text for all your images. This helps accessibility and discoverability.

This image of the GitHub Octocats uses in-line image references:

 ![GitHub Octocats using inline link](./media/markdown-template-for-new-articles/octocats.png)

The sample markdown looks like this:
```
![GitHub Octocats using inline link](./media/markdown-template-for-new-articles/octocats.png)
```

This second image of the octocats uses reference style syntax, where you define the target as "5" and at the bottom of the article, and you list the path to image 5 in a reference section.

 ![GitHub Octocats using ref style link][5]

 The sample markdown looks like this:
 ```
  ![GitHub Octocats image][5]

  <!--Image references-->
  [5]: ./media/markdown-template-for-new-articles/octocats.png
 ``` 

## Linking
Your article will most likely contain links. Here's sample markdown for a link to a target that is not on the docs.microsoft.com site:

    [link text](url)
    [Scott Guthrie's blog](http://weblogs.asp.net/scottgu)

Here's sample markdown for a link to another technical article in the azure-docs-pr repository:

    [link text](../service-directory/article-name.md)
    [ExpressRoute circuits and routing domains](../expressroute/expressroute-circuit-peerings.md)

You can also use so-called reference style links where you define the links at the bottom of the article, and reference them like this:

    I get 10 times more traffic from [Google][gog] than from [Yahoo][yah] or [MSN][msn].

For more information about linking, see the [linking guidance](../contributor-guide/create-links-markdown.md)

## Notes and tips
You should use notes and tips judiciously. A little bit goes a long way. Put the text of the note or tip on the line after the custom markdown extension.

```
> [!NOTE]
> Note text.

> [!TIP]
> Tip text.

> [!IMPORTANT]
> Important text.
```

## Lists

A simple numbered list in markdown creates a numbered list on your published page.

1. First step.
2. Second step.
3. Third step.

Use hyphens to create unordered lists:

- Item
- Item
- Item


## Next steps
Every topic should end with 1 to 3 concrete, action oriented next steps and links to the next logical piece of content to keep the customer engaged. 

- See the [content quality guidelines](../contributor-guide/contributor-guide-pr-criteria.md#non-blocking-content-quality-items) for an example of what a good next steps section looks like. 

- Review the [custom markdown extensions](../contributor-guide/custom-markdown-extensions.md) we use for videos, reusable content, selectors, and other content features.

- Make sure your articles meet [the content quality guidelines](../contributor-guide/contributor-guide-pr-criteria.md) before you sign-off on a PR. 


<!--Image references-->
[5]: ./media/markdown-template-for-new-articles/octocats.png

<!--Reference style links - using these makes the source content way more readable than using inline links-->
[gog]: http://google.com/        
[yah]: http://search.yahoo.com/  
[msn]: http://search.msn.com/    