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


# Adding Log Aggregation to our Envoy Example

This is the 2nd blog post in a series of getting started guides for using Envoy and Open Policy Agent to authorize API requests. Later on as we start to develop authorization rules it may be handy to have all of the logs aggregated and displayed in one place for your development and troubleshooting activies. In this article we will walk through how to setup the EFK stack to pull your logs together from all of the docker containers in your local development environment. All of the [source code for this getting started example](https://github.com/helpfulBadger/envoy_getting_started/tree/master/02_front_proxy_kibana) is located on github.

## Adding EFK containers

We will be using Fluent Bit in this example because it is lite weight and simpler to deal with than Logstash or full fledged FluentD. Below is a gist with a very basic configuration and no special optimizations. Lines 9 and 30 use the `depend_on` property to cause docker to start elasticSearch first and then the other containers that depend on elasticSearch. 

{{< gist helpfulBadger 7414ff8326c13f5ab2890e4a55c9edc3 >}}

## Wiring our containers into EFK

With the log aggregation containers added to our docker-compose file, we now need to wire them into the other containers in our environment. The gist below shows a couple of small changes that we needed to make to our compose file from Getting Started Guide #1. We added the property at line 14 below which expresses our dependency on elasticSeach. Additionally, we need to wire standard out and standard error from our containers to fluentBit. This is done through the logging properties on lines 17 and 27. The driver line tells docker which log driver to use and the tag help make it more clear which container is the source of the logs. 

{{< gist helpfulBadger e701287a1847345c6997946a5b98787e >}}

# Taking things for a spin

The demonstration script spins everything up for us. Just run `./demonstrate_front_proxy.sh` to get things going:
1. It downloads and spins up all of our containers. 
1. Then it waits 30 seconds to give elasticSearch some time to get ready and some time for Kibana to know that elasticSearch is ready. 
1. A curl command sends Envoy a request to make sure the end-to-end flow is working. 
1. If that worked, proceed forward. If not wait a bit longer to make sure elasticSearch and Kibana are both ready.
1. If you are running on Mac OS X then the next step will open a browser and take you to the page to setup your Kibana index. If it doesn't work, simply open your browser and go to `http://localhost:5601/app/kibana#/management/kibana/index_pattern?_g=()` you should see something like this:     <img class="special-img-class" src="/img/2020/08/Kibana_index_pattern_1.png" /><br>
1. I simply used `log*` as my index and clicked next. Which should bring up a screen to select the timestamp field name. <img class="special-img-class" src="/img/2020/08/Kibana_index_pattern_2.png" /><br> Select `@timestamp` and click create index. 
1. You should see some field information about your newly created index. <img class="special-img-class" src="/img/2020/08/Kibana_index_pattern_3.png" /><br>
1. The script then uses the Open command to navigate to the log search interface. If it doesn't work on your operating system then simply navigate to `http://localhost:5601/app/kibana#/discover`. You should see something like this with some log results already coming in.<img class="special-img-class" src="/img/2020/08/Kibana_results_coming_in.png" /><br>
1. If you have an interest, you may want to select the `container_name` and `log` columns to make it easier to read through the debug logs and results of your testing efforts.  <img class="special-img-class" src="/img/2020/08/Kibana_select_columns.png" /><br>
1. The script sends another request through envoy and you should be able to see the logs coming into EFK. <img class="special-img-class" src="/img/2020/08/Kibana_z_Envoy_request.png" /><br>
1. The script with then take down the environment. 

In the next getting started guide, we will add in Open Policy Agent and begin experimenting with a simple authorization rule. 

