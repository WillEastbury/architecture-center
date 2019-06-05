---
title: Asynchronous Request-Reply Pattern
description: Allow decoupling of backend processing from a frontend host, where backend processing needs to be asynchronous, but the frontend still needs a clear response.
keywords: async, http, https, request, reply, synchronous, asynchronous, queue, cqrs, valet, extreme load, transactional, retry-after, location, 202, accepted
ms.date: 04/06/2019
pnp.series.title: Cloud Design Patterns
pnp.pattern.categories: [design-implementation, messaging, performance-scalability]
ms.author: wieastbu
ms.fasttrack: new
---

# Asynchronous Request-Reply Operations

[!INCLUDE [header](../_includes/header.md)]

## Context and Problem

In modern application development it is very usual for client applications - often code running in a web-client (browser) - to depend on remote APIs to provide business logic and compose functionality. These APIs may be directly related to the application, or they may be shared services provided by a third party. 
It is popular for client => API calls to take place over HTTP Application protocol and for these calls to follow REST and JSON semantics.

In many cases dependent APIs for a Client application will be engineered to respond very quickly (in the order of low tens of millisecond, not seconds). 

Whilst the hosting stack, security components, the relative geographic location of caller and API, and network infrastructure can all add additional latency; the time for a client to solicit the response from a dependent API is usually short enough that requests can be made and the response returned efficiently and synchronously, on the same connection. 

Note: Client application code will often make a synchronous API call in a non-blocking way, giving the appearance of asynchronous processing, which is generally recommended for IO bound, or blocking, operations.

However, in some scenarios the work being undertaken by the API may be considered “long running” (order of minutes) which becomes problematic for traditional synchronous solicit response actions over HTTP; or, the application architecture requires that request and response phases be separated, often through use of a queue or message broker, to allow the client process to scale and prevent backend APIs from being overwhelmed.

Many of the same considerations discussed for Client (browser) applications also apply for server to server REST API calls.


## Solution

There is no one size fits all solution or pattern for implementing asynchronous background processing and splitting request and response phases. 
We will discuss one pattern, client pull, utilizing HTTP polling; polling is especially useful to client browser application scenarios where it is often very difficult to provide call-back endpoints and where use of long running connections and the libraries required to facilitate this robustly can add unwanted additional complexity. 
- The client application will send a synchronous request to the API performing a long running operation, this can be considered an action which will instruct the API to begin. 
- The API will respond as quickly as possible, synchronously with an acknowledgement. 
- The response will also contain a location for which the client application can poll to receive the end result of the long running operation. 
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
- If an error occurs, you should write the error away to the location described in the location header.
- Not all solutions implement this in the same way and some services will include additional or alternate headers, for example Azure Resource Manager uses a modified version of this pattern.
[See Azure Resource Manager Async Operations](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-async-operations)
-	This pattern does not allow for the long running operation to be cancelled.
-	This pattern is less suitable if responses need to be streamed back, or many results are to be collected and latency is important. In these scenarios a server push pattern may be more applicable


## When to use this pattern

_Deliver specific guidance on when this pattern should and should not be used. Use the following boilerplate text, followed by bullet points, to help the reader determine if the solution is applicable to their specific scenario._

Use this pattern when:

- Item 1
- Item 2
- Item 3

This pattern might not be suitable:

- Item 1
- Item 2
- Item 3

## Example

_Include a working sample that shows the reader how the pattern solution is used in a real-world situation. The sample should be specific and provide code snippets when appropriate._

## Next steps

_Provide links to other topics that provide additional information about the pattern covered in the article. Topics can include links to pages that provide additional context for the pattern discussed in the article or links to pages that may be useful in a next-steps context. Use the following boilerplate sentence followed by a bulleted list._

The following information may be relevant when implementing this pattern:

- Item 1
- Item 2
- Item 3
