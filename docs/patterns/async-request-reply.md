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

## Examples

![Image of the structure of an ASync Init logic app](/docs/patterns/_images/arr-Async-Request-Init.png)

# [In Azure Functions](#tab/azfunc)

Let's suppose we have a Function App with 2 http bound endpoints exposed and an Azure Storage Queue bound function to do the work :-



## Http POST -> AsyncProcessingWorkAcceptor
> This function needs to accept the work, put it into an envelope wrapper with some metadata, then pop it onto the queue for processing and generate the SAS signature and 202 response back to the client, the location returned should point at the location of the  AsyncOperationStatusChecker Endpoint.

````csharp
#r "Newtonsoft.json" 
#r "Microsoft.WindowsAzure.Storage"
 
using System;
using System.Net;
using Microsoft.WindowsAzure.Storage; 
using Microsoft.WindowsAzure.Storage.Blob;
using System.Collections.Generic;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Primitives;
using Newtonsoft.Json; 
using Newtonsoft.Json.Linq; 

// Accept the Request and pop the message onto the queue
public static async Task<IActionResult> Run(HttpRequest req, ILogger log, CloudBlobContainer inputBlob, IAsyncCollector<string> OutMessage)
{
    
    string reqid = Guid.NewGuid().ToString();
    
    string rqs = $"https://{Environment.GetEnvironmentVariable("WEBSITE_HOSTNAME")}/api/RequestStatus/{reqid}";

    string q = EnvelopeJSONBody(
        await new StreamReader(req.Body).ReadToEndAsync(), 
        new Dictionary<string, JToken>() {
                { "RequestGUID", reqid },
                { "RequestSubmittedAt", DateTime.Now },
                { "RequestStatusURL", rqs}
            }
    );
    
    await OutMessage.AddAsync(q);  

    CloudBlockBlob cbb = inputBlob.GetBlockBlobReference($"{reqid}.blobdata");

    return (ActionResult) new AcceptedResult(rqs, $"Request Accepted for Processing{Environment.NewLine}ValetKey: {GenerateSASURIForBlob(cbb)}{Environment.NewLine}ProxyStatus: {rqs}");  
    
}

public static string GenerateSASURIForBlob(CloudBlockBlob blob)
{

    SharedAccessBlobPolicy sasConstraints = new SharedAccessBlobPolicy();
    sasConstraints.SharedAccessStartTime = DateTimeOffset.UtcNow.AddMinutes(-5);
    sasConstraints.SharedAccessExpiryTime = DateTimeOffset.UtcNow.AddMinutes(10);
    sasConstraints.Permissions = SharedAccessBlobPermissions.Read;

    //Generate the shared access signature on the blob, setting the constraints directly on the signature.
    string sasBlobToken = blob.GetSharedAccessSignature(sasConstraints);

    //Return the URI string for the container, including the SAS token.
    return blob.Uri + sasBlobToken;

}

// takes the input and adds an additional property to the JSON body and returns a serialized string
public static string AppendGUIDToJSONBody(string inputjson, JToken propValue, string propID)
{
   
    JObject job = JObject.Parse(inputjson); 
    job.Add(propID, propValue);
    return job.ToString();

}

// https://en.wikipedia.org/wiki/Decorator_pattern#Overview
// takes the input and augments the object with an additional set of properties to the JSON body as am additional header level property (EnvelopeProperties)
// and returns a serialized string with the original object as the root
public static string DecorateJSONBody(string inputjson, Dictionary<string, JToken> HeaderProperties)
{

    JObject job = JObject.Parse(inputjson); 
    job.Add("EnvelopeProperties",JObject.FromObject(HeaderProperties)); 
    return job.ToString();

}

// https://www.enterpriseintegrationpatterns.com/patterns/messaging/EnvelopeWrapper.html
// Classic 'Envelope' , takes the input and augments the object with an additional set of properties to the JSON body as am additional header level property (EnvelopeProperties) 
// whilst appending the original object as a new property called 'requestobject' and returns a serialized string
public static string EnvelopeJSONBody(string inputjson, Dictionary<string, JToken> HeaderProperties)
{
    
    JObject job = new JObject();
    job.Add("RequestObject", JObject.Parse(inputjson));
    job.Add("EnvelopeProperties",JObject.FromObject(HeaderProperties)); 
    return job.ToString();

}

// To get to the original object, deserialize it with the correct class
public class CustomerPOCO
{
    public string id;
    public string customername;

}

// Or make it inherit and bring in the headers into the object
public class CustomerPOCOwithMessageHeaders : CustomerPOCO {

    public string RequestGUID {get;set;}
    public string RequestSubmittedAt {get;set;}
    public string RequestStatusURL {get;set;}

    // + Your other class POCO fields
    // public string id;
    // public string customername;
    
}

public class EnvelopedMessage {

    public EnvelopeProperties envelopeProperties {get;set;}
    public string RequestObject {get;set;}

}

// To get the headers deserialize it with decoratedmessage.
public class DecoratedMessage {

    public EnvelopeProperties envelopeProperties {get;set;}
    
}

// To get to the original object, deserialize it with the correct class, to get both, inherit your POCO base class from DecoratedMessage
public class CustomerPOCODecoratedMessage : DecoratedMessage {

    public string id;
    public string customername;

}

public class EnvelopeProperties {

    public string RequestGUID {get;set;}
    public string RequestSubmittedAt {get;set;}
    public string RequestStatusURL {get;set;}
}
````

````json
{
  "bindings": [
    {
      "authLevel": "anonymous",
      "name": "req",
      "type": "httpTrigger",
      "direction": "in",
      "methods": [
        "post"
      ]
    },
    {
      "name": "$return",
      "type": "http",
      "direction": "out"
    },
    {
      "type": "queue",
      "name": "OutMessage",
      "queueName": "outqueue",
      "connection": "AzureWebJobsStorage",
      "direction": "out"
    },
    {
      "type": "blob",
      "name": "inputBlob",
      "path": "data/blob.blob",
      "connection": "AzureWebJobsStorage",
      "direction": "in"
    }
  ]
}
````

## Queue AsyncProcessingBackgroundWorker

> This should pick the operation up off the queue. remove the payload from the envelope, do something meaningful with it, then write away the end result to the provided blob SAS signature.

````csharp
#r "Newtonsoft.json"
#r "Microsoft.WindowsAzure.Storage"
 
using System;
using Microsoft.WindowsAzure.Storage; 
using Microsoft.WindowsAzure.Storage.Blob;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;

// Perform some kind of background processing here and write the result to a blob created in the data folder with a guid as the blob name

public static void Run(EnvelopedMessage<ThisPayload> myQueueItem, ILogger log, CloudBlobContainer inputBlob)
{

    // Perform an actual action against the blob data source for the async readers to be able to check against.
    // This is where your actual service worker processing will be performed

    CloudBlockBlob cbb = inputBlob.GetBlockBlobReference($"{myQueueItem.envelopeProperties.RequestGUID}.blobdata");

    // Now write away the process 
    cbb.UploadTextAsync(JsonConvert.SerializeObject(myQueueItem.RequestObject));
          
}

public class EnvelopedMessage<T> {

    public EnvelopeProperties envelopeProperties {get;set;}
    public T RequestObject {get;set;}

}

public class EnvelopeProperties {  

    public string RequestGUID {get;set;}
    public string RequestSubmittedAt {get;set;}
    public string RequestStatusURL {get;set;}

}

public class ThisPayload {

    public string Id {get;set;}
    public string name {get;set;} 

}
````

````json
{
  "bindings": [
    {
      "name": "myQueueItem",
      "type": "queueTrigger",
      "direction": "in",
      "queueName": "outqueue",
      "connection": "AzureWebJobsStorage"
    },
    {
      "type": "blob",
      "name": "inputBlob",
      "path": "data",
      "connection": "AzureWebJobsStorage",
      "direction": "in"
    }
  ]
}
````

## Http GET AsyncOperationStatusChecker
> This needs to check if the response is complete and if so either return a valet-key to the actual response OR redirect the user to the address in the valet key, if the response is not completed, then the service should return a 202 accepted link back to itself in the http Location header, with an expectation of the time to completion in the http Retry-After header.

````csharp
#r "Newtonsoft.json"
#r "Microsoft.WindowsAzure.Storage"
 
using System;
using System.Net;
using Microsoft.WindowsAzure.Storage; 
using Microsoft.WindowsAzure.Storage.Blob;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq; 
using Microsoft.AspNetCore.Mvc; 
using Microsoft.Extensions.Primitives;

public static async Task<IActionResult> Run(HttpRequest req, CloudBlockBlob inputBlob, string thisGUID, ILogger log)
{ 

    OnCompleteEnum OnComplete = Enum.Parse<OnCompleteEnum>(req.Query["OnComplete"].FirstOrDefault() ?? "Redirect");
    OnPendingEnum OnPending = Enum.Parse<OnPendingEnum>(req.Query["OnPending"].FirstOrDefault() ?? "Accepted");

    log.LogInformation($"C# HTTP trigger function processed a request for status on {thisGUID} - OnComplete {OnComplete} - OnPending {OnPending}");

    // ** Check to see if the blob is present **
    if (await inputBlob.ExistsAsync())
    {
        // ** If it's present, depending on the value of the optional "OnComplete" parameter choose what to do. **
        // Default (OnComplete not present or set to "Redirect") is to return a 302 redirect with the location of a SAS for the document in the location field.   
        // If OnComplete is present and set to "Stream", the function should return the response inline.

        log.LogInformation($"Blob {thisGUID}.blobdata exists, hooray!");

        switch(OnComplete)
        {
            case OnCompleteEnum.Redirect:
            {
                // Awesome, let's use the valet key pattern to offload the download via a SAS URI to blob storage
                return (ActionResult) new RedirectResult(GenerateSASURIForBlob(inputBlob));
            }

            case OnCompleteEnum.Stream:
            {
               // If the action is set to return the file then lets download it and return it back
               // ToDo: this operation is horrible for larger files, we should use a stream to minimize RAM usage.
               return (ActionResult) new OkObjectResult(await inputBlob.DownloadTextAsync());
            }
            
            default:
            {
                throw new InvalidOperationException("How did we get here??");
            }
        }
    }
    else   
    {
        // ** If it's NOT present, then we need to back off, so depending on the value of the optional "OnPending" parameter choose what to do. **
        // Default (OnPending not present or set to "Accepted") is to return a 202 accepted with the location and Retry-After Header set to 5 seconds from now.
        // If OnPending is present and set to "Synchronous" then loop and keep retrying via an exponential backoff in the function until we time out.

        string rqs = $"https://{Environment.GetEnvironmentVariable("WEBSITE_HOSTNAME")}/api/RequestStatus/{thisGUID}";

        log.LogInformation($"Blob {thisGUID}.blob does not exist, still working ! result will be at {rqs}");
        

        switch(OnPending)
        {
            case OnPendingEnum.Accepted:
            {
                // This SHOULD RETURN A 202 accepted 
                return (ActionResult) new AcceptedResult(){ Location = rqs};
            }

            case OnPendingEnum.Synchronous:
            {
                // This should back off and retry returning the data after a period of time timing out when the backoff period hits one minute
                
                int backoff = 250; 

                while (! await inputBlob.ExistsAsync() && backoff < 64000)
                {
                    log.LogInformation($"Synchronous mode {thisGUID}.blob - retrying in {backoff} ms");
                    backoff = backoff * 2; 
                    await Task.Delay(backoff);
                    
                }

                if (await inputBlob.ExistsAsync())
                {
                    switch(OnComplete)
                    {
                        case OnCompleteEnum.Redirect:
                        {
                            log.LogInformation($"Synchronous Redirect mode {thisGUID}.blob - completed after {backoff} ms");
                            // Awesome, let's use the valet key pattern to offload the download via a SAS URI to blob storage
                            return (ActionResult) new RedirectResult(GenerateSASURIForBlob(inputBlob));
                        }

                        case OnCompleteEnum.Stream:
                        {
                            log.LogInformation($"Synchronous Stream mode {thisGUID}.blob - completed after {backoff} ms");
                            // If the action is set to return the file then lets download it and return it back
                            // ToDo: this operation is horrible for larger files, we should use a stream to minimize RAM usage.
                            return (ActionResult) new OkObjectResult(await inputBlob.DownloadTextAsync());
                        }
                        
                        default:
                        {
                            throw new InvalidOperationException("How did we get here??");
                        }
                    }
                }
                else
                {  
                    log.LogInformation($"Synchronous mode {thisGUID}.blob - NOT FOUND after timeout {backoff} ms");
                    return (ActionResult) new NotFoundResult();

                }              
                         
            }

            default:
            {
                throw new InvalidOperationException("How did we get here??");
            }
        }
    }
}

public static string GenerateSASURIForBlob(CloudBlockBlob blob)
{

    SharedAccessBlobPolicy sasConstraints = new SharedAccessBlobPolicy();
    sasConstraints.SharedAccessStartTime = DateTimeOffset.UtcNow.AddMinutes(-5);
    sasConstraints.SharedAccessExpiryTime = DateTimeOffset.UtcNow.AddMinutes(10);
    sasConstraints.Permissions = SharedAccessBlobPermissions.Read;

    //Generate the shared access signature on the blob, setting the constraints directly on the signature.
    string sasBlobToken = blob.GetSharedAccessSignature(sasConstraints);

    //Return the URI string for the container, including the SAS token.
    return blob.Uri + sasBlobToken;

}

public enum OnCompleteEnum {

    Redirect,
    Stream

}

public enum OnPendingEnum {

    Accepted,
    Synchronous

}
````

````json
{
  "bindings": [
    {
      "authLevel": "anonymous",
      "name": "req",
      "type": "httpTrigger",
      "direction": "in",
      "methods": [
        "get"
      ],
      "route": "RequestStatus/{thisGUID}"
    },
    {
      "name": "$return",
      "type": "http",
      "direction": "out"
    },
    {
      "type": "blob",
      "name": "inputBlob",
      "path": "data/{thisGUID}.blobdata",
      "connection": "AzureWebJobsStorage",
      "direction": "in"
    }
  ]
}
````

# [In Logic Apps](#tab/logicapps)

## http POST AsyncRequestInit

> This function needs to accept the work, put it into an envelope wrapper with some metadata, then pop it onto the queue for processing and generate the SAS signature and 202 response back to the client, the location returned should point at the location of the  AsyncOperationStatusChecker Endpoint.

![Image of the structure of an ASync Init logic app](/docs/patterns/_images/arr-Async-Request-Init.png)
````json
{
    "$connections": {
        "value": {
            "azurequeues": {
                "connectionId": "/subscriptions/XXXXXXXX-2ab5-4460-ba7d-d34b5135c24a/resourceGroups/Async_Demos/providers/Microsoft.Web/connections/azurequeues",
                "connectionName": "azurequeues",
                "id": "/subscriptions/XXXXXXXX-2ab5-4460-ba7d-d34b5135c24a/providers/Microsoft.Web/locations/ukwest/managedApis/azurequeues"
            }
        }
    },
    "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "actions": {
            "Condition": {
                "actions": {
                    "HTTP": {
                        "inputs": {
                            "method": "GET",
                            "uri": "@variables('ASyncLogicAppAddress')"
                        },
                        "operationOptions": "DisableAsyncPattern",
                        "runAfter": {},
                        "type": "Http"
                    },
                    "Response": {
                        "inputs": {
                            "body": "@body('HTTP')",
                            "headers": {
                                "location": "@{outputs('HTTP')['headers']['location']}",
                                "retry-after": "@{outputs('HTTP')['headers']['retry-after']}"
                            },
                            "statusCode": "@outputs('HTTP')['statusCode']"
                        },
                        "kind": "Http",
                        "runAfter": {
                            "HTTP": [
                                "Succeeded"
                            ]
                        },
                        "type": "Response"
                    }
                },
                "else": {
                    "actions": {
                        "HTTP_2": {
                            "inputs": {
                                "method": "GET",
                                "uri": "@variables('ASyncLogicAppAddress')"
                            },
                            "limit": {
                                "timeout": "PT120S"
                            },
                            "runAfter": {},
                            "type": "Http"
                        },
                        "Response_2": {
                            "inputs": {
                                "body": "@body('HTTP_2')",
                                "headers": {
                                    "location": "@{outputs('HTTP_2')['headers']['location']}"
                                },
                                "statusCode": "@outputs('HTTP_2')['statusCode']"
                            },
                            "kind": "Http",
                            "runAfter": {
                                "HTTP_2": [
                                    "Succeeded"
                                ]
                            },
                            "type": "Response"
                        }
                    }
                },
                "expression": {
                    "and": [
                        {
                            "equals": [
                                "@variables('sync')",
                                "false"
                            ]
                        }
                    ]
                },
                "runAfter": {
                    "Set_variable": [
                        "Succeeded",
                        "Failed",
                        "Skipped"
                    ]
                },
                "type": "If"
            },
            "Create_a_new_queue": {
                "inputs": {
                    "host": {
                        "connection": {
                            "name": "@parameters('$connections')['azurequeues']['connectionId']"
                        }
                    },
                    "method": "put",
                    "path": "/putQueue",
                    "queries": {
                        "queueName": "@{toLower(concat(triggerOutputs()['relativePathParameters']['RequestQueue'],triggerOutputs()['relativePathParameters']['ObjectType']))}"
                    }
                },
                "runAfter": {
                    "Set_The_Child_Logic_App_Address": [
                        "Succeeded"
                    ]
                },
                "type": "ApiConnection"
            },
            "Gimme_a_GUID": {
                "inputs": {
                    "variables": [
                        {
                            "name": "GUID",
                            "type": "String",
                            "value": "@{guid()}"
                        }
                    ]
                },
                "runAfter": {},
                "type": "InitializeVariable"
            },
            "Init_Sync_Variable": {
                "inputs": {
                    "variables": [
                        {
                            "name": "sync",
                            "type": "String",
                            "value": "false"
                        }
                    ]
                },
                "runAfter": {
                    "Put_a_message_on_a_queue": [
                        "Succeeded"
                    ]
                },
                "type": "InitializeVariable"
            },
            "Initialize_variable": {
                "inputs": {
                    "variables": [
                        {
                            "name": "payload",
                            "type": "String",
                            "value": "@{addProperty(json(triggerBody()),'RequestGUID',variables('GUID'))}"
                        }
                    ]
                },
                "runAfter": {
                    "Create_a_new_queue": [
                        "Succeeded"
                    ]
                },
                "type": "InitializeVariable"
            },
            "Put_a_message_on_a_queue": {
                "inputs": {
                    "body": "@{json(variables('payload'))}",
                    "host": {
                        "connection": {
                            "name": "@parameters('$connections')['azurequeues']['connectionId']"
                        }
                    },
                    "method": "post",
                    "path": "/@{encodeURIComponent(toLower(concat(triggerOutputs()['relativePathParameters']['RequestQueue'],triggerOutputs()['relativePathParameters']['ObjectType'])))}/messages"
                },
                "runAfter": {
                    "Initialize_variable": [
                        "Succeeded"
                    ]
                },
                "type": "ApiConnection"
            },
            "Set_The_Child_Logic_App_Address": {
                "inputs": {
                    "variables": [
                        {
                            "name": "ASyncLogicAppAddress",
                            "type": "String",
                            "value": "https://prod-19.ukwest.logic.azure.com/workflows/XXXXXXXXXXXX/triggers/manual/paths/invoke/@{variables('GUID')}?api-version=2016-10-01&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=XXXXXXXXXXXBq-j6yNch6j5A0ukc8FQSmSW2rok"
                        }
                    ]
                },
                "runAfter": {
                    "Gimme_a_GUID": [
                        "Succeeded"
                    ]
                },
                "type": "InitializeVariable"
            },
            "Set_variable": {
                "inputs": {
                    "name": "sync",
                    "value": "@{coalesce(triggerOutputs()?['Headers']?['Synchronous'],'false')}"
                },
                "runAfter": {
                    "Init_Sync_Variable": [
                        "Succeeded"
                    ]
                },
                "type": "SetVariable"
            }
        },
        "contentVersion": "1.0.0.0",
        "outputs": {},
        "parameters": {
            "$connections": {
                "defaultValue": {},
                "type": "Object"
            }
        },
        "triggers": {
            "manual": {
                "inputs": {
                    "method": "POST",
                    "relativePath": "/{RequestQueue}/{ObjectType}",
                    "schema": {}
                },
                "kind": "Http",
                "type": "Request"
            }
        }
    }
}
````

### Queue Initiated ASync_Worker_Processing

> This should pick the operation up off the queue. remove the payload from the envelope, do something meaningful with it, then write away the end result to the provided blob SAS signature.

![Image of the structure of an ASync Processing logic app](/docs/patterns/_images/arr-Async-Worker.png)

````json
{
    "$connections": {
        "value": {
            "azureblob": {
                "connectionId": "/subscriptions/XXXXXXX-2ab5-4460-ba7d-d34b5135c24a/resourceGroups/Async_Demos/providers/Microsoft.Web/connections/azureblob",
                "connectionName": "azureblob",
                "id": "/subscriptions/XXXXXXX-2ab5-4460-ba7d-d34b5135c24a/providers/Microsoft.Web/locations/ukwest/managedApis/azureblob"
            },
            "azurequeues": {
                "connectionId": "/subscriptions/XXXXXXX-2ab5-4460-ba7d-d34b5135c24a/resourceGroups/Async_Demos/providers/Microsoft.Web/connections/azurequeues",
                "connectionName": "azurequeues",
                "id": "/subscriptions/XXXXXXX-2ab5-4460-ba7d-d34b5135c24a/providers/Microsoft.Web/locations/ukwest/managedApis/azurequeues"
            }
        }
    },
    "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "actions": {
            "Compose": {
                "inputs": {
                    "InsertedID": "no-op",
                    "etag": "@{guid()}"
                },
                "runAfter": {
                    "Parse_JSON": [
                        "Succeeded"
                    ]
                },
                "type": "Compose"
            },
            "Create_blob": {
                "inputs": {
                    "body": "@outputs('Compose')",
                    "headers": {
                        "Content-Type": "application/json"
                    },
                    "host": {
                        "connection": {
                            "name": "@parameters('$connections')['azureblob']['connectionId']"
                        }
                    },
                    "method": "post",
                    "path": "/datasets/default/files",
                    "queries": {
                        "folderPath": "/data",
                        "name": "@body('Parse_JSON')?['RequestGUID']",
                        "queryParametersSingleEncoded": true
                    }
                },
                "runAfter": {
                    "Compose": [
                        "Succeeded"
                    ]
                },
                "runtimeConfiguration": {
                    "contentTransfer": {
                        "transferMode": "Chunked"
                    }
                },
                "type": "ApiConnection"
            },
            "Delete_message": {
                "inputs": {
                    "host": {
                        "connection": {
                            "name": "@parameters('$connections')['azurequeues']['connectionId']"
                        }
                    },
                    "method": "delete",
                    "path": "/@{encodeURIComponent('requestqueueobjecttype')}/messages/@{encodeURIComponent(triggerBody()?['MessageId'])}",
                    "queries": {
                        "popreceipt": "@triggerBody()?['PopReceipt']"
                    }
                },
                "runAfter": {
                    "Create_blob": [
                        "Succeeded"
                    ]
                },
                "type": "ApiConnection"
            },
            "Failed_Processing_-_move_to_secondary_queue": {
                "actions": {
                    "Create_failed_queue": {
                        "inputs": {
                            "host": {
                                "connection": {
                                    "name": "@parameters('$connections')['azurequeues']['connectionId']"
                                }
                            },
                            "method": "put",
                            "path": "/putQueue",
                            "queries": {
                                "queueName": "requestqueueobjecttypefailed"
                            }
                        },
                        "runAfter": {},
                        "type": "ApiConnection"
                    },
                    "Delete_message_2": {
                        "inputs": {
                            "host": {
                                "connection": {
                                    "name": "@parameters('$connections')['azurequeues']['connectionId']"
                                }
                            },
                            "method": "delete",
                            "path": "/@{encodeURIComponent('requestqueueobjecttype')}/messages/@{encodeURIComponent(triggerBody()?['MessageId'])}",
                            "queries": {
                                "popreceipt": "@triggerBody()?['PopReceipt']"
                            }
                        },
                        "runAfter": {
                            "Put_a_message_on_a_queue": [
                                "Succeeded"
                            ]
                        },
                        "type": "ApiConnection"
                    },
                    "Put_a_message_on_a_queue": {
                        "inputs": {
                            "body": "@triggerBody()?['MessageText']",
                            "host": {
                                "connection": {
                                    "name": "@parameters('$connections')['azurequeues']['connectionId']"
                                }
                            },
                            "method": "post",
                            "path": "/@{encodeURIComponent('requestqueueobjecttypefailed')}/messages"
                        },
                        "runAfter": {
                            "Create_failed_queue": [
                                "Succeeded"
                            ]
                        },
                        "type": "ApiConnection"
                    }
                },
                "runAfter": {
                    "Create_blob": [
                        "Failed",
                        "Skipped",
                        "TimedOut"
                    ]
                },
                "type": "Scope"
            },
            "Parse_JSON": {
                "inputs": {
                    "content": "@triggerBody()?['MessageText']",
                    "schema": {
                        "properties": {
                            "OwnerRegion": {
                                "type": "string"
                            },
                            "PayloadContent": {
                                "type": "string"
                            },
                            "PayloadType": {
                                "type": "string"
                            },
                            "ProcessedAt": {
                                "type": "string"
                            },
                            "ProcessedUserEmail": {
                                "type": "string"
                            },
                            "RequestGUID": {
                                "type": "string"
                            },
                            "Source": {
                                "type": "string"
                            }
                        },
                        "type": "object"
                    }
                },
                "runAfter": {},
                "type": "ParseJson"
            }
        },
        "contentVersion": "1.0.0.0",
        "outputs": {},
        "parameters": {
            "$connections": {
                "defaultValue": {},
                "type": "Object"
            }
        },
        "triggers": {
            "Check_the_queue_every_5_seconds": {
                "inputs": {
                    "host": {
                        "connection": {
                            "name": "@parameters('$connections')['azurequeues']['connectionId']"
                        }
                    },
                    "method": "get",
                    "path": "/@{encodeURIComponent('requestqueueobjecttype')}/message_trigger",
                    "queries": {
                        "visibilitytimeout": "15"
                    }
                },
                "recurrence": {
                    "frequency": "Second",
                    "interval": 5
                },
                "splitOn": "@triggerBody()?['QueueMessagesList']?['QueueMessage']",
                "type": "ApiConnection"
            }
        }
    }
}
````
### http GET AsyncResponse
> This needs to check if the response is complete and if so either return a valet-key to the actual response OR redirect the user to the address in the valet key, if the response is not completed, then the service should return a 202 accepted link back to itself in the http Location header, with an expectation of the time to completion in the http Retry-After header.

![Image of the structure of an ASync Request Init logic app](/docs/patterns/_images/arr-Async-Response.png)

````json
{
    "$connections": {
        "value": {
            "azureblob": {
                "connectionId": "/subscriptions/XXXXXXX-2ab5-4460-ba7d-d34b5135c24a/resourceGroups/Async_Demos/providers/Microsoft.Web/connections/azureblob",
                "connectionName": "azureblob",
                "id": "/subscriptions/XXXXXXX-2ab5-4460-ba7d-d34b5135c24a/providers/Microsoft.Web/locations/ukwest/managedApis/azureblob"
            }
        }
    },
    "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "actions": {
            "Completed_Processing_-_Return_Redirect_to_SAS_Url": {
                "inputs": {
                    "body": {
                        "ProcessingStatus": "CompleteShouldRedirect",
                        "Status": "Processing Complete, follow the location header to download the resource",
                        "location": "@{body('Create_SAS_URI_by_path')?['WebUrl']}"
                    },
                    "headers": {
                        "ProcessingStatus": "ShouldRedirect",
                        "location": "@body('Create_SAS_URI_by_path')?['WebUrl']"
                    },
                    "statusCode": 201
                },
                "kind": "Http",
                "runAfter": {
                    "Create_SAS_URI_by_path": [
                        "Succeeded"
                    ]
                },
                "type": "Response"
            },
            "Create_SAS_URI_by_path": {
                "inputs": {
                    "body": {
                        "AccessProtocol": "HttpsOnly",
                        "ExpiryTime": "@{addMinutes(utcNow(),10)}",
                        "Permissions": "Read"
                    },
                    "host": {
                        "connection": {
                            "name": "@parameters('$connections')['azureblob']['connectionId']"
                        }
                    },
                    "method": "post",
                    "path": "/datasets/default/CreateSharedLinkByPath",
                    "queries": {
                        "path": "/data/@{triggerOutputs()['relativePathParameters']['filename']}"
                    }
                },
                "runAfter": {
                    "Set_Retry_Time": [
                        "Succeeded"
                    ]
                },
                "type": "ApiConnection"
            },
            "Not_completed,_return_202_with_location_and_retry-after_headers": {
                "inputs": {
                    "body": {
                        "ProcessingStatus": "ProcessingInProgress",
                        "Status": "Processing In Progress, please come back to this URL after the retry-after time has expired (in @{variables('retry')} seconds).",
                        "location": "@{body('Create_SAS_URI_by_path')?['WebUrl']}",
                        "retry-after": "@{variables('retry')}"
                    },
                    "headers": {
                        "ProcessingStatus": "ProcessingInProgressRetryAfter",
                        "location": "https://prod-19.ukwest.logic.azure.com/workflows/XXXXXXXXXXXXXXXXXXXXX/triggers/manual/paths/invoke/@{triggerOutputs()['relativePathParameters']['filename']}?api-version=2016-10-01&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=XXXXXXXXXXXXXXq-j6yNch6j5A0ukc8FQSmSW2rok",
                        "retry-after": "@{variables('retry')}"
                    },
                    "statusCode": 202
                },
                "kind": "Http",
                "runAfter": {
                    "Create_SAS_URI_by_path": [
                        "Failed"
                    ]
                },
                "type": "Response"
            },
            "Set_Retry_Time": {
                "inputs": {
                    "variables": [
                        {
                            "name": "retry",
                            "type": "String",
                            "value": "@addSeconds(utcNow(),5)"
                        }
                    ]
                },
                "runAfter": {},
                "type": "InitializeVariable"
            },
            "Terminate": {
                "inputs": {
                    "runStatus": "Cancelled"
                },
                "runAfter": {
                    "Not_completed,_return_202_with_location_and_retry-after_headers": [
                        "Succeeded"
                    ]
                },
                "type": "Terminate"
            },
            "Terminate_2": {
                "inputs": {
                    "runStatus": "Succeeded"
                },
                "runAfter": {
                    "Completed_Processing_-_Return_Redirect_to_SAS_Url": [
                        "Succeeded"
                    ]
                },
                "type": "Terminate"
            }
        },
        "contentVersion": "1.0.0.0",
        "outputs": {},
        "parameters": {
            "$connections": {
                "defaultValue": {},
                "type": "Object"
            }
        },
        "triggers": {
            "manual": {
                "inputs": {
                    "method": "GET",
                    "relativePath": "/{filename}",
                    "schema": {}
                },
                "kind": "Http",
                "type": "Request"
            }
        }
    }
}
````
---

## Next steps

The following information may be relevant when implementing this pattern:

- [Azure Logic Apps - Perform Long Running Tasks] (https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-create-api-app#perform-long-running-tasks-with-the-polling-action-pattern)
- [Sample Code - ASP.NET Implementation] (https://github.com/jeffhollan/LogicAppsAsyncResponseSample)
