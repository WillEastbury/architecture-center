---
title: Asynchronous Request-Reply Pattern
description: Allow decoupling of backend processing from a frontend host, where backend processing needs to be asynchronous, but the frontend still needs a clear response.
keywords: async, http, https, request, reply, synchronous, asynchronous, queue, cqrs, valet, extreme load, transactional, retry-after, location, 202, accepted
ms.date: 06/06/2019
pnp.series.title: Cloud Design Patterns
pnp.pattern.categories: [design-implementation, messaging, performance-scalability]
ms.author: wieastbu, begim
ms.custom: fasttrack-new
---

# Asynchronous Request-Reply Operations

[!INCLUDE [header](../_includes/header.md)]

## Context and Problem

In modern application development it is very usual for client applications - often code running in a web-client (browser) - to depend on remote APIs to provide business logic and compose functionality. These APIs may be directly related to the application, or they may be shared services provided by a third party.   
It is popular for client => API calls to take place over HTTP Application protocol and for these calls to follow REST and JSON semantics.  
  
In many cases dependent APIs for a Client application will be engineered to respond very quickly (in the order of low tens of milliseconds, not seconds).  

Whilst the hosting stack, security components, the relative geographic location of caller and API, and network infrastructure can all add additional latency; the time for a client to solicit the response from a dependent API is usually short enough that requests can be made and the response returned efficiently and synchronously, on the same connection.  

Note: Client application code will often make a synchronous API call in a non-blocking way, giving the appearance of asynchronous processing, which is generally recommended for I/O bound (or blocking) operations.  

However, in some scenarios the work being undertaken by the API may be considered “long running” (in the order of minutes of elapsed time) which becomes problematic for traditional synchronous solicit response actions over HTTP; or, the application architecture requires that request and response phases be separated, often through use of a queue or message broker (see the [Queue Based Load Leveling Pattern](./queue-based-load-leveling.md)), to allow the client process to scale and prevent backend APIs from being overwhelmed.  

Many of the same considerations discussed for Client (browser) applications also apply for server to server REST API calls in distributed systems.

## Solution

There is no one size fits all solution or pattern for implementing asynchronous background processing and splitting request and response phases.   
We will discuss one pattern, client pull, utilizing HTTP polling; polling is especially useful to client browser application scenarios where it is often very difficult to provide call-back endpoints and where use of long running connections and the libraries required to facilitate this robustly can add unwanted additional complexity.   
- The client application will send a synchronous request to the API performing a long running operation, this can be considered an action which will instruct the API to begin.  
- The API will respond as quickly as possible, synchronously with an acknowledgement that the message has been received for processing. 
- The response will also contain a location reference, specifying a location that the client application can poll to receive the end result of the long running operation.  
- Optionally, the response may also contain additional data (in the form of a response message header) – for example if the API knows that a given operation will average a given amount of time (e.g. 5minutes) it will be wasteful for the client to poll from the moment its acknowledged. Instead an “poll-after” property can be returned in the response with a duration after which the client should start polling.  
- Additional data might be a suggestion by the API for the callers polling interval. At most the polling interval duration will contribute to the overall duration between request and result being collected. But, over a long running operation the longer polling duration is much less wasteful of client and server resources.  

Note: Best practice is for the API to validate both the request and the action asked from a business rules perspective before starting the long running process and indicate the success on the synchronous response to the caller.

## Issues and considerations

Consider the following points when deciding how to implement this pattern:
- A 202-Accepted http response status is a sensible choice to indicate that the request has been accepted for processing, but that processing is not complete.  
- There a number of possible ways to implement this pattern over http and not all upstream services have the same semantics, i.e. not all services that you might choose to host the response payload on are capable of returning a 202 Accepted response in response to a GET method call, most services will correctly return a 404 not found in this case.  
- To indicate the location and frequency that your client should poll for the response, your 202-Accepted reply should have additional headers, described below.  
- A location header should be set to point to the http location to poll for, a SAS token can be used with the Valet Key Pattern, if this needs to be secured.  
- You may need to use a processing proxy or facade to manipulate the response headers and / or payload depending on the underlying services used. We will demonstrate this below.  
- A retry-after header should be set to an estimate of when the request procesing is going to complete to prevent eager polling clients from overwhelming the back end target system with retries.  
- If an error occurs, you should write the error away to the resource described in the location header and ideally return an appropriate response code to the client from that resource (4xx code).  
- Upon successful processing, the resource specified by the location header should return an appropriate http response code (200 OK, 201 Created, or 204 No Content).  
- Not all solutions implement this in the same way and some services will include additional or alternate headers, for example Azure Resource Manager uses a modified version of this pattern.  
[See Azure Resource Manager Async Operations](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-async-operations)  

## When to use this pattern

Use this pattern when:

- Client to Service Async Polling is especially useful to client browser application scenarios where it is often very difficult to provide call-back endpoints and where use of long running connections and the libraries required to facilitate this robustly can add unwanted additional complexity.   
- Client calls that are made asynchronously to a REST API that need a method to ascertain the status of a long-running request.
- Service to Service calls that need to be made across restrictive network boundaries, where only the https protocol is available, and the service cannot call back through a firewall due to NAT / Firewall restrictions.  
- Service to Service calls that need to be integrated with Legacy-type architectures that do not support modern callback technologies like Websockets, SignalR or Event Grid / WebHooks.  

This pattern might not be suitable:

-	This pattern is less suitable if responses need to be streamed back, or many results are to be collected and latency is important. In these scenarios a server push pattern may be more applicable. If you have access to server-side persistent network connections, such as websockets and / or SignalR to notify that the service has completed processing.  
-	This particular implementation of this pattern does not allow for the long running operation to be cancelled, in order to allow the processing of cancel messages the back end service must support some form of cancellation instruction which is out of the scope of this document but could be within the scope of the service.  
- As a substitute for pure event driven callbacks or WebHooks, use a service that is designed for asynchronous event notifications, such as Azure Event Grid.  

## Example

Let's suppose we have a Function App with 2 http bound endpoints exposed and an Azure Storage Queue bound function to do the work :-

### 1. Http POST -> AsyncProcessingWorkAcceptor
> This needs to accept the work, put it into an envelope wrapper with some metadata then pop it onto the queue for processing and generate the SAS signature and 202 response back to the client, the location should point at the AsyncOperationStatusChecker Endpoint.

### Queue AsyncProcessingBackgroundWorker
> This should pick the operation up off the queue

### Http GET AsyncOperationStatusChecker
> This needs to check if the response is complete, and if so return a valet-key to the actual response

## Next steps

The following information may be relevant when implementing this pattern:

- [Azure Logic Apps - Perform Long Running Tasks] (https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-create-api-app#perform-long-running-tasks-with-the-polling-action-pattern)
- [Sample Code - ASP.NET Implementation] (https://github.com/jeffhollan/LogicAppsAsyncResponseSample)
