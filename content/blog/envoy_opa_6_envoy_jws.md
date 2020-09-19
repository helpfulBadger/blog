+++
author = "Helpful Badger"
title = "JWS Signature Validation with Envoy"
draft = false
date  = 2020-09-07T20:58:25-04:00
description = "Learn how to validate JWS signatures natively with Envoy"
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
images  = ["img/2020/08/jws-signatures-GJao3ZTX9gU-unsplash.jpg"]
+++

<span>Photo by <a href="https://unsplash.com/@cytonn_photography?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Cytonn Photography</a> on <a href="https://unsplash.com/s/photos/signature?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>


# Getting Started with Envoy & Open Policy Agent --- 06 ---
## JWS Signature Validation with Envoy

This is the 6th Envoy & Open Policy Agent Getting Started Guide. Each guide is intended to explore a single feature and walk through a simple implementation. Each guide builds on the concepts explored in the previous guide with the end goal of building a very powerful authorization service by the end of the series. 

The source code for this getting started example is located on Github. <span style="color:blue"> ------>  [Envoy & OPA GS # 6](https://github.com/helpfulBadger/envoy_getting_started/tree/master/06_envoy_validate_jws) </span>



Here is a list of the Getting Started Guides that are currently available.

## Getting Started Guides

1. <span style="color:blue">[Using Envoy as a Front Proxy]({{< ref "/blog/envoy_opa_1_front_proxy.md" >}} "Learn how to set up Envoy as a front proxy with docker")</span>
1. <span style="color:blue">[Adding Observability Tools]({{< ref "/blog/envoy_opa_2_adding_observability.md" >}} "Learn how to add ElasticSearch and Kibana to your Envoy front proxy environment")</span>
1. <span style="color:blue">[Plugging Open Policy Agent into Envoy]({{< ref "/blog/envoy_opa_3_adding_open_policy_agent.md" >}} "Learn how to use Open Policy Agent with Envoy for more powerful authorization rules")</span>
1. <span style="color:blue">[Using the Open Policy Agent CLI]({{< ref "/blog/envoy_opa_4_opa_cli.md" >}} "Learn how to use Open Policy Agent Command Line Interface")</span>
1. <span style="color:blue">[JWS Signature Validation with OPA]({{< ref "/blog/envoy_opa_5_opa_jws.md" >}} "Learn how to validate JWS signatures with Open Policy Agent")</span>
1. <span style="color:blue">[JWS Signature Validation with Envoy]({{< ref "/blog/envoy_opa_6_envoy_jws.md" >}} "Learn how to validate JWS signatures natively with Envoy")</span>
1. <span style="color:blue">[Putting It All Together with Composite Authorization]({{< ref "/blog/envoy_opa_7_app_contracts.md" >}} "Learn how to Implement Application Specific Authorization Rules")</span>
1. <span style="color:blue">[Configuring Envoy Logs Taps and Traces]({{< ref "/blog/envoy_opa_8_logs_taps_and_traces.md" >}} "Learn how to configure Envoy's access logs taps for capturing full requests & responses and traces")</span>

## Introduction

One of the HTTP filters available is the JSON Web Token filter. It is lines 14 - 27 highlighted below. You an specify any number of `providers`. For each provider the developer specifies the desired validation rules. In our case we have 3 tokens that we will be validating. For each we specify:
* The issuers and audiences that must be present. Just like with Open Policy Agent audiences, the requirement is that the specified audience is matches one of the audiences in the array. 
* The `from_headers` property tells Envoy where to find the JWS
* The `forward` property specifies if the token should be forwarded to the protected API or if the header should be removed before forwarding. If the JWS tokens can be misused by the protected API then they should be removed. 
* The `local_jwks` propery allows you to specify the JSON web keyset in the configuration. Another option is to retrieve them from an HTTP endpoint that hosts the key set. 

The configuration below shows the properties we just described on lines 22, 25 and 26.

``` yaml {linenos=inline,hl_lines=["14-27"],linenostart=1}
static_resources:
  listeners:
    - address:
      ...
      filter_chains:
          - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                ...
                route_config:
                ...
                http_filters:
                  - name: envoy.filters.http.jwt_authn
                    typed_config:
                      "@type": "type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication"
                      providers:
                        workforce_provider:
                          issuer: workforceIdentity.example.com
                          audiences:
                          - apigateway.example.com
>                          from_headers:
                          - name: "actor-token"
                            value_prefix: ""
>                          forward: true
>                          local_jwks:
                            inline_string: "..."
```

<br>

The section highlighted below is the rules section. It defines under what conditions to look for and validate a JWS. The match section defines what prefixes to look for. The slash will look for JWS tokens on every URI path. We can specify quite a few rules to determine how many and which tokens we require and under what circumstances. The <span style="color:blue">[Official JWT Auth documentation](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/filter/http/jwt_authn/v2alpha/config.proto)</span> specifies several other options on how to craft logic using and, or and any operations as well as other locations where JWS tokens can be located. There isn't as much control over what response to return to clients in the case of a failed authentication. OPA lets the developer change chose the HTTP Status code, include messages in the response body add headers etc. Envoy's built in feature simply returns an HTTP 401 Unauthorized response. 

<br>

``` yaml {linenos=inline,hl_lines=["5-13"],linenostart=28}
                        consumer_provider:
                          ...
                        gateway_provider:
                          ...
>                      rules:
                        - match:
                            prefix: /
                          requires:
                            requires_all:
                              requirements:
                                - provider_name: workforce_provider
                                - provider_name: consumer_provider
                                - provider_name: gateway_provider
                  - name: envoy.filters.http.router
                    typed_config: {}
  clusters:
  ...
admin:
...
```

To see this capability in action, simply run the `demonstrate_envoy_jws_validation.sh` script. The output is similar to the completed solution from the previous lesson. The script also dumps the Envoy logs to show the information that you have available to trouble shoot issues. Below is a screen shot of the log statements that show Envoy extracting the tokens, performing the validation and logging the result. 

<img class="special-img-class" src="/img/2020/08/06_envoy_jws_log.png" /><br>

# Conclusion

In this getting started example, we successfully validated 3 different JWS tokens in a single request and had flexibility to chose where the tokens were pulled from, how many tokens were required and under what conditions those tokens were needed. In our next getting started guide, we will use Open Policy Agent and our identity tokens to make some more sophisticated authorization decisions. 