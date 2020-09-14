+++
author = "Helpful Badger"
title = "Configuring Envoy Logs, Taps and Traces"
draft = false
date  = 2020-09-10T10:36:36-04:00
description = "Learn how to configure Envoy's access logs, taps for capturing full requests & responses and traces"
tags = [
    "logs",
    "taps",
    "traces",
    "envoy"
]
categories = [
    "cloud",
    "observability",
    "service mesh"
]
images  = ["img/2020/08/tap-dPr7uyrFMXo-unsplash.jpg"]
+++


<span>Photo by <a href="https://unsplash.com/@kazukyakayashi?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Kazuky Akayashi</a> on <a href="https://unsplash.com/@kazukyakayashi?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>


# Getting Started with Envoy & Open Policy Agent --- 08 ---
## Learn how to configure Envoy's access logs, taps for capturing full requests & responses and traces for capturing end to end flow

This is the 8th Envoy & Open Policy Agent Getting Started Guide. Each guide is intended to explore a single feature and walk through a simple implementation. Each guide builds on the concepts explored in the previous guide with the end goal of building a very powerful authorization service by the end of the series. 

The source code for this getting started example is located on Github. <span style="color:blue"> ------>  [Envoy & OPA GS # 8](https://github.com/helpfulBadger/envoy_getting_started/tree/master/08_log_taps_traces) </span>


Here is a list of the Getting Started Guides that are currently available.

## Getting Started Guides

1. <span style="color:blue">[Using Envoy as a Front Proxy]({{< ref "/blog/envoy_opa_1_front_proxy.md" >}} "Learn how to set up Envoy as a front proxy with docker")</span>
1. <span style="color:blue">[Adding Observability Tools]({{< ref "/blog/envoy_opa_2_adding_observability.md" >}} "Learn how to add ElasticSearch and Kibana to your Envoy front proxy environment")</span>
1. <span style="color:blue">[Plugging Open Policy Agent into Envoy]({{< ref "/blog/envoy_opa_3_adding_open_policy_agent.md" >}} "Learn how to use Open Policy Agent with Envoy for more powerful authorization rules")</span>
1. <span style="color:blue">[Using the Open Policy Agent CLI]({{< ref "/blog/envoy_opa_4_opa_cli.md" >}} "Learn how to use Open Policy Agent Command Line Interface")</span>
1. <span style="color:blue">[JWS Signature Validation with OPA]({{< ref "/blog/envoy_opa_5_opa_jws.md" >}} "Learn how to validate JWS signatures with Open Policy Agent")</span>
1. <span style="color:blue">[JWS Signature Validation with Envoy]({{< ref "/blog/envoy_opa_6_envoy_jws.md" >}} "Learn how to validate JWS signatures natively with Envoy")</span>
1. <span style="color:blue">[Putting It All Together with Composite Authorization]({{< ref "/blog/envoy_opa_7_app_contracts.md" >}} "Learn how to Implement Application Specific Authorization Rules")</span>
1. <span style="color:blue">[Configuring Envoy Logs Taps and Traces]({{< ref "/blog/envoy_opa_8_logs_taps_and_traces.md" >}} "Learn how to configure Envoy's access logs, taps for capturing full requests & responses and traces")</span>

# Draft Article that is Under construction 
## Introduction

In this example we are going to use both a Front Proxy deployment and a service mesh deployment to centralize log configuration, capture full requests and responses with taps and to inject trace information. Logs and taps can be done transparently without the knowledge nor cooperation from our applications. However, for traces there cooperation from our app is required to forward the trace headers to the next link in the chain. 

The diagram below shows what we will be building. 

<img class="special-img-class" src="/img/2020/09/Envoy-front proxy-gs_08_logs_taps_traces.svg" /><br>

The Envoy instances throughout our network will be streaming logs, taps and traces on behalf of the applications involved in the request flow.

## Let's Start with Configuring Our Logs

Envoy gives you the ability configure what it logs as a request goes though the proxy. Envoy's web site has documentation for <span style="color:blue">[access log configuration](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#configuration)</span>. There are a few things to be aware of:
* Each request / response header must be individually configured to make it to the logs. I haven't found a log-all-headers capability. This is can be a good thing because sensitive information doesn't get logged without a conscious decision to log it. On the other side of the coin, if you are using multiple trace providers, then you will miss trace headers from these other solutions until:
    1. You are even aware that they are in use
    1. You update the configuration for every node
    1. There is no ability to run analytics from the time they started being used. You will only be able to go back to the time you became aware of these new headers and updated your configurations. 
* The property name `match_config` that we use in this article is entering deprecation when Envoy 1.16 comes out. I'll will update this post to the new property name once 1.16 is released.
* We will be using Elastic Common Schema in this example to the extent that we can with Envoy 1.15's configuration limitations.
* Also coming with Envoy 1.16 will be nested JSON support. Once released I'll update the example to switch from the dot notation used here to nested JSON. 

Read through this snippet of envoy configuration. The full configuration files for our 3 envoy instances are located here:
* <span style="color:blue">[Front Proxy's Configuration](https://github.com/helpfulBadger/envoy_getting_started/blob/master/08_log_taps_traces/front-envoy-jaeger.yaml#L54-L119)</span>
* <span style="color:blue">[Service 1's Envoy Configuration](https://github.com/helpfulBadger/envoy_getting_started/blob/master/08_log_taps_traces/service1-envoy-jaeger.yaml#L51-L116)</span>
* <span style="color:blue">[Service 2's Envoy Configuration](https://github.com/helpfulBadger/envoy_getting_started/blob/master/08_log_taps_traces/service2-envoy-jaeger.yaml#L51-L116)</span>
* Elastic common schema was introduced to make it easier to analyze logs across applications. <span style="color:blue">[This article on the elastic.co web site](https://www.elastic.co/blog/introducing-the-elastic-common-schema)</span> is a good read and explains it in more detail.
* The <span style="color:blue">[reference for elastic common schema](https://www.elastic.co/guide/en/ecs/current/index.html)</span> provides names for many typical deployment environments, and cloud environments etc. 

``` yaml {linenos=inline,hl_lines=[15,18,19,20,22,24,31],linenostart=1}
static_resources:
  listeners:
  ...
  - address:
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          generate_request_id: true
          ...
          route_config:
          http_filters:
          ...
>          access_log:
          - name: envoy.access_loggers.file
            typed_config:
>              "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
>              path: "/dev/stdout"
>              typed_json_format: 
                "@timestamp": "%START_TIME%"
>                client.address: "%DOWNSTREAM_REMOTE_ADDRESS%"
                client.local.address: "%DOWNSTREAM_LOCAL_ADDRESS%"
>                envoy.route.name: "%ROUTE_NAME%"
                envoy.upstream.cluster: "%UPSTREAM_CLUSTER%"
                host.hostname: "%HOSTNAME%"
                http.request.body.bytes: "%BYTES_RECEIVED%"
                http.request.duration: "%DURATION%"
                http.request.headers.accept: "%REQ(ACCEPT)%"
                http.request.headers.authority: "%REQ(:AUTHORITY)%"
>                http.request.headers.id: "%REQ(X-REQUEST-ID)%"
                http.request.headers.x_forwarded_for: "%REQ(X-FORWARDED-FOR)%"
                http.request.headers.x_forwarded_proto: "%REQ(X-FORWARDED-PROTO)%"
                http.request.headers.x_b3_traceid: "%REQ(X-B3-TRACEID)%"
                http.request.headers.x_b3_parentspanid: "%REQ(X-B3-PARENTSPANID)%"
                http.request.headers.x_b3_spanid: "%REQ(X-B3-SPANID)%"
                http.request.headers.x_b3_sampled: "%REQ(X-B3-SAMPLED)%"
                http.request.method: "%REQ(:METHOD)%"
                http.response.body.bytes: "%BYTES_SENT%"
>                service.name: "envoy"
                service.version: "1.16"
                ...
  clusters:
admin:
```

The `access_log` configuration section is part of the HTTP Connection Manager Configuration and at the same nesting level as the `route_config` and `http_filters` sections. 
* We use a typed config `type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog` in version 3 of the configuration API
* The file path is set to standard out to so that log aggregation can be managed by docker. 
* The `typed_json_format` property is what allows us to create logs in JSON Lines format
* Envoy gives us template strings to insert into the configuration that it will replace with the request specific values at run time. `client.address: "%DOWNSTREAM_REMOTE_ADDRESS%"`. In these examples, we have pretty much used every available template string. The client section of elastic common schema holds everything that we know about the client application that is originating the request. We use dot notation here since nested JSON support is not available in Envoy 1.15
* Elastic Common Schema does not have standard names for everything. So we created a section for unique Envoy properties `envoy.route.name: "%ROUTE_NAME%"`
* The configuration language also has macros that enable us to do lookups. In this example we lookup the X-Request-Id header and put it in the http.request.headers.id field `http.request.headers.id: "%REQ(X-REQUEST-ID)%"`
* We can also insert static values into the logs for hardcoding labels such as which service the log entry is for etc. `service.name: "envoy"`

## Now Let's Tackle Taps
 
Taps let us capture full requests and responses. More details are for <span style="color:blue">[how the tap feature works](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/tap_filter)</span> is located in the Envoy Documentation. There are 2 ways that we can capture taps:
1. Statically configured with a local filesystem directory as the target for output. Envoy will store the request and response for each request in a separate file. The output directory must be specified with a trailing slash. Otherwise the files will not be written. The good news here is that you don't need log rotate. You can have a separate process scoop up each file, send the data to a log aggregator and then delete the file. 
1. The second method for getting taps is by sending a configuration update to the admin port with a selection criteria and then holding the connection open while waiting for taps to stream in over the network. This would obviously be the preferred approach for a production environment. 

``` yaml {linenos=inline,hl_lines=["12-22"],linenostart=1}
static_resources:
  listeners:
  ...
  - address:
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          route_config:
          http_filters:
          - name: envoy.filters.http.tap
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.tap.v3.Tap
              common_config:
                static_config:
                  match_config:
                    any_match: true
                  output_config:
                    sinks:
                      - file_per_tap:
                          path_prefix: /tmp/any/
          - name: envoy.filters.http.router
            typed_config: {}
          access_log:
          ...
  clusters:
  ...
admin:
...
```
