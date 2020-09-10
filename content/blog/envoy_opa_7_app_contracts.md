+++
author = "Helpful Badger"
title = "Putting It All Together with Composite Authorization"
draft = false
date  = 2020-09-07T22:37:37-04:00
description = "Learn how to Implement Application Specific Authorization Rules"
tags = [
    "OPA",
    "Open Policy Agent",
    "JWS",
    "Signatures"
]
categories = [
    "cloud",
    "Security",
    "Authentication"
]
images  = ["img/2020/08/app-authorization-A6qNzfJXRGQ-unsplash.jpg"]
+++

<span>Photo by <a href="https://unsplash.com/@thombradley?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Thom Bradley</a> on <a href="https://unsplash.com/s/photos/ios-app?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>


# Getting Started with Envoy & Open Policy Agent --- 07 ---
## Learn how to Implement Application Specific Authorization Rules

This is the 7th Envoy & Open Policy Agent Getting Started Guide. Each guide is intended to explore a single feature and walk through a simple implementation. Each guide builds on the concepts explored in the previous guide with the end goal of building a very powerful authorization service by the end of the series. 

The source code for this getting started example is located on Github. <span style="color:blue"> ------>  [Envoy & OPA GS # 7](https://github.com/helpfulBadger/envoy_getting_started/tree/master/07_opa_validate_method_uri) </span>


Here is a list of the Getting Started Guides that are currently available.

## Getting Started Guides

1. <span style="color:blue">[Using Envoy as a Front Proxy]({{< ref "/blog/envoy_opa_1_front_proxy.md" >}} "Learn how to set up Envoy as a front proxy with docker")</span>
1. <span style="color:blue">[Adding Observability Tools]({{< ref "/blog/envoy_opa_2_adding_observability.md" >}} "Learn how to add ElasticSearch and Kibana to your Envoy front proxy environment")</span>
1. <span style="color:blue">[Plugging Open Policy Agent into Envoy]({{< ref "/blog/envoy_opa_3_adding_open_policy_agent.md" >}} "Learn how to use Open Policy Agent with Envoy for more powerful authorization rules")</span>
1. <span style="color:blue">[Using the Open Policy Agent CLI]({{< ref "/blog/envoy_opa_4_opa_cli.md" >}} "Learn how to use Open Policy Agent Command Line Interface")</span>
1. <span style="color:blue">[JWS Signature Validation with OPA]({{< ref "/blog/envoy_opa_5_opa_jws.md" >}} "Learn how to validate JWS signatures with Open Policy Agent")</span>
1. <span style="color:blue">[JWS Signature Validation with Envoy]({{< ref "/blog/envoy_opa_6_envoy_jws.md" >}} "Learn how to validate JWS signatures natively with Envoy")</span>
1. <span style="color:blue">[Putting It All Together with Composite Authorization]({{< ref "/blog/envoy_opa_7_app_contracts.md" >}} "Learn how to Implement Application Specific Authorization Rules")</span>

# Draft Article that is Under construction 
## Introduction

In this example we are going to simulate an ecommerce company called `Example.com` that has a published set of APIs and multiple client applications. Each client application has:
* Different access rights to each published API 
* Different operations they are allowed to perform on the APIs that it has access to
* Different types of users that are allowed to use the application

There are a lot of other rules that we will eventually be interested in implementing. However, this set of rules is complex enough to demonstrate how we can put together what we have learned so far into a little authorization system.

### Example.com's APIs

Here are Example.com's published APIs.

<pre><code>
/<span style="color:brown"><strong>api</strong></span>/<span style="color:blue"><strong>customer</strong></span>/*
/<span style="color:brown"><strong>api</strong></span>/<span style="color:blue"><strong>customer</strong></span>/*/account/*
/<span style="color:brown"><strong>api</strong></span>/<span style="color:blue"><strong>customer</strong></span>/*/messages/*
/<span style="color:brown"><strong>api</strong></span>/<span style="color:blue"><strong>customer</strong></span>/*/order/*
/<span style="color:brown"><strong>api</strong></span>/<span style="color:blue"><strong>customer</strong></span>/*/paymentcard/*
/<span style="color:brown"><strong>api</strong></span>/featureFlags
/<span style="color:brown"><strong>api</strong></span>/<span style="color:green"><strong>order</strong></span>/*
/<span style="color:brown"><strong>api</strong></span>/<span style="color:green"><strong>order</strong></span>/*/payment/*
/<span style="color:brown"><strong>api</strong></span>/product/*
/<span style="color:brown"><strong>api</strong></span>/shipment/*
</code></pre>

* The customer API, `/api/customer/*`, allows users manage customer profiles in our customer system of record. 
* The accounts API, `/api/customer/*/account/*`, allows users to manage accounts for a specific customer in the system of record for accounts. 
* The messages API, `/api/customer/*/messages/*`, allows users to send and receive messages related to a specific customer in the messaging system. 
* The customer's order API, `/api/customer/*/order/*`, allows users to manage a customer's orders.
* The customer's payment card API, `/api/customer/*/paymentcard/*`, allows users to manage a customer's debit or credit cards.
* The feature flag API, `/api/featureFlags`, allows an application to retrieve the feature flags for the app e.g. which features are turned on or off and any customer specific features that are available or not.
* The order API, `/api/order/*`, allows users to get, create, update or delete orders for any customer.
* The order payments API, `/api/order/*/payment/*`, allows users to manage payments for any order. No shopping cart API is provided. An order in draft status is the equivalent of a shopping cart.
* The product API, `/api/product/*`, allows users to manage the products that are available for purchase.
* The shipment API, `/api/shipment/*`, allows users to manage shipment for an order.

### API Endpoint Definition

As the basis for our security policies we need a data structure that contains all of possible actions that a user and client application can take. For this example, we define a URI pattern and a method being attempted on that URI as an `endpoint`. The `id` field uniquely identifies each endpoint and will be used in the process of actually specifying what endpoints an application has access to.  
<pre><code>
[
    {<span style="color:blue"><strong>"id"</strong></span>:"001",<span style="color:blue"><strong>"method"</strong></span>:"GET",   <span style="color:blue"><strong>"pattern"</strong></span>:"/api/customer"},
    {<span style="color:blue"><strong>"id"</strong></span>:"002",<span style="color:blue"><strong>"method"</strong></span>:"POST",  <span style="color:blue"><strong>"pattern"</strong></span>:"/api/customer"},
    {<span style="color:blue"><strong>"id"</strong></span>:"003",<span style="color:blue"><strong>"method"</strong></span>:"DELETE",<span style="color:blue"><strong>"pattern"</strong></span>:"/api/customer/*"},
    {<span style="color:blue"><strong>"id"</strong></span>:"004",<span style="color:blue"><strong>"method"</strong></span>:"GET",   <span style="color:blue"><strong>"pattern"</strong></span>:"/api/customer/*"},
    {<span style="color:blue"><strong>"id"</strong></span>:"005",<span style="color:blue"><strong>"method"</strong></span>:"POST",  <span style="color:blue"><strong>"pattern"</strong></span>:"/api/customer/*"},
    {<span style="color:blue"><strong>"id"</strong></span>:"006",<span style="color:blue"><strong>"method"</strong></span>:"PUT",   <span style="color:blue"><strong>"pattern"</strong></span>:"/api/customer/*"},
    {<span style="color:blue"><strong>"id"</strong></span>:"007",<span style="color:blue"><strong>"method"</strong></span>:"GET",   <span style="color:blue"><strong>"pattern"</strong></span>:"/api/customer/*/account"},
    {<span style="color:blue"><strong>"id"</strong></span>:"008",<span style="color:blue"><strong>"method"</strong></span>:"POST",  <span style="color:blue"><strong>"pattern"</strong></span>:"/api/customer/*/account"},
...
]
</code></pre>

### Client Application API Contracts

The next piece of data that we need to build our example authorization solution is a mapping between each client application and the endponts that it is allowed to access. The data structure below holds that information. The unique ID for each application is a `key` in this data structure and the value is an array of all of the endpoint IDs that the application has access to. 
<pre><code>
apiPermissions = {
  <span style="color:blue"><strong>"app_123456"</strong></span>: [ <span style="color:green"><strong>
      "001","004","007","010","012","015","018","021","024","027","031","034","037","040","043","046","049","052","055" </strong></span>
      ],
  <span style="color:blue"><strong>"app_000123"</strong></span>:[ <span style="color:green"><strong>
    "001", "002", "003", "004", "005", "006", "007", "008", "009", "010", 
    "011", "012", "013", "014", "015", "016", "017", "018", "019", "020",
    "021", "022", "023", "024", "025", "026", "027", "028", "029", "030",
    "031", "032", "033", "034", "035", "036", "037", "038", "039", "040",
    "041", "042", "043", "044", "045", "046", "047", "048", "049", "050",
    "051", "052", "053", "054", "055", "056", "057" </strong></span>
  ]
}
</code></pre>

### Authorized End User Identity Providers / JWS Issuers for each Client Application

Just for fun and simplicity we want to make sure that a hacker has not found a way to use or simulate using an application that is intended for another type of user. So, the data structure below lists the identity providers that are permitted for each application. So, the data structure below means that `app_123456` is only allowed to be used by external customers. `app_000123` is intended only for use by employees and contractors of example.com (i.e. the workforce). This is a more powerful application that can take action on behalf of the company and on behalf of any of the company's customer. This is very coarse grained security. In a real application we would also put in a lot of user specific rights and access controls. In a later getting started guide we will show how we can start to layer OPA Policies, getting volatile data from external sources and other techniques to implement more realistic security policies. 

<pre><code>
idProviderPermissions = {
  <span style="color:blue"><strong>"app_123456"</strong></span>:["customerIdentity.example.com"],
  <span style="color:blue"><strong>"app_000123"</strong></span>:["workforceIdentity.example.com"]
}
</code></pre>


