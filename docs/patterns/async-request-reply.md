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

_Allow decoupling of backend processing from a frontend host, whilst making the backend processing asynchronous. 
This pattern allows frontend applications to continue processing other tasks whilst maintaining the illusion of synchronous processing and also protecting the service from being overloaded due to extreme polling demands.
It is also useful in situations where you wish to break apart front and back end services (such as the introduction of a processing queue in between the front and backend hosts) whilst maintaining a level of pseudo-synchronicity from the client frontend._

## Context and problem

It is often the case that you need to break apart the request and reply phases of an http(s) request for a longer running operation, but maintain the traffic flow over vanilla http(s). The actual processing of a long running request may be transmitted through the back end via a queue or other async messaging technology, but you need to express this delay in processing and provide the client with a reliable method to check for and obtain the result of the call over standard http semantics. 

## Solution

Respond to the caller with a http response payload representing the current state in time of the request, and a pointer to where the result will be when it is completed and an estimate of time till completion. Then the client can poll the pointer location at the interval suggested by the estimate to see if the request is completed and obtain the appropriate result when it is complete without a synchronous call blocking both the caller and called API. 

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
