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

<pre><code>/<span style="color:brown"><strong>api</strong></span>/<span style="color:blue"><strong>customer</strong></span>/*
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
<pre><code>[
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
<pre><code>apiPermissions = {
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

<pre><code>idProviderPermissions = {
  <span style="color:blue"><strong>"app_123456"</strong></span>:["customerIdentity.example.com"],
  <span style="color:blue"><strong>"app_000123"</strong></span>:["workforceIdentity.example.com"]
}
</code></pre>

## Our Environment

In this example we have replaced our HTTPBin container as our ultimate destination with Hoverfly. <span style="color:blue">[Hoverfly](https://hoverfly.io/)</span> is a light-weight, super fast API mocking server that is API driven and highly extensible. This is used to simulate all of the APIs that we defined above. The API Mocks are defined in the <span style="color:blue">[config.json](https://github.com/helpfulBadger/envoy_getting_started/blob/master/07_opa_validate_method_uri/compose/hoverfly/config.json)</span> file in the `compose\hoverfly` directory.

The other change that we have made is a much more sophisticated and comprehensive set of tests using Newman. <span style="color:blue">[Newman](https://learning.postman.com/docs/running-collections/using-newman-cli/command-line-integration-with-newman/)</span> is a command line driven postman collection runner. Postman has a testing capability. This feature runs tests that are expressed in Javascript and uses a built in library of test functions. The <span style="color:blue">[Postman collection for Getting Started #7](https://github.com/helpfulBadger/envoy_getting_started/blob/master/07_opa_validate_method_uri/data/Envoy_OPA.postman_collection.json)</span> with tests is included in this example in the data directory that 

<img class="special-img-class" src="/img/2020/09/Envoy-front proxy-Hoverfly.svg" /><br>

## Authorization Policies

User and system identity information is communicated as follows:
* We have some sort of access token in the Authorization header. What API gateway is in use and how it works is out of scope for this example.
* Some user authentication system that we are using (out of scope of this article) has produced evidence of authentication in the form of an JWS Identity Token and given this to our client application. The client application places the identity token in the `Actor-Token` header of the request.
* The API gateway, the client application itself or another system (out of scope of this article), has securely authenticated the client application and given it a JWS token to assert information in a tamper proof way about our client application. 

### Stage 1 - Envoy validates JWS Tokens
These JWS tokens are validated by Envoy before they are ever sent to OPA. Envoy and OPA Getting Started Guide #5 showed us how to do this. Our Envoy.yaml file for this example has made one small change to the <span style="color:blue">[JWT provider configurations](https://github.com/helpfulBadger/envoy_getting_started/blob/master/07_opa_validate_method_uri/envoy.yaml#L26-L56)</span>. In this example we need to be able to support a variety of different types of users. Since we have 2 different user types / token issuers, we create to different JWT Provider configurations (one for each identity provider). Both of them use the `from_headers:` parameter to specify the token source as the `Actor-Token` header. 

Because of this variability we had to change our logic for token validation and enforcement:

The configuration snippet below is how we express this logic:
* The `match` object specifies that any URI must meet the requirements in the `requires` object
* The `requires` object specifies that a request must have:  <pre><code>(<span style="color:red"><u>a Gateway issued system identity token</u></span>) <strong>AND a user with</strong> ( <span style="color:blue"><u>a workforce Identity token</u></span> <strong>OR</strong> <span style="color:blue"><u>a Consumer Identity Token</u></span> ))</code></pre> NOTE: Scroll right as needed. The color coding above matches the translation to YAML configuration below.

<pre><code>                      rules:
                        - match:
                            prefix: /
                          requires:
                            <span style="color:red"><strong>requires_all</strong></span>:
                              requirements:
                                - <span style="color:red"><strong>provider_name</strong></span>: gateway_provider
                                - <span style="color:blue"><strong>requires_any</strong></span>:
                                    requirements:
                                      - <span style="color:blue"><strong>provider_name</strong></span>: workforce_provider
                                      - <span style="color:blue"><strong>provider_name</strong></span>: consumer_provider
</code></pre>

* The `requires_all` object specifies that all of the requirements in the `requirements` array must be true to pass.
* The `requirements` array contains a `provider_name` and a `requires_any` clause. If there were 3 or 4 elements in the array then all of the requirements would need to be true to pass. In this example we only have 2 requirements and one of those has sub-requirements.
* The `requires_any` object contains another `requirements` array. If any of the requirements in this array are true then the `requires_any` will also evaluate to true.

#### Passing Envoy's Dynamic Meta data to OPA

We can pass the validated JWS token body to Open Policy Agent by adding a couple of tags to Envoy's configuration. In the token definition section, we add the `payload_in_metadata` property and give the token a name. In the example below we've given the token the name `actor_token`. See below for the configuration snippet.

<pre><code>                        workforce_provider:
                          issuer: workforceIdentity.example.com
                          audiences:
                          - apigateway.example.com
                          from_headers:
                          - name: "actor-token"
                            value_prefix: ""
                          forward: true
                          <span style="color:blue"><strong>payload_in_metadata: "actor_token"</strong></span>
                          local_jwks:
                            inline_string: "{\"keys\":[...]}" 
</code></pre>

To pass this data to Open Policy Agent, in the `envoy.filters.http.ext_authz` section, add the array named `metadata_context_namespaces`. This array specifies the names of the Envoy dynamic metadata namespaces to forward to the OPA Authorization Service. See the configuration change highlighted in blue below.

<pre><code>                  - name: envoy.filters.http.ext_authz
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
                      failure_mode_allow: false
                      grpc_service:
                        google_grpc:
                          target_uri: opa:9191
                          stat_prefix: ext_authz
                        timeout: 0.5s
                      include_peer_certificate: true
                      <span style="color:blue"><strong>metadata_context_namespaces:
                      - envoy.filters.http.jwt_authn</strong></span>
                  - name: envoy.filters.http.router
                    typed_config: {}
</code></pre>

These changes add some data to the `metadata_context`. For each dynamic metadata namespace added, it will add it's data under a key that matches the namespace name in the `filter_metadata` object. For this example it adds each token's payload in the `envoy.filters.http.jwt_authn` object underneath the `fields` object. As you can see from the example below, it adds a lot of extra hierarchy via nested objects that express data types to the original object. So, it may not be worth it to have Envoy forward the validated payload. For us configuring Envoy to forward the token and simply reading the token payload directly is less complicated. 

See the resultant `metadata_context` below for an example.
<pre><code>  "input": {
    "attributes": {
      "destination": { "...":"..." },
      "metadata_context": {
        <span style="color:blue"><strong>"filter_metadata": {
          "envoy.filters.http.jwt_authn": {
            "fields": {
              "workforce_provider": {
                "Kind": {
                  "StructValue": {
                    "fields": {
                      "aud": {
                        "Kind": {
                          "ListValue": {
                            "values": [
                              { "Kind": { "StringValue": "apigateway.example.com"} },
                              { "Kind": { "StringValue": "protected-stuff.example.com" } }
                            ]
                          }
                        }
                      },
                      "auth_time": { "Kind": { "NumberValue": 1597676718 } },
                      "azp": { "Kind": { "StringValue": "app_123456" } },
                      "exp": { "Kind": { "NumberValue": 2735689600 } },
                      "iat": { "Kind": { "NumberValue": 1597676718 } },
                      "iss": { "Kind": { "StringValue": "workforceIdentity.example.com" } },
                      "jti": { "Kind": { "StringValue": "mPPdwJ7Jrr2MQzS" } },
                      "sub": { "Kind": { "StringValue": "workforceIdentity:406319" } }
                    }
                  }
                }
              }
            }
          }
        }</strong></span>
      },
      "request": {"...": "..."},
      "source":  {"...": "..."}
    },
    "parsed_body": null,
    "parsed_path": ["..."],
    "parsed_query": {},
    "truncated_body": false
  }
</code></pre>

### Authorization Stage 2 - Open Policy Agent Policies

#### Extracting the JWS Token Payloads

Since we also have the configuration setting `forward: true`, we can simply use OPA's built in capabilities to more easily access the data that we need from the tokens. 
* The REGO snippets below show how we pull the tokens that we need out of the `actor-token` header and `app-token` header. 
* Since we won't get to this point unless Envoy successfully validated the tokens, we can simply decode the payload. The `io.jwt.decode()` functions do that for us.
* We don't need the header and signature values so we tell OPA to discard those with underscores in the destructuring fragment `[ _, p, _ ]`
* The payload is then stored in the `actor` and `app` variables

See below for the REGO code. 

<pre><code>import input.attributes.request.http.headers["actor-token"] as <span style="color:blue"><strong>actorToken</strong></span>
import input.attributes.request.http.headers["app-token"] as <span style="color:red"><strong>appToken</strong></span>

<span style="color:blue"><strong>actor</strong></span> = p {
  [ _, p, _ ] := io.jwt.decode(<span style="color:blue"><strong>actorToken</strong></span>)	
}

<span style="color:red"><strong>app</strong></span>  = p {
  [ _, p, _ ] :=  io.jwt.decode(<span style="color:red"><strong>appToken</strong></span>)
}
</code></pre>

#### Making the Authorization Decision

Next, we will move over to the punchline, the REGO rule that makes our authorization decision. As we can see from the rule below our rule defaults to `false` a.k.a. `deny` access. The next part of the rule says:
* IF the API is permitted for our client app
* AND the type of user is appropriate for the client app
* THEN set the decision to `true` a.k.a. `allow` the request to flow through.

<pre><code>default decision = false
decision {
  <span style="color:blue"><strong>apiPermittedForClient</strong></span>
  <span style="color:red"><strong>userTypeAppropriateForClient</strong></span>
}
</code></pre>

#### How do we determine if the API is Permitted for the Client Application?

Now lets work backwards to see how we determine if these two conditions are true. 

Let's look at how `apiPermittedForClient` is defined. Its default value is false. To see if it should override this default value:
* The first section says that the API is permitted for the client application if the endpoint ID that we are calling is found in the array endpoints that our app is allowed to access. 
* `apiPermissions[ app.client_id ][_] == endpointID` means take every `[_]` element in the permissions array that matches our Client App's ID and see if it equals our endpointID.

Determining our endpoint with our endpoint rule is a little more complicated:
* The series of rules effectively acts as progressive filters that reduces the search space with each statement
* `some i` is used when we need to match a specific element in a dataset with some other specific value. This just declares our iterator and names it.
* `endpoints[i].method == http_request.method` searches for a value for i where the method property of that array element equals the request method that we are decisioning. This statement effectively discards ~75% of our reference data and only keeps the array elements that match our request method.
* `p := trim_right( http_request.path, "/")  #strip trailing slash if present` sets `p` equal to the request path with any trailing slashes removed
* With the path now normalized, it should work in our REGEX expression search.
* `glob.match( lower(endpoints[i].pattern), ["/"], lower(p) )` takes the lower case pattern from our reference data and the lower case path from our request and looks for any patterns in our reference array that match. Since the statement above discards ~75% of the reference array to satisfy that match criteria, this statement searches that space for any values of i where this statement is true too.
* Our magic variable i should only represent a single value at this point. 
* So, now we want to pull back the endpoint id from the only reference data object that should match the statement `epID := endpoints[i].id` does this for us.

See the REGO code below for how this section fits together. 

<pre><code>default <span style="color:blue"><strong>apiPermittedForClient</strong></span> = false
<span style="color:blue"><strong>apiPermittedForClient</strong></span> {
  apiPermissions[ app.client_id ][_] == <span style="color:blue"><strong>endpointID</strong></span>
}

<span style="color:blue"><strong>endpointID</strong></span> = epID {
  some i
  endpoints[i].method == http_request.method
  p := trim_right( http_request.path, "/")  #strip trailing slash if present
  glob.match( lower(endpoints[i].pattern), ["/"], lower(p) )
  epID := endpoints[i].id
} else = "none"
</code></pre>

#### How do we determine if the Type of User is Permitted for the Client Application?

Now, let's look at how `userTypeAppropriateForClient` is defined. Its default value is false. To see if it should override this default value:
* It uses our reference data object, `idProviderPermissions`
* By using the client ID from the app token, `app.client_id`, it narrows down the dataset search
* The `[_]` fragment says: for any element in the array, see if one of the equals the issuer that we have in the actor token `actor.iss`.

<pre><code>default <span style="color:red"><strong>userTypeAppropriateForClient</strong></span> = false
<span style="color:red"><strong>userTypeAppropriateForClient</strong></span> {
  idProviderPermissions[ app.client_id ][_] == actor.iss
}
</code></pre>

#### If we don't have a match, how do we communicate reject reasons?

Whether rejection reasons are returned to the user or not depends on the circumstances and security risks posed by providing this transparency. However, it still makes sense to log the reject reasons. This example shows you how to report back which rules failed and resulted in a denial of access. 
* We do this by defining multiple messages array rules. 
* For each occurrence, we put a single rule reference and statically define a message describes a matching failure reason. This is another reason why we defined single condition rules above. It allows us to test each individual rule to determine what the rejection reason was.
* In this first messages block we look for `not apiPermittedForClient`, this will result in a value of true if `apiPermittedForClient` is either `false` or `undefined`. It then adds the message defined in this block to the messages array.
* The same is true for the second messages rule block. 
* This section will result in a messages array that has either 0, 1 or 2 elements. 

<pre><code>messages[ msg ]{
  <span style="color:blue"><strong>not apiPermittedForClient</strong></span>
  msg := {
    "id"  : "1",
    "priority"  : "5",
    "message" : "The requested API Endpoint is not permitted for the client application"
  }
}

messages[ msg ]{
  <span style="color:red"><strong>not userTypeAppropriateForClient</strong></span>
  msg := {
    "id"  : "2",
    "priority"  : "5",
    "message" : "The authenticated user type is not allowed for the client application"
  }
}
</code></pre>

#### Now let's return the results to Envoy

Envoy is looking for results of the `allow` rule. The structure of our response is defined by the Envoy external authorization API contract. OPA is also a little bit finicky when assigning values. So we were very careful to make sure that there was no way that any of the variables in this section could end up as undefined. If anything is undefined in the object that we are trying to return, then this rule will fail and the entire result returned to Envoy will be undefined.
* Since we have setup a default value for the decision variable, it will always be either true or false
* Since Envoy validated our JWS tokens before our policy executed, the User and App tokens will always be valid. So, we hard coded the value of these headers to the string "true". The header contract states that all headers must have strings as keys and strings as values. 
* The endpointID has a default value of the string "none". Since all of the IDs in our reference data are already strings, if there is a match in the reference data, the result here will always be a string.
* The body value must also be a string. So, we can't return any of the other mock data that we have here as an object. So first we declare it as an anonymous object then immediately pass it to the json marshaller. This will return a string. The json marshaller should be able to handle any numeric, null or undefined values for us and always return a string. 
* With these guarantees, we can be confident that this block will always return a defined result to Envoy

<pre><code>allow = response {
  response := {
    "allowed": <span style="color:blue"><strong>decision</strong></span>,
    "headers": {
      "X-Authenticated-User": "true", 
      "X-Authenticated-App": "true",
      "Endpoint-ID": endpointID,
      "Content-Type": "application/json"
    },
    "body" : <span style="color:blue"><strong>json.marshal</strong></span>(
                      {
                        "Authorized-User-Type": userTypeAppropriateForClient,
                        "Authorized-Endpoint": apiPermittedForClient,
                        "Authorization-Failures" : messages,
                        "Requested-Path": http_request.path,
                        "Requested-Method": http_request.method
                      })
  }
} 
</code></pre>

## Running our Example Through Its Paces


