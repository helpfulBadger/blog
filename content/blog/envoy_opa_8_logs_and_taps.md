+++
author = "Helpful Badger"
title = "Configuring Envoy Logs and Taps"
draft = false
date  = 2020-09-10T10:36:36-04:00
description = "Learn how to configure Envoy's access logs and taps for capturing full requests & responses"
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
images  = ["img/2020/08/tap-dPr7uyrFMXo-unsplash.jpg"]
+++


<span>Photo by <a href="https://unsplash.com/@kazukyakayashi?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Kazuky Akayashi</a> on <a href="https://unsplash.com/@kazukyakayashi?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>


# Getting Started with Envoy & Open Policy Agent --- 08 ---
## Learn how to configure Envoy's access logs and taps for capturing full requests & responses

This is the 7th Envoy & Open Policy Agent Getting Started Guide. Each guide is intended to explore a single feature and walk through a simple implementation. Each guide builds on the concepts explored in the previous guide with the end goal of building a very powerful authorization service by the end of the series. 

The source code for this getting started example is located on Github. <span style="color:blue"> ------>  [Envoy & OPA GS # 8](https://github.com/helpfulBadger/envoy_getting_started/tree/master/08_log_config_taps) </span>


Here is a list of the Getting Started Guides that are currently available.

## Getting Started Guides

1. <span style="color:blue">[Using Envoy as a Front Proxy]({{< ref "/blog/envoy_opa_1_front_proxy.md" >}} "Learn how to set up Envoy as a front proxy with docker")</span>
1. <span style="color:blue">[Adding Observability Tools]({{< ref "/blog/envoy_opa_2_adding_observability.md" >}} "Learn how to add ElasticSearch and Kibana to your Envoy front proxy environment")</span>
1. <span style="color:blue">[Plugging Open Policy Agent into Envoy]({{< ref "/blog/envoy_opa_3_adding_open_policy_agent.md" >}} "Learn how to use Open Policy Agent with Envoy for more powerful authorization rules")</span>
1. <span style="color:blue">[Using the Open Policy Agent CLI]({{< ref "/blog/envoy_opa_4_opa_cli.md" >}} "Learn how to use Open Policy Agent Command Line Interface")</span>
1. <span style="color:blue">[JWS Signature Validation with OPA]({{< ref "/blog/envoy_opa_5_opa_jws.md" >}} "Learn how to validate JWS signatures with Open Policy Agent")</span>
1. <span style="color:blue">[JWS Signature Validation with Envoy]({{< ref "/blog/envoy_opa_6_envoy_jws.md" >}} "Learn how to validate JWS signatures natively with Envoy")</span>
1. <span style="color:blue">[Putting It All Together with Composite Authorization]({{< ref "/blog/envoy_opa_7_app_contracts.md" >}} "Learn how to Implement Application Specific Authorization Rules")</span>
1. <span style="color:blue">[Configuring Envoy Logs and Taps]({{< ref "/blog/envoy_opa_8_logs_and_taps.md" >}} "Learn how to configure Envoy's access logs and taps for capturing full requests & responses")</span>

# Draft Article that is Under construction 
## Introduction

