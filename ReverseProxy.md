title: Managing Concurrent Connection Limits for Reverse Proxy | Microsoft Docs
description: Architecting against chatty function calls in Reverse Proxy
services: FabricApplicationGateway
documentationcenter: dev-center-name
author: GitHub-alias-of-only-one-author
manager: manager-alias



ms.service: required
ms.devlang: may be required
ms.topic: article
ms.tgt_pltfrm: may be required
ms.workload: required
ms.date: 04/23/2017
ms.author: jason.hamilton@wolterskluwer.com;semicolon separates two or more aliases
This is a Test

---

# Introduction 
This article is about managing the concurrent connection limits for FabricApplicationGateway reverse proxy. It discusses and addresses how to configure, manage, and eliminate exceptions from causing your application to denial of service (DDOS) itself in a self healing cluster such as Azure Service Fabric.

## Sceanrio
AppFoo has an excessive function call (for a few possible reasons including overly chatty code, aggressive retry mechanisms on some error/exception, etc...).  This is causing a saturation/flooding scenario on the connection limit.

## Investigation Points
Issues to investigate and consider:

- Is the connection limit running out of sockets?  There is a default 16k winsocket port
- Is the default conncurrent connection limit being exceeded which is managed by service fabric reverse proxy

For the focus of this article, the assumption is sockets have been investigated and there are no issues.

## Service Fabric Concurrent Connection Limits

- Currently Reverse proxy handles 1000 concurrent requests by default
- Beyond this limit, requests will be pushed into the http.sys queue
- This queue can affect response time from the target service
- This increased response time will affect the topology chain as well within the Reverse Proxy.  In other words if you are using more than one service in your chain (e.g. AppCentral Tollbridge etc..), the delay will permeate across these requests.

## ARM Template for Increasing Reverse Proxy Defaults
In a scenario where there is need to increase the reverse proxy concurrent request defaults there are a few methods/approaches

### Using the Resource Mgr to Modify the Service Fabric Cluster

- Go to https://resources.azure.com 
- Navigate down to your subscriptions -- >  resourceGroups -- > providers -- > Microsoft.ServiceFabirc --- > your cluster
 
 
- Click "Read/Write" then "Edit"
 
1.	Scroll down to “fabricSettings” part, so you can add or modify.
 

Note: Please test the suggestion on the testing environment first before use it to any production cluster.



Also, We are able to find a way to increase 1000 concurrent requests to a bigger value in our reverse proxy layer today by involving the reverse proxy code component developer, basically, you will have to push a change through ARM template on fabricSettings for using a bigger ApplicationGateway/Http/NumberOfParallelOperations value.

The ARM template code snippet is as following.

```c#

"fabricSettings": 

[

      {
        "name": " ApplicationGateway/Http",
        "parameters": [
          {
            "name": "NumberOfParallelOperations",
            "value": "10000"
          }
        ]
      }
]

```

•	You can set the above setting by using an incremental ARM template deployment. 
•	Or you can use the https://resources.azure.com to modify the Service Fabric cluster under the “fabricSettings” part.

If ARM template is not possible for you, you can also push the upgrade change from https://resources.azure.com 
1.	Go to https://resources.azure.com
2.	

I hope the information helps.

Thanks

-hz

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


