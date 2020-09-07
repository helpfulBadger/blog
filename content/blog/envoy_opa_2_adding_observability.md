+++
author = "Helpful Badger"
title = "Adding Observability Tools"
draft = false
date = 2020-09-01T15:59:54-04:00
description = "Learn how to add ElasticSearch and Kibana to your Envoy front proxy environment"
tags = [
    "Envoy",
    "Service Mesh",
    "Observability",
    "elasticSearch",
    "kibana",
]
categories = [
    "cloud",
    "service mesh",
    "Observability",
]
images  = ["img/2020/08/observatory-unsplash.jpg"]
+++

<span>Photo by <a href="https://unsplash.com/@alexeckermann?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Alex Eckermann</a> on <a href="https://unsplash.com/?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>


# Getting Started with Envoy & Open Policy Agent --- 02 ---
## Adding Log Aggregation to our Envoy Example

This is the 2nd Envory & Open Policy Agent (OPA) Getting Started Guide. Each guide is intended to explore a single Envoy or OPA feature and walk through a simple implementation. Each guide builds on the concepts explored in the previous guide with the goal of creating a very powerful authorization service by the end of the series. 

While our solution is still very simple, it is a great time to show how to make our solution observable with log aggregation. This makes it easier to think about how to scale and productionize our solution.  As we start to develop and apply authorization rules at scale it will be handy to have all of the logs aggregated and displayed in one place for development and troubleshooting activies. In this article we will walk through how to setup the EFK stack to pull your logs together from all of the docker containers in your local development environment. 

All of the source code for this getting started example is located on github. <span style="color:blue"> ------> [Envoy & OPA GS # 2](https://github.com/helpfulBadger/envoy_getting_started/tree/master/02_front_proxy_kibana) </span>

Here is a list of the Getting Started Guides that are currently available.

## Getting Started Guides

1. <span style="color:blue">[Using Envoy as a Front Proxy]({{< ref "/blog/envoy_opa_1_front_proxy.md" >}} "Learn how to set up Envoy as a front proxy with docker")</span>
1. <span style="color:blue">[Adding Observability Tools]({{< ref "/blog/envoy_opa_2_adding_observability.md" >}} "Learn how to add ElasticSearch and Kibana to your Envoy front proxy environment")</span>
1. <span style="color:blue">[Plugging Open Policy Agent into Envoy]({{< ref "/blog/envoy_opa_3_adding_open_policy_agent.md" >}} "Learn how to use Open Policy Agent with Envoy for more powerful authorization rules")</span>
1. <span style="color:blue">[Using the Open Policy Agent CLI]({{< ref "/blog/envoy_opa_4_opa_cli.md" >}} "Learn how to use Open Policy Agent Command Line Interface")</span>
1. <span style="color:blue">[JWS Signature Validation with OPA]({{< ref "/blog/envoy_opa_5_opa_jws.md" >}} "Learn how to validate JWS signatures with Open Policy Agent")</span>

# Solution Overview

The solution that we will build in this blog is shown below. We will send docker logs into an EFK stack that is also running inside docker. Each of the containers in our solution simply send logs to Standard out and / or Standard Error. No agents nor other special software is requirements are imposed on the observered applications.

<img class="special-img-class" src="/img/2020/08/Envoy-front proxy-Observability_1.svg" /><br>

## Adding EFK containers

We will be using Fluent Bit in this example because it is lite weight and simpler to deal with than Logstash or full fledged FluentD. Below is a very basic configuration with no special optimizations. Lines 18 - 49 add the EFK stack to our environment.  Lines 16 and 47 use the `depend_on` property to cause docker to start elasticSearch first and then Kiban and Fluent Bit that depend on elasticSearch. 

<img class="special-img-class" src="/img/2020/08/02_compose_step_1.png" /><br>

## Wiring our containers into EFK

With the EFK log aggregation containers added to our docker-compose file, we now need to wire them into the other containers in our environment. The changes below show a couple of small changes that we needed to make to our compose file from where we left off in Getting Started Guide #1. We added the property at line 14 below which expresses our dependency on elasticSeach. Additionally, we need to wire standard out and standard error from our containers to Fluent Bit. This is done through the logging properties on lines 17 and 27. The driver line tells docker which log driver to use and the tag help make it more clear which container is the source of the logs. 

<img class="special-img-class" src="/img/2020/08/02_compose_step_2.png" /><br>


# Taking things for a spin

The demonstration script spins everything up for us. Just run `./demonstrate_front_proxy.sh` to get things going:
1. It downloads and spins up all of our containers. 
1. Then it waits 30 seconds to give elasticSearch some time to get ready and some time for Kibana to know that elasticSearch is ready. 
1. A curl command sends Envoy a request to make sure the end-to-end flow is working. 
1. If that worked, proceed forward. If not wait a bit longer to make sure elasticSearch and Kibana are both ready.
1. If you are running on Mac OS X then the next step will open a browser and take you to the page to setup your Kibana index. If it doesn't work, simply open your browser and go to `http://localhost:5601/app/kibana#/management/kibana/index_pattern?_g=()` you should see something like this:     <img class="special-img-class" src="/img/2020/08/02_Kibana_index_pattern_1.png" /><br>
1. I simply used `log*` as my index and clicked next. Which should bring up a screen to select the timestamp field name. <img class="special-img-class" src="/img/2020/08/02_Kibana_index_pattern_2.png" /><br> Select `@timestamp` and click create index. 
1. You should see some field information about your newly created index. <img class="special-img-class" src="/img/2020/08/02_Kibana_index_pattern_3.png" /><br>
1. The script then uses the Open command to navigate to the log search interface. If it doesn't work on your operating system then simply navigate to `http://localhost:5601/app/kibana#/discover`. You should see something like this with some log results already coming in.<img class="special-img-class" src="/img/2020/08/02_Kibana_results_coming_in.png" /><br>
1. If you have an interest, you may want to select the `container_name` and `log` columns to make it easier to read through the debug logs and results of your testing efforts.  <img class="special-img-class" src="/img/2020/08/02_Kibana_select_columns.png" /><br>
1. The script sends another request through envoy and you should be able to see the logs coming into EFK. <img class="special-img-class" src="/img/2020/08/02_Kibana_z_Envoy_request.png" /><br>
1. The script with then take down the environment. 

In the next getting started guide, we will add in Open Policy Agent and begin experimenting with a simple authorization rule. 

