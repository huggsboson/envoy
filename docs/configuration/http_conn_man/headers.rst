.. _config_http_conn_man_headers:

HTTP header manipulation
========================

The HTTP connection manager manipulates several HTTP headers both during decoding (when the request
is being received) as well as during encoding (when the response is being sent).

.. contents::
  :local:

.. _config_http_conn_man_headers_user-agent:

user-agent
----------

The *user-agent* header may be set by the connection manager during decoding if the
:ref:`add_user_agent <config_http_conn_man_add_user_agent>` option is enabled. The header is only
modified if it is not already set. If the connection manager does set the header, the value is
determined by the :option:`--service-cluster` command line option.

.. _config_http_conn_man_headers_server:

server
------

The *server* header will be set during encoding to the value in the :ref:`server_name
<config_http_conn_man_server_name>` option.

.. _config_http_conn_man_headers_x-client-trace-id:

x-client-trace-id
-----------------

If an external client sets this header, Envoy will join the provided trace ID with the internally
generated :ref:`config_http_conn_man_headers_x-request-id`. x-client-trace-id needs to be globally
unique and generating a uuid4 is recommended. If this header is set, it has similar effect to
:ref:`config_http_conn_man_headers_x-envoy-force-trace`. See the :ref:`tracing.client_enabled
<config_http_conn_man_runtime_client_enabled>` runtime configuration setting.

.. _config_http_conn_man_headers_downstream-service-cluster:

x-envoy-downstream-service-cluster
----------------------------------

Internal services often want to know which service is calling them. This header is cleaned from
external requests, but for internal requests will contain the service cluster of the caller. Note
that in the current implementation, this should be considered a hint as it is set by the caller and
could be easily spoofed by any internal entity. In the future Envoy will support a mutual
authentication TLS mesh which will make this header fully secure. Like *user-agent*, the value
is determined by the :option:`--service-cluster` command line option. In order to enable this
feature you need to set the :ref:`user_agent <config_http_conn_man_add_user_agent>` option to true.

.. _config_http_conn_man_headers_x-envoy-external-address:

x-envoy-external-address
------------------------

It is a common case where a service wants to perform analytics based on the client IP address. Per
the lengthy discussion on :ref:`XFF <config_http_conn_man_headers_x-forwarded-for>`, this can get
quite complicated. A proper implementation involves forwarding XFF, and then choosing the first non
RFC1918 address *from the right*. Since this is such a common occurrence, Envoy simplifies this by
setting *x-envoy-external-address* during decoding if and only if the request ingresses externally
(i.e., it's from an external client). *x-envoy-external-address* is not set or overwritten for
internal requests. This header can be safely forwarded between internal services for analytics
purposes without having to deal with the complexities of XFF.

.. _config_http_conn_man_headers_x-envoy-force-trace:

x-envoy-force-trace
-------------------

If an internal request sets this header, Envoy will modify the generated
:ref:`config_http_conn_man_headers_x-request-id` such that it forces traces to be collected.
This also forces :ref:`config_http_conn_man_headers_x-request-id` to be returned in the response
headers. If this request ID is then propagated to other hosts, traces will also be collected on
those hosts which will provide a consistent trace for an entire request flow. See the
:ref:`tracing.global_enabled <config_http_conn_man_runtime_global_enabled>` and
:ref:`tracing.random_sampling <config_http_conn_man_runtime_random_sampling>` runtime
configuration settings.

.. _config_http_conn_man_headers_x-envoy-internal:

x-envoy-internal
----------------

It is a common case where a service wants to know whether a request is internal origin or not. Envoy
uses :ref:`XFF <config_http_conn_man_headers_x-forwarded-for>` to determine this and then will set
the header value to *true*.

This is a convenience to avoid having to parse and understand XFF.

.. _config_http_conn_man_headers_x-forwarded-for:

x-forwarded-for
---------------

*x-forwarded-for* (XFF) is a standard proxy header which indicates the IP addresses that a request has
flowed through on its way from the client to the server. A compliant proxy will *append* the IP
address of the nearest client to the XFF list before proxying the request. Some examples of XFF are:

1. ``x-forwarded-for: 50.0.0.1`` (single client)
2. ``x-forwarded-for: 50.0.0.1, 40.0.0.1`` (external proxy hop)
3. ``x-forwarded-for: 50.0.0.1, 10.0.0.1`` (internal proxy hop)

Envoy will only append to XFF if the :ref:`use_remote_address
<config_http_conn_man_use_remote_address>` HTTP connection manager option is set to true. This means
that if *use_remote_address* is false, the connection manager operates in a transparent mode where
it does not modify XFF. This is needed for certain types of mesh deployments depending on whether
the Envoy in question is an edge node or an internal service node.

Envoy uses the final XFF contents to determine whether a request originated externally or
internally. This influences whether the :ref:`config_http_conn_man_headers_x-envoy-internal` header
is set.

A few very important notes about XFF:

1. Since IP addresses are appended to XFF, only the last address (furthest to the right) can be
   trusted. More specifically, the first external (non RFC1918) address from *the right* is the only
   trustable addresses. Anything to the left of that can be spoofed. To make this easier to deal
   with for analytics, etc., front Envoy will also set the
   :ref:`config_http_conn_man_headers_x-envoy-external-address` header.
2. XFF is what Envoy uses to determine whether a request is internal origin or external origin. It
   does this by checking to see if XFF contains a *single* IP address which is an RFC1918 address.

   * **NOTE**: If an internal service proxies an external request to another internal service, and
     includes the original XFF header, Envoy will append to it on egress if
     :ref:`use_remote_address <config_http_conn_man_use_remote_address>` is set. This will cause
     the other side to think the request is external. Generally, this is what is intended if XFF is
     being forwarded. If it is not intended, do not forward XFF, and forward
     :ref:`config_http_conn_man_headers_x-envoy-internal` instead.
   * **NOTE**: If an internal service call is forwarded to another internal service (preserving XFF),
     Envoy will not consider it internal. This is a known "bug" due to the simplification of how
     XFF is parsed to determine if a request is internal. In this scenario, do not forward XFF and
     allow Envoy to generate a new one with a single internal origin IP.

x-forwarded-proto
-----------------

It is a common case where a service wants to know what the originating protocol (HTTP or HTTPS) was
of the connection terminated by front/edge Envoy. *x-forwarded-proto* contains this information. It
will be set to either *http* or *https*.

.. _config_http_conn_man_headers_x-request-id:

x-request-id
------------

The *x-request-id* header is used by Envoy to uniquely identify a request as well as perform stable
access logging and tracing. Envoy will generate an *x-request-id* header for all external origin
requests (the header is sanitized). It will also generate an *x-request-id* header for internal
requests that do not already have one. This means that *x-request-id* can and should be propagated
between client applications in order to have stable IDs across the entire mesh. Due to the out of
process architecture of Envoy, the header can not be automatically forwarded by Envoy itself. This
is one of the few areas where a thin client library is needed to perform this duty. How that is done
is out of scope for this documentation. If *x-request-id* is propagated across all hosts, the
following features are available:

* Stable :ref:`access logging <config_http_conn_man_access_log>` via the
  :ref:`runtime filter<config_http_con_manager_access_log_filters_runtime>`.
* Stable tracing when performing random sampling via the :ref:`tracing.random_sampling
  <config_http_conn_man_runtime_random_sampling>` runtime setting or via forced tracing using the
  :ref:`config_http_conn_man_headers_x-envoy-force-trace` and
  :ref:`config_http_conn_man_headers_x-client-trace-id` headers.
