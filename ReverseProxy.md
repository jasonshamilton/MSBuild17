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

## Scenario

AppFoo has an excessive function call (for a few possible reasons including overly chatty code, aggressive retry mechanisms on some error/exception, etc...).  This is causing a saturation/flooding scenario on the connection limit.

## Investigation Points

Issues to investigate and consider:

- Is the connection limit running out of sockets? 
- Is the default concurrent connection limit being exceeded which is managed by service fabric reverse proxy?

For the focus of this article, the assumption is sockets have been investigated and there are no issues.  See the Tools and Tips section to validate you are not running into socket issues.

## Service Fabric Concurrent Connection Limits

- Currently Reverse proxy handles 1000 concurrent requests by default
- Beyond this limit, requests will be pushed into the http.sys queue
- This queue can affect response time from the target service
- This increased response time will affect the topology chain as well within the Reverse Proxy.  In other words if you are using more than one service in your chain, the delay will permeate across these requests

## ARM Template for Increasing Reverse Proxy Defaults

Note: Please test the suggestion on the testing environment first before use it to any production cluster

In a scenario where there is need to increase the reverse proxy concurrent request defaults there are a few methods/approaches

Users can push a change through ARM template on fabricSettings for using a bigger ApplicationGateway/Http/NumberOfParallelOperations value.

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

### Using the Resource Mgr to Modify the Service Fabric Cluster

Note: Please test the suggestion on the testing environment first before use it to any production cluster

Making the changes through the Management Admin Portal

- Go to https://resources.azure.com 
- Navigate down to your subscriptions -- >  resourceGroups -- > providers -- > Microsoft.ServiceFabirc --- > your cluster
- Click "Read/Write" then "Edit"
- Scroll down to “fabricSettings” part, so you can add or modify


### Tools and Tips for Troubleshooting

- Use “netstat -ano” output during the repro might give you the full details of the root cause
- Note: Connection limits are not at the VM size level, such as D13 and D15. Winsock limit are at the OS level.
- You can query it by using “netsh int ipv4 show dynamicport tcp” from admin dos prompt
- To increase socket limits numbers, use netsh int <ipv4|ipv6> 
- Note: Any closed connection to be qualified for reuse has to be wait up to 240 seconds time elapse
  - See: https://technet.microsoft.com/en-us/library/cc938217.aspx
  - The minimum you can decrease this wait is to set 30 seconds in that registry of above link



