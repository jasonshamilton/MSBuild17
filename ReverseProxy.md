#Introduction 
This article is about managing the concurrent connection limits for FabricApplicationGateway reverse proxy. It discusses and addresses how to configure and manages to elimiate exceptions from causing your application to denial of service (DDOS) itself in a self healing cluster such as Azure Service Fabric.

#Sceanrio
AppFoo has an excessive function call (for a few possible reasons including overly chatty code, aggressive retry mechanisms onsome error/exception, etc...).  This is causing a saturation/flooding scenario on the connection limit.

#Investigation Points





Hi William,

Good afternoon! Just hope to follow up with you to see if you need any additional info on this reverse proxy performance issue.

With additional debugging, we can confirm when fabric:/bpp was crashing, there are thousands of reverse proxy connection requests were asked to the following Urls over and over again:
http://10.234.151.133:8080/Tollbridge/v1.0/api/config/messaging_publish_service:default@messaging_eventhub
http://10.234.151.133:8080/Tollbridge/v1.0/api/config/Business%20Process%20Platform:default@audit_queue_publish
http://10.234.151.133:8080/Tollbridge/v1.0/api/config/Business%20Process%20Platform:default@endpointSettings
http://10.234.151.133:8080/Tollbridge/v1.0/api/config/Business%20Process%20Platform:default@relaySettings
http://10.234.151.133:8080/Tollbridge/v1.0/api/config/message_platform:default@queue_publisher
…

We were guessing the high connection numbers were coming from fabric:/bpp and somehow they were associated with this function WK.Axcess.Tollbridge.TollBridgeWebClient.GetConfig[T](BridgePath path)
In bpp app. It can be  either a retry from the above code path on some error/exception or from the nature of that function asking for more Urls than other functions.

Eventually, the above flooding requests had saturated the concurrent connection limit. We were able to confirm the connection limit is not that we were running out of sockets. The default 16k winsock port still seems to be available.
After digging into the FabricApplicationGateway reverse proxy code and implementation, we are able to confirm another bottleneck @ the default concurrent connection limit, which is enforced by service fabric reverse proxy itself.

Basically, Reverse proxy handles 1000 concurrent requests by default and additional requests beyond that will be pushing and sitting in the http.sys queue. The time taken by the target service to process a forwarded request would affect the time a request spends in the http.sys queue. Since the same reverse proxy is handling requests for both the TollBridge service from bpp and the AppCentralApp, the flooding requests to TollBridge is also delaying the requests for AppCentralApp as well

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
2.	Navigate down to your subscriptions -- >  resourceGroups -- > providers -- > Microsoft.ServiceFabirc --- > your cluster
 
 
3.	Click "Read/Write" then "Edit"
 
1.	Scroll down to “fabricSettings” part, so you can add or modify.
 

Note: Please test the suggestion on the testing environment first before use it to any production cluster.

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



#Getting Started
The documentation will follow the Azure Doc best practices across the differnt types of articles
1.	[Azure Technical Documentation Contributor Guide](https://github.com/Microsoft/azure-docs/blob/master/README.md)
2.	[Markdown Cheatsheet](https://github.com/Microsoft/azure-docs/blob/master/README.md)

If you want to learn more about creating good readme files then refer the following [guidelines](https://www.visualstudio.com/en-us/docs/git/create-a-readme). You can also seek inspiration from the below readme files:
- [ASP.NET Core](https://github.com/aspnet/Home)
- [Visual Studio Code](https://github.com/Microsoft/vscode)
- [Chakra Core](https://github.com/Microsoft/ChakraCore)