<!-- Copy and paste the converted output. -->

<!-----
NEW: Check the "Suppress top comment" option to remove this info from the output.

Conversion time: 7.009 seconds.


Using this Markdown file:

1. Paste this output into your source file.
2. See the notes and action items below regarding this conversion run.
3. Check the rendered output (headings, lists, code blocks, tables) for proper
   formatting and use a linkchecker before you publish this page.

Conversion notes:

* Docs to Markdown version 1.0β29
* Mon Aug 24 2020 13:28:51 GMT-0700 (PDT)
* Source doc: xDS transport next steps
* Tables are currently converted to HTML tables.

WARNING:
Inline drawings not supported: look for ">>>>>  gd2md-html alert:  inline drawings..." in output.


ERROR:
undefined internal link to this URL: "#heading=h.85gibjowrc4a".link text: udpa:// URI encoding
?Did you generate a TOC?


ERROR:
undefined internal link to this URL: "#heading=h.n1f49q8is712".link text: alternatives considered
?Did you generate a TOC?


ERROR:
undefined internal link to this URL: "#heading=h.wdyjyh724jol".link text: global context parameters
?Did you generate a TOC?


ERROR:
undefined internal link to this URL: "#heading=h.wdyjyh724jol".link text: per-node context parameters
?Did you generate a TOC?


ERROR:
undefined internal link to this URL: "#heading=h.hkc4btyff116".link text: delta and SotW examples
?Did you generate a TOC?


ERROR:
undefined internal link to this URL: "#heading=h.nnqgti1ico3k".link text: effective resource names
?Did you generate a TOC?


ERROR:
undefined internal link to this URL: "#heading=h.2rwk417x2h8".link text: singleton resource requests
?Did you generate a TOC?


ERROR:
undefined internal link to this URL: "#heading=h.2rwk417x2h8".link text: singleton resource request
?Did you generate a TOC?


ERROR:
undefined internal link to this URL: "#heading=h.1xfr9pprvqpe".link text: above
?Did you generate a TOC?


ERROR:
undefined internal link to this URL: "#heading=h.1xfr9pprvqpe".link text: above
?Did you generate a TOC?


ERROR:
undefined internal link to this URL: "#heading=h.4iqy2glbz1r1".link text: method of ConfigSource resolution
?Did you generate a TOC?


WARNING:
You have 12 H1 headings. You may want to use the "H1 -> H2" option to demote all headings by one level.

----->


<p style="color: red; font-weight: bold">>>>>>  gd2md-html alert:  ERRORs: 11; WARNINGs: 2; ALERTS: 15.</p>
<ul style="color: red; font-weight: bold"><li>See top comment block for details on ERRORs and WARNINGs. <li>In the converted Markdown or HTML, search for inline alerts that start with >>>>>  gd2md-html alert:  for specific instances that need correction.</ul>

<p style="color: red; font-weight: bold">Links to alert messages:</p><a href="#gdcalert1">alert1</a>
<a href="#gdcalert2">alert2</a>
<a href="#gdcalert3">alert3</a>
<a href="#gdcalert4">alert4</a>
<a href="#gdcalert5">alert5</a>
<a href="#gdcalert6">alert6</a>
<a href="#gdcalert7">alert7</a>
<a href="#gdcalert8">alert8</a>
<a href="#gdcalert9">alert9</a>
<a href="#gdcalert10">alert10</a>
<a href="#gdcalert11">alert11</a>
<a href="#gdcalert12">alert12</a>
<a href="#gdcalert13">alert13</a>
<a href="#gdcalert14">alert14</a>
<a href="#gdcalert15">alert15</a>

<p style="color: red; font-weight: bold">>>>>> PLEASE check and correct alert issues and delete this message and the inline alerts.<hr></p>



# xDS transport next steps

**Authors:** [htuch@google.com](mailto:htuch@google.com), [lryan@google.com](mailto:lryan@google.com), [roth@google.com](mailto:roth@google.com), [costin@google.com](mailto:costin@google.com), [mattklein123@gmail.com](mailto:mattklein123@gmail.com) 

**Last modified:** 6/29/2020

**Status:** Plan-of-record

**This document is publicly shared with the Envoy/Istio community**


# Overview

This document provides design discussion on the direction forward for the [xDS transport protocol.](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol) Since the introduction of v2 xDS in Envoy, the transport protocol has provided a generic pub-sub mechanism between client (Envoy) and management server (xDS control plane). The resources have largely[^1] been opaque, identified by a type URL, opaque resource name string and opaque version string.

While this model has been successful in Envoy’s v2 and v3 xDS APIs, limitations have emerged as the use of xDS has grown to encompass additional clients (namely gRPC service mesh), new xDS resource types have been introduced and the complexity of control plane topology has increased. We describe known limitations below and provide a solution for implementation.

This document’s goal is to map previous design discussions around [UDPA-TP](https://docs.google.com/document/d/1eubmNM2Kynzf7Rpms4nncvTSL6NnANTrpHNWzeIBoZw/edit#), Istio xDS proxy and federation to the concrete details of implementations in the v3 xDS proto3 APIs in a manner that allows for gradual adoption by operators and xDS client developers. Most ideas and concepts are borrowed from these original sources.

A [summary document](https://docs.google.com/document/d/1m5_Q9LUlzvDdImP0jqh1lSTrLMldsApTQX6ibbd3i7c/edit?ts=5ec2fd8a#heading=h.w8hw2zbv3jwl) provides a guide to the objectives and calls out the key points in the proposal below.


# Background

**TL;DR** This section exists as reference, feel free to skip if you are familiar with v3 xDS transport messages.

We provide some fragments of the v3 xDS discovery messages that play a role in the discussion below. We elide details such as response nonce, error handling and other fields that are not relevant to the proposal. This section is a bit verbose, but helps make concrete the existing playing field.


## ConfigSource

Envoy (and other xDS clients) learn how to address a management server via `ConfigSource` messages. These may be present in the bootstrap for root-level resources such as LDS or CDS. Or, a `ConfigSource` message might be embedded in a delivered resource, such as a `Cluster` delivered via CDS from server X, pointing to where the configuration of its dependencies may be found, e.g. `ClusterLoadAssignment` M via EDS from server Y. The `ConfigSource` describes a single hop transport configuration for xDS. Some details deriving from the `ConfigSource` are important when thinking about naming; the `GrpcService` provides an effective authority today for resources, the resource API version influences the type URL requested.


<table>
  <tr>
   <td rowspan="62" colspan="2" ><code>message ApiConfigSource {</code>
<p>
<code>  … REST vs. gRPC, transport API versioning fields ...</code>
<p>
<code> // Multiple gRPC services can be provided for GRPC. If > 1 service is defined,</code>
<p>
<code> // services will be cycled through if any kind of failure occurs.</code>
<p>
<code> repeated GrpcService grpc_services = 4;</code>
<p>
<code>… timeouts, rate limiting, misc. ….</code>
<p>
<code>}</code>
   </td>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
</table>



<table>
  <tr>
   <td rowspan="41" colspan="2" ><code>message ConfigSource {</code>
<p>
<code> oneof config_source_specifier {</code>
<p>
<code>   // Path on the filesystem to source and watch for configuration updates.</code>
<p>
<code>    string path = 1;</code>
<p>
<code>   // API configuration source.</code>
<p>
<code>   ApiConfigSource api_config_source = 2;</code>
<p>
<code>   // When set, ADS will be used to fetch resources. The ADS API configuration</code>
<p>
<code>   // source in the bootstrap configuration is used.</code>
<p>
<code>   AggregatedConfigSource ads = 3;</code>
<p>
      …
<p>
    }
<p>
     <code>… fetch timeout, …</code>
<p>
<code>   // API version for xDS resources. This implies the type URLs that the client</code>
<p>
<code>   // will request for resources and the resource type that the client will in</code>
<p>
<code>   // turn expect to be delivered.</code>
<p>
<code>   ApiVersion resource_api_version = 6</code>
<p>
<code> }</code>
   </td>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
</table>



## Node

Below we provide the entire `Node` message for reference. The Node message describes the client making a discovery request, providing identity, locality, client capabilities, client version, known extensions, etc. While it would be desirable for a resource’s name and contents to be immutable and client independent, this is not the case today in some deployments. For the discussion below, there are two fields, `metadata` and `listening_addresses` that are particularly relevant to resource naming, since they have been used by clients and servers to provide mechanisms for qualifying resources (e.g. listeners) during discovery and mutating/generating resources on a per-client basis. However, any field below may figure into the contents that an xDS server will return for a given discovery request, e.g. client feature capabilities, user agent.


<table>
  <tr>
   <td rowspan="68" colspan="2" ><code>message Node {</code>
<p>
<code> // An opaque node identifier for the Envoy node. </code>
<p>
<code> string id = 1;</code>
<p>
<code> // Defines the local service cluster name where Envoy is running.</code>
<p>
<code> string cluster = 2;</code>
<p>
<code> // Opaque metadata extending the node identifier. Envoy will pass this</code>
<p>
<code> // directly to the management server.</code>
<p>
<code> google.protobuf.Struct metadata = 3;</code>
<p>
<code> // Locality specifying where the Envoy instance is running.</code>
<p>
<code> Locality locality = 4;</code>
<p>
<code> // Free-form string that identifies the entity requesting config.</code>
<p>
<code> // E.g. "envoy" or "grpc"</code>
<p>
<code> string user_agent_name = 6;</code>
<p>
<code> oneof user_agent_version_type {</code>
<p>
<code>   // Free-form string that identifies the version of the entity requesting config.</code>
<p>
<code>   // E.g. "1.12.2" or "abcd1234", or "SpecialEnvoyBuild"</code>
<p>
<code>   string user_agent_version = 7;</code>
<p>
<code>   // Structured version of the entity requesting config.</code>
<p>
<code>   BuildVersion user_agent_build_version = 8;</code>
<p>
<code> }</code>
<p>
<code> // List of extensions and their versions supported by the node.</code>
<p>
<code> repeated Extension extensions = 9;</code>
<p>
<code> // Client feature support list. These are well known features described</code>
<p>
<code> // in the Envoy API repository for a given major version of an API. Client features</code>
<p>
<code> // use reverse DNS naming scheme, for example `com.acme.feature`.</code>
<p>
<code> // See :ref:`the list of features &lt;client_features>` that xDS client may</code>
<p>
<code> // support.</code>
<p>
<code> repeated string client_features = 10;</code>
<p>
<code> // Known listening ports on the node as a generic hint to the management server</code>
<p>
<code> // for filtering :ref:`listeners &lt;config_listeners>` to be returned. For example,</code>
<p>
<code> // if there is a listener bound to port 80, the list can optionally contain the</code>
<p>
<code> // SocketAddress `(0.0.0.0,80)`. The field is optional and just a hint.</code>
<p>
<code> repeated Address listening_addresses = 11;</code>
<p>
<code>}</code>
   </td>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
</table>



## State-of-the-World (SotW) discovery

The xDS state-of-the-world (SotW) discovery protocol involves the client sending a request to the server and receiving back either the requested resources, or in the case of wildcard requests like LDS/CDS, receiving the entire set of resources for the given type.

Several aspects of the discovery request and response are relevant to naming:



*   `DiscoveryRequest`s contain a `Node` message, identifying the client. Some servers use this to generate or modify the resource returned for a given name.
*   The authority is implied by the `ConfigSource` in Envoy responsible for the DiscoveryRequest.
*   The client provides either a set of resource names, which are today opaque identifiers, or an empty list, indicating a wildcard query.
*   A type URL is provided indicating the set of resources being queried. Each resource has a type, provided by its canonical proto3 representation. The [protobuf type UR](https://developers.google.com/protocol-buffers/docs/reference/csharp/class/google/protobuf/well-known-types/any)L is then used to represent the type of resource in a request, e.g. `type.googleapis.com/envoy.config.route.v3.RouteConfiguration.`

The` DiscoveryResponse `contains a version (applicable to all resources returned) and a set of opaque resources represented by `google.protobuf.Any`. Deriving the name of the resource requires per-resource type handling by the client, all resources have some form of name field. SotW response resource payloads do not support generic name discovery.


<table>
  <tr>
   <td rowspan="72" colspan="2" ><code>message DiscoveryRequest {</code>
<p>
<code> // The version_info provided in the request messages will be the version_info</code>
<p>
<code> // received with the most recent successfully processed response or empty on</code>
<p>
<code> // the first request. It is expected that no new request is sent after a</code>
<p>
<code> // response is received until the Envoy instance is ready to ACK/NACK the new</code>
<p>
<code> // configuration. ACK/NACK takes place by returning the new API config version</code>
<p>
<code> // as applied or the previous API config version respectively. Each type_url</code>
<p>
<code> // (see below) has an independent version associated with it.</code>
<p>
<code> string version_info = 1;</code>
<p>
<code> // The node making the request.</code>
<p>
<code> config.core.v3.Node node = 2;</code>
<p>
<code> // List of resources to subscribe to, e.g. list of cluster names or a route</code>
<p>
<code> // configuration name. If this is empty, all resources for the API are</code>
<p>
<code> // returned. LDS/CDS may have empty resource_names, which will cause all</code>
<p>
<code> // resources for the Envoy instance to be returned. The LDS and CDS responses</code>
<p>
<code> // will then imply a number of resources that need to be fetched via EDS/RDS,</code>
<p>
<code> // which will be explicitly enumerated in resource_names.</code>
<p>
<code> repeated string resource_names = 3;</code>
<p>
<code> // Type of the resource that is being requested, e.g.</code>
<p>
<code> // "type.googleapis.com/envoy.api.v2.ClusterLoadAssignment". This is implicit</code>
<p>
<code> // in requests made via singleton xDS APIs such as CDS, LDS, etc. but is</code>
<p>
<code> // required for ADS.</code>
<p>
<code> string type_url = 4;</code>
<p>
<code> … response nonce, error detail ...</code>
<p>
<code>}</code>
<p>
<code>message DiscoveryResponse {</code>
<p>
<code> // The version of the response data.</code>
<p>
<code> string version_info = 1;</code>
<p>
<code> // The response resources. These resources are typed and depend on the API being called.</code>
<p>
<code> repeated google.protobuf.Any resources = 2;</code>
<p>
<code> // Type URL for resources. Identifies the xDS API when muxing over ADS.</code>
<p>
<code> // Must be consistent with the type_url in the 'resources' repeated Any (if non-empty).</code>
<p>
<code> string type_url = 4;</code>
<p>
<code>}</code>
   </td>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
  <tr>
  </tr>
</table>



## Delta discovery

The xDS delta discovery protocol aims to make resource discovery more efficient by only communicating deltas for xDS types such as LDS/CDS. Resources continue to be named with an opaque string, but there are some key differences:



*   Delta xDS introduces per-resource versions in responses.
*   Resources are wrapped in a `Resource` message. This provides direct access to the resource name, supporting generic resource name handling, e.g. by intermediate relays.
*   Resource names may have aliases. These were introduced to support VHDS, where an on-demand resource representing `{foo.com, bar.com}` might be fetched via a resource name referring to either `foo.com` or `bar.com`. 

<table>
  <tr>
   <td rowspan="2" colspan="2" >
<code>message DeltaDiscoveryRequest {</code>
<p>
<code> // The node making the request.</code>
<p>
<code> config.core.v3.Node node = 1;</code>
<p>
<code> // Type of the resource that is being requested, e.g.</code>
<p>
<code> // "type.googleapis.com/envoy.api.v2.ClusterLoadAssignment".</code>
<p>
<code> string type_url = 2;</code>
<p>
<code> // A list of Resource names to add to the list of tracked resources.</code>
<p>
<code> repeated string resource_names_subscribe = 3;</code>
<p>
<code> // A list of Resource names to remove from the list of tracked resources.</code>
<p>
<code> repeated string resource_names_unsubscribe = 4;</code>
<p>
<code> // Informs the server of the versions of the resources the xDS client knows of, to enable the</code>
<p>
<code> // client to continue the same logical xDS session even in the face of gRPC stream reconnection.</code>
<p>
<code> // The map's keys are names of xDS resources known to the xDS client.</code>
<p>
<code> // The map's values are opaque resource versions.</code>
<p>
<code> map&lt;string, string> initial_resource_versions = 5;</code>
<p>
<code> … nonce, error detail ...</code>
<p>
<code>}</code>
<p>
<code>message DeltaDiscoveryResponse {</code>
<p>
<code> // The version of the response data (used for debugging).</code>
<p>
<code> string system_version_info = 1;</code>
<p>
<code> // The response resources. These are typed resources, whose types must match</code>
<p>
<code> // the type_url field.</code>
<p>
<code> repeated Resource resources = 2;</code>
<p>
<code> // Type URL for resources. Identifies the xDS API when muxing over ADS.</code>
<p>
<code> // Must be consistent with the type_url in the Any within 'resources' if 'resources' is non-empty.</code>
<p>
<code> string type_url = 4;</code>
<p>
<code> // Resources names of resources that have be deleted and to be removed from the xDS Client.</code>
<p>
<code> // Removed resources for missing resources can be ignored.</code>
<p>
<code> repeated string removed_resources = 6;</code>
<p>
<code> … nonce ...</code>
<p>
<code>}</code>
<p>
<code>message Resource {</code>
<p>
<code> // The resource's name, to distinguish it from others of the same type of resource.</code>
<p>
<code> string name = 3;</code>
<p>
<code> // The aliases are a list of other names that this resource can go by.</code>
<p>
<code> repeated string aliases = 4;</code>
<p>
<code> // The resource level version. It allows xDS to track the state of individual</code>
<p>
<code> // resources.</code>
<p>
<code> string version = 1;</code>
<p>
<code> // The resource being tracked.</code>
<p>
<code> google.protobuf.Any resource = 2;</code>
<p>
<code>}</code>
   </td>
  </tr>
  <tr>
  </tr>
</table>



# Control plane model

Control plane topologies for xDS have been relatively simple single hop, direct connection from 1 or more xDS clients to a management server:



<p id="gdcalert1" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline drawings not supported directly from Docs. You may want to copy the inline drawing to a standalone drawing and export by reference. See <a href="https://github.com/evbacher/gd2md-html/wiki/Google-Drawings-by-reference">Google Drawings by reference</a> for details. The img URL below is a placeholder. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert2">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![drawing](https://docs.google.com/drawings/d/12345/export/png)

Going forward, we anticipate more general xDS control plane topologies, as Envoy and other xDS clients are deployed in more complex environments, with:



*   Hybrid on-premise/cloud clients and control planes
*   Federated control planes
*   Mobile/edge, with O(millions of xDS clients)
*   xDS caching
*   Hierarchical control planes with xDS used for both intra control plane communication as well as client.



<p id="gdcalert2" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline drawings not supported directly from Docs. You may want to copy the inline drawing to a standalone drawing and export by reference. See <a href="https://github.com/evbacher/gd2md-html/wiki/Google-Drawings-by-reference">Google Drawings by reference</a> for details. The img URL below is a placeholder. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert3">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![drawing](https://docs.google.com/drawings/d/12345/export/png)



<p id="gdcalert3" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline drawings not supported directly from Docs. You may want to copy the inline drawing to a standalone drawing and export by reference. See <a href="https://github.com/evbacher/gd2md-html/wiki/Google-Drawings-by-reference">Google Drawings by reference</a> for details. The img URL below is a placeholder. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert4">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![drawing](https://docs.google.com/drawings/d/12345/export/png)



<p id="gdcalert4" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline drawings not supported directly from Docs. You may want to copy the inline drawing to a standalone drawing and export by reference. See <a href="https://github.com/evbacher/gd2md-html/wiki/Google-Drawings-by-reference">Google Drawings by reference</a> for details. The img URL below is a placeholder. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert5">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![drawing](https://docs.google.com/drawings/d/12345/export/png)


# Problem statement

The xDS transport protocol has a number of known issues, enumerated in the next section, which are becoming more acute as the number of resource types grow, control plane topologies become more sophisticated and we have increased client diversity.

We want to develop a **structured naming convention for xDS resources** and some **backwards compatible tweaks** to the xDS transport that fix the following known issues or missing capabilities.


# Known issues


## In-scope


### Collections

Borrowing some terminology from [REST resource naming](https://restfulapi.net/resource-naming/), xDS supports both singleton and collection discovery today. For xDS types like RDS/EDS/SDS, singleton resource names are provided explicitly in the discovery request, and returned. For other xDS types, e.g. CDS/LDS, a wildcard subscription is performed by Envoy[^2]. A collection is requested using some combination of type URL and `Node` information (often metadata)[^3].

The limitations of this for collections become apparent when you consider what might be needed to select a subset of listeners or clusters via LDS/CDS. The type URL must remain constant, so the Node metadata is generally used to encode subsetting information. This is a conflation of client identity and selection criteria.

Reasons to subset the universe of resources for a given type include wanting to differentiate inbound/outbound listeners in a service mesh topology, wanting to partition listeners between different nodes in a multi-tenant proxy and wanting to distinguish based on node function in a service mesh (e.g. ingress/egress).

Collections require a form of identity. We have considered introducing [namespaces](https://github.com/envoyproxy/envoy/pull/10689) as a mechanism to address this, where a discovery request would include an additional opaque label to support subsetting, but a more general approach would be to introduce a set of key/value attributes to select resources in the request (at the expense of increased complexity).

Collections with delta support are also necessary for scalable endpoint support in the [LEDS proposal](https://github.com/envoyproxy/envoy/issues/10373).


### Resource immutability

Another problematic example of the use of `Node` information today is that xDS does not have an immutable resource name to resource value mapping. Instead, management servers combine `Node` information in generating or modifying resource values. Two clients may see distinct resources values for the same resource name.

This impacts cacheability; the `Node` message becomes part of the cache key (and must be distributed across [xDS relays](https://docs.google.com/document/d/1X9fFzqBZzrSmx2d0NkmjDW2tc8ysbLQS9LaRQRxdJZU/edit?disco=AAAAGQ_84vU&ts=5e61532c&usp_dm=false)). This conflates client identity with resource identity. Some of the `Node` message is only relevant on a single hop to the management server, but in more complex topologies other attributes (e.g. metadata) are what really matter when trying to fetch a resource. Ideally we divorce the node identity from the resource naming and permit control planes to serve equivalence classes of clients. This also impacts federation support, since distinct control plane entities need to know how to handle node identities globally.

Historically, the requirements around resource [idempotence](https://restfulapi.net/idempotent-rest-apis/) have been only implicitly stated. It is assumed that the same request for a resource will produce the same value (subject to versioning). A next generation naming scheme should be more direct in this requirement to ensure that caching and federation of resources works in a predictable way. If this is not done, it will be very hard to debug or predict the function of distributed control planes where a resource can be cached or delivered on more than one path.


### Per-xDS type resource name semantics

While most xDS resources today permit opaque resource naming at the discretion of the control plane, some xDS resources, namely VHDS, have specific resource name structure. VHDS requires the [form](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/vhds.html?highlight=vhds#virtual-host-resource-naming-convention) of `&lt;route configuration name>/&lt;host entry>`. This permits the management server to lookup the missing virtual host with the host entry inside the specified `RouteConfiguration` resource name. In this scheme, the route configuration name is acting as a namespace for the virtual host name.

It’s likely other resource types in the future will require some structure. Rather than living with an arbitrary mix of opaque and structured forms, an ideal resource naming scheme would permit structure while leaving some freedom to the control plane to continue to use opaque naming if desired.


### Flat representation

Today, resource identifies are a hodge podge of resource name, `Node` information, authority from `ConfigSource`, type URL and version. When communicating between humans or systems that don’t work with the pertinent proto3 messages, there is no standard format for concisely encoding this information.

URIs provide a powerful standards compliant way of specifying this information. Future xDS resource naming schemes will benefit from having a canonical URI representation (in addition to canonical proto3 definitions).


### Implicit resource payload naming

While in delta discovery we have resource names explicitly provided in the `Resource` messages in a `DeltaDiscoveryResponse`, this is not the case for SotW resource payloads. This means that xDS proxies, caches and other intermediaries have to have per-resource type awareness and deserialization to be able to know what resource payload is being delivered.

We need to provide a better solution for SotW resource naming in `DiscoveryResponses`, likely deprecating the existing `repeated Any` and replacing with `repeated Resource` in v3/v4 xDS.


### Aliasing

VHDS introduced aliasing to xDS, since the same virtual host resource may be known by multiple names, e.g. `routeConfigA/foo.com, routeConfigA/bar.com`. This mechanism is not widely used yet elsewhere in xDS and it would be ideal to replace it with something more generic.

An attractive alternative, that is also useful for other purposes in the federation discussion below, will be to introduce redirect capabilities to xDS. When querying for either `routeConfigA/foo.com` or `routeConfigA/bar.com`, the management server might issue a redirect causing the xDS client to fetch `someCanonicalVirtualHost` resource. This simplifies the requirements of the xDS control plane by removing conceptual complexity.


### Federation

Federation will require Envoy addressing multiple management servers (in the general case) to support disaggregation of resources across multiple authorities, delegation of authority, replica failover and hybrid topologies. Different authorities and hops in the control plane may exchange information with each other via xDS. Example concrete use cases:



    *   Failover from a Control Plane as-a-service (CPaaS) to on-premise control plane on network partition.
    *   On-premise authoring and publication of xDS fragments to a CPaaS.
    *   Disaggregating responsibility for LB APIs between on-premise and CPaaS. For example splitting the health check and load assignment responsibilities.
    *   Disaggregating a CPaaS internally. CPaaS may benefit from having multiple independent xDS providers with limited coupling combine and aggregate xDS.
    *   Bridging independent service meshes with independent control planes.

There are many gnarly issues to tackle in xDS to make this happen, including determining how resources compose, the granularity of resources, how authority is delegated, trust management and so on. We do not aim to tackle all of these below, instead restricting ourselves to providing a naming scheme and transport protocol that is flexible enough to grow to encompass future needs here.

xDS currently uses `ConfigSource` to specify authority. This works well when communicating with one or more management servers directly, each of which is authoritative for the resources they serve. This works less well when a management server is providing resources from multiple authorities, or the same resource is available from multiple management servers (e.g. failover replicas). Another scenario to consider is when a resource requested from some `ConfigSource` is not directly available, but instead the management server wishes to delegate (redirect) the client to another resource.


## Out-of-scope & future work

We do not plan to solve problems in this section in this proposal due to the potential complexity and lack of immediate motivation, but believe the proposal will be extendable to support these in the future.


### RTT optimization

In the proposal below, multiple round trips are required to fetch collection resources. The proposal allows for single round-trip optimization but does not specify details. We defer this to a followup document specifically addressing this concern. Optimizations of this nature are complex and should be driven by use cases that demand them.


### Minor/patch version negotiation

Version negotiation is discussed in the xDS minor/patch version [proposal](https://docs.google.com/document/d/1afQ9wthJxofwOMCAle8qF-ckEfNWUuWtxbmdhxji0sI/edit). We believe it should be possible to extend the proposal below to satisfy requirements around client version exchange, either through the use of context parameters with constraints or explicit first class versioning in the discovery request.  


### Versioning and consistency

Envoy’s fundamental consistency model is eventual, which has been reflected in xDS. The latest version of a resource is consumed at all times by Envoy as determined by the management server. With a more general xDS use case and layers of caching, relay and delegated authority, it might be necessary to provide minimum resource versions, resource version ranges or some other indicator in xDS resource requests. It’s conceivable that this version information becomes part of the name.


### Implicit resource dependency

Both SotW and delta discovery do not have first class representation of inter-resource dependencies. For example, a `Cluster` resource may depend on a `ClusterLoadAssignment` resource, but any entity handling the resource requires knowledge of the `Cluster` proto to extract this information. This is not a huge issue for simple topologies, but more complex multi-hop topologies might require (or at the very least benefit from) the ability to prefetch related resources and deliver them together.

There has also been some thinking in the past that clients might want the ability to request and stage all related resources before applying them atomically, providing a stronger consistency model than either eventual or the serialized ADS. To achieve this without significant client complexity would require a resource independent library to be able to discover and stage related resources.

This argues for the inclusion of additional resource dependency information in delivered resource payloads.


### Resource signing

When resources are distributed over untrusted intermediaries, e.g. a transparent cache server, it is desirable to be able to validate their integrity and origin. We propose that an X.509 certificate based signing of xDS resource payloads be used for this. The xDS `Resource` wrapper will be extended to include a signature (of the `Any` bytes), together with the X.509 entity. Note that since we use the byte representation, we bypass the issue around a [lack of deterministic serialization](https://github.com/protocolbuffers/protobuf/issues/5731) of Protobuf messages today 

We should consider prior art such as [https://github.com/theupdateframework/notary](https://github.com/theupdateframework/notary) and [Web Push](https://tools.ietf.org/html/rfc8291) when looking to implement this.

Since there is no immediate need for this mechanism, we exclude it from the scope of next steps.


### General delegation of resources

It would be ideal to be able to have an authority for `RouteConfiguration` be able to delegate control over clusters to an arbitrary authority. This is not possible today due to the CDS source being a singleton for the client, specific in the bootstrap. Ideally resources like `RouteConfiguration` would be augmented where necessary to allow the origin of the resource to delegate any DAG edges at will. See [https://github.com/envoyproxy/envoy/pull/8200](https://github.com/envoyproxy/envoy/pull/8200/files) for a step in this direction.


### Low-latency federated on-demand xDS

In some applications, e.g. serverless functions, it might be desirable to fetch from multiple control planes on a resource miss with on-demand xDS. The proposal outlined below permits this, but only by having a primary control plane responsible for delegation via redirect. This implies an additional round-trip cost. In future a proposal we might consider addressing multiple control planes simultaneously on a miss to reduce this latency as an optimization.


# Proposal


## Logical resource naming

For existing users of xDS transport, we plan no change. The use of opaque resource names, with the known issues above, will be continued.

For operators of control planes that require additional sophistication and need to overcome a number of the issues in the previous section, we will introduce a new resource naming scheme, which will replace the opaque string used today for resource names. This will be a structured proto3 message with URI[^4] canonical encoding. This new name will be effectively a fully qualified cache key, suitable for addressing xDS resources without the need for ancillary information (e.g. `Node` metadata). The key will be exactly matched for a given resource in any given lookup, i.e. it uniquely identifies a resource. A resource at a given version is immutable and the contents should be identical, no matter which path it is retrieved by.


```
message UdpaResourceName {
  // Opaque identifiers for the resource. These are effectively concatenated 
  // with '/' to form the non-query param path as resource ID. 
  repeated string id = 1;

  // Logical authority for resource (not necessarily transport network 
  // address)
  string authority = 2;

  // Resource type URL.
  string type_url = 3;

  // Additional parameters that can be used to select resource variants.
  map<string, string> context_params = 4;
}
```


The above message has a 

<p id="gdcalert5" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: undefined internal link (link text: "udpa:// URI encoding"). Did you generate a TOC? </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert6">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>

[udpa:// URI encoding](#heading=h.85gibjowrc4a), e.g. an example resource name is `udpa://some-authority/envoy.config.listener.v3.Listener/foo/bar?some=thing`.


## Collections


### List

_Collections_ will be modeled via an explicit proto3 type, and will be treated as first-class resources. Collections will be named via `UdpaResourceName`, with a type of the form `&lt;resource>Collection.` For example, listeners with resource type `envoy.config.listener.v3.Listener` will be grouped in an `envoy.config.listener.v3.ListenerCollection`. A `&lt;resource>Collection` is essentially a resource list and will have the following format:


```
message CollectionEntry {
  message InlineEntry {
    string name = 1;
    string version = 2;
    google.protobuf.Any resource = 3;
  }

  oneof resource_specifier {
    // Resource reference.
    UdpaResourceLocator locator = 1;

    // Inlined resource.
    InlineEntry inline_entry = 2;
  }
}
```


 `message &lt;T>Collection {`


```
  repeated CollectionEntry resources = 1;
}
```


The rationale for a distinct type for each collection, rather than a generic collection message, is that this allows for distinct opaque identifier namespaces for collections of each type, e.g. we can have` foo/bar be` used as a name for both listener and cluster collections. This is a form of phantom typing. We call this type of collection a “_list collection”_.

A simple example of a listener fetch sequence is:



1. Client requests `udpa://some-authority/envoy.config.listener.v3.ListenerCollection/foo.`
2. Server responds with a list of resources: `[reference: udpa://some-authority/envoy.config.listener.v3.Listener/bar, reference: udpa://some-authority/envoy.config.listener.v3.Listener/baz].`
3. Client requests both `udpa://some-authority/envoy.config.listener.v3.Listener/bar `and `udpa://some-authority/envoy.config.listener.v3.Listener/baz.`
4. Server responds with resources contents for `udpa://some-authority/envoy.config.listener.v3.Listener/bar `and `udpa://some-authority/envoy.config.listener.v3.Listener/baz`.

This involves two round-trips. Inlining of resources is supported, which can reduce RTT, i.e.



1. Client requests `udpa://some-authority/envoy.config.listener.v3.ListenerCollection/foo.`
2. Server responds with a collection of literal resources inlined in `foo`: [`entry: udpa://some-authority/envoy.config.listener.v3.Listener/bar, entry: udpa://some-authority/envoy.config.listener.v3.Listener/baz].`


### Glob

We also allow for one further optimization. Having explicit collection resources allows delta xDS to work well on individual resources but the resource list itself may become a bottleneck for frequently updated large collections of resources. For a LEDS collection of 20k endpoints that updates every 10s will have a large collection listing, perhaps O(1MB), this can be prohibitive. For this situation, we introduce “_glob collections”_, which allow the `&lt;resource>Collection` to be elided when resource names have a directory structure and a `/*` component is appended to the resource path. Continuing the previous example, this now looks like:



1. Client requests `udpa://some-authority/envoy.config.listener.v3.Listener/foo/*.`
2. Server responds with resources [`udpa://some-authority/envoy.config.listener.v3.Listener/foo/bar, udpa://some-authority/envoy.config.listener.v3.Listener/foo/baz].`

Since each resource returned is subject to independent update via delta xDS and there is no explicit collection list to update, glob collections are highly scalable.


### Use cases

This proposal provides multiple collection mechanisms, as there is a need to tradeoff ergonomics (e.g. being able to reason about a collection via single object), performance (e.g. number of round trips) and scalability (e.g. avoiding a collection object becoming a bottleneck). We further explore this tradeoff in 

<p id="gdcalert6" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: undefined internal link (link text: "alternatives considered"). Did you generate a TOC? </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert7">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>

[alternatives considered](#heading=h.n1f49q8is712). 

Some suggested use patterns for the collections are as follows:



*   Federated collections (list collections with references)
*   HTTP CDN delivery of configuration (list collections with references)
*   Handcrafted configuration on a filesystem (list collections with inline entries)
*   Migration of an existing SotW Envoy configuration (list collections with inline entries)
*   LEDS and high scalability delta APIs (glob collections)

For any other use case with a:



*   Small collection of objects or relatively static configuration, it’s suggested to use list collections with inline entries. This is both performant and requires minimal management server state tracking and round trips.
*   Medium sized collection of objects, list collections with reference entries. The management server only needs to handle subscribe/unsubscribe state tracking, since both the collection and its constituent resources are independent xDS resources.
*   For any other use case with a large collection of objects and/or highly dynamic updates, use glob collections. The management server needs to provide full delta diffs, since there is no explicit collection object to inform the client when resources come and go.


## Resource locators

Resource locators describe how a resource name may be resolved:


```
message UdpaResourceLocator {
  // Opaque identifiers for the resource. These are effectively concatenated 
  // with '/' to form the non-query param path as resource ID. This may end w
  // with '*' for glob collection references.
  repeated string id = 1;

  // Logical authority for resource (not necessarily transport network 
  // address)
  string authority = 2;

  // Resource type URL.
  string type_url = 3;

  oneof context_param_specifier {
    // Additional parameters that can be used to select resource variants.
    // Matches must be exact, i.e. all context parameters must match exactly
    // there must be no additional context parameters set on the matched
    // resource.
    map<string, string> context_params = 4;

    // .. space reserved for future potential matchers, e.g. CEL expressions 
  }

  // Not part of the resource identity, instead these are used at 'points of 
  // reference', such as resource names in a returned resource, to control
  // client behavior while resolving resources. Directives are never sent to 
  // servers when requesting resources, even when part of a 
  // DeltaDiscoveryRequest.
  message Directives {
    oneof directive {
      // An alternative resource name for fallback if the resource is 
      // unavailable. This supports failover. This is useful when control 
      // planes that support failover disagree on resource naming. It's 
      // necessary for the primary control plane to provide an alternative 
      // resource in the secondary. For example, a Listener resource might
      // reference a RouteConfiguration udpa://foo/../routes/some-route-table#alt=udpa://bar/..routes/another-route-table. If the
      // client is unable to reach `foo` to fetch the resource, it will 
      // fallback to `bar`. Alternative resources do not need to have
      // equivalent content, but they should be functionally substitutable.
      UdpaResourceLocator alt = 1;

      // List collections support inlining of resources via the entry field 
      // in UdpaResource. These inlined Resource objects may have an optional 
      // name field specified. When specified, the entry directive allows
      // UdpaResourceLocator to directly reference these inlined resources, 
      // e.g. udpa://.../foo#entry=bar.
      string entry = 2;
    }
  }
  repeated Directive directives = 5;
}
```


Simple string resource names, e.g. RDS route config names in HCM, will be deprecated and mutually exclusive with the use of `UdpaResourceLocator`, e.g.:


```
message Rds {                                                                                                                                                                                                                                                                                                                                                  
  config.core.v3.ConfigSource config_source = 1;                                                                                                         


  string route_config_name = 2 [deprecated = true];

  UdpaResourceLocator udpa_resource_locator = 3;                                                                                                                               
}
```


For resources such as clusters and listeners, `UdpaResourceLocator`s specifying collections will be added to the bootstrap.


## URI encoding

Both `UdpaResourceName`s and `UdpaResourceLocator`s have a canonical transform to a URI:

**_udpa://[{authority}]/{resource type}/{id/*}?{contextual_params}{#processing-directive,*}_**

The resource type is derived from the type URL, e.g. in the above example, it will be `envoy.config.route.v3.RouteConfiguration`. This is the type URL without the need to prefix with `type.googleapis.com/`. A key feature of this typing scheme is the inclusion of the resource API version in the package name. There can be a well known set of resource type shorthands (e.g. route-config-v3) for core Envoy resources.

The processing directives are encoded as:



*   `alt=&lt;canonical UdpaResourceName URI>` if the `alt` field is set. This is typically set inside some resource name in a returned resource, providing fallbacks for the client to pursue.
*   `entry=&lt;resource name>` if the `res` field is set. This is typically set to refer to an inlined resource within a list collection. Resource names must [a-zA-Z0-9_-\./]+.

Multiple directives are comma separated in the fragment component of the URI.

`UdpaResourceName`s will name resources and provide the authority, the logical entity responsible for authoring the resource content. In the future, when resource signing is introduced, it’s likely this will be a DNS name matching the resource author’s X.509 certificate.


## Context parameters

Literal resource names will be embedded in configuration supplied by the control plane for an xDS client. When the xDS client makes a discovery request for a resource, it will modify the `context_params` field to include any 

<p id="gdcalert7" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: undefined internal link (link text: "global context parameters"). Did you generate a TOC? </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert8">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>

[global context parameters](#heading=h.wdyjyh724jol), per-resource type [client feature](https://www.envoyproxy.io/docs/envoy/latest/api/client_features.html) capabilities and per-resource type functional attributes. The algorithm to compute context parameters at the client is as follows. Each step overrides the previous one (i.e. if a same named field appears, it overwrites the value that already exists):



1. Start with the base layers of the 

<p id="gdcalert8" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: undefined internal link (link text: "per-node context parameters"). Did you generate a TOC? </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert9">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>

[per-node context parameters](#heading=h.wdyjyh724jol). E.g. from the bootstrap we might have `{udpa.node.user-agent-name: envoy, udpa.node.metadata.foo: bar}`. All these attributes will be `udpa.`node.`` prefixed. 
2. Overlay with context parameters from the literal resource name.  E.g. from `udpa://foo/listeners/.../bar?some=thing` we might have `{some: thing}.`
3. Overlay with per-resource type [client feature capabilities](https://docs.google.com/document/d/1afQ9wthJxofwOMCAle8qF-ckEfNWUuWtxbmdhxji0sI/edit?ts=5ec19b01#heading=h.wgxgguflfw5o). E.g. listeners at the client might have `{udpa.client_feature.lb.least_loaded: false}`. Client feature capabilities will be required to be declared with respect to a specific resource type URL. All these attributes will be `udpa.`client_feature.`` prefixed.
4. Overlay with per-resource type well-known attributes. E.g. an on-demand listener load might have `{udpa.resource.vip: 96.54.3.1}`. These attributes are generated by code, e.g. an on-demand LDS implementation will first detect the VIP, halt the request, ask for the udpa.`resource.vip` from the control plane, and then resume the request after discovery is completed. For VHDS this might be `{udpa.resource.host: foo.com}`. All these attributes will be `udpa.`resource.`` prefixed.

For the above literal resource name of `udpa://foo/listeners/.../bar?some=thing, `we now have an effective resource name of `udpa://foo/listeners/.../bar?udpa.node.user-agent-name=envoy&udpa.node.metadata.foo=bar&some=thing&udpa.client_feature.lb.least_loaded=false&udpa.resource.vip=96.54.3.1`

We envisage that in the future it might be possible to make use of HTTP-like [Vary](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Vary) header support in both the xDS and HTTP transports to improve caching effectiveness, regardless of what the client includes in the effective resource name.

In addition, we reserve the right to add a context parameter value format in the future that may be used to specify range constraints, e.g. with CEL expressions. These will be prefixed with “`constraint:`”, this prefix for context parameters values is RESERVED. 


## Transport network addressing

Since the UdpaResourceName only provides a logical authority, it’s necessary to map to some physical address during the resource fetch process. In existing xDS, this is provided by a `ConfigSource` delivered alongside the resource name.

We will continue to allow this to happen, however there are limitations that need to be addressed. A single `ConfigSource` may be suitable for one or more authorities, but may not be capable of processing all the authorities that can be encountered due to the `ref` and `alt` directives. For example, suppose a `ConfigSource` was authoritative for `foo.com`, but a `udpa://foo.com/bar `was redirected to `udpa://baz.com/bar`, where the `ConfigSource` was not authoritative for `baz.com`. We would need some way to discover a `ConfigSource` for `baz.com`. We propose amending `ConfigSource` with a list of authorities that it is capable of processing.


```
message Authority {
  string name = 1;

  … when signing is added, items such as CA trust chain, cert pinning … 
}

message ConfigSource {
  repeated Authority authorities = N;

  …  extant ConfigSource … 
}
```


In addition, to support situations in which the `ConfigSource` delivered with a resource name is incapable of acting as a transport for the effective authority, we will introduce in the bootstrap a list of well-known `ConfigSource`s. This list will be processed in order, allowing potentially more than one `ConfigSource` per authority. The last entry is the default. This will typically point to a cluster capable of DNS based resolution based on authority name.


```
message Bootstrap {
  repeated ConfigSource config_sources = N;

  … extant Bootstrap ...
}
```


To support ADS with multiple control planes, the [ApiType](https://github.com/envoyproxy/envoy/blob/31128e7dc22355876020188bc8feb99304663041/api/envoy/config/core/v3/config_source.proto#L44) enum in` Api`ConfigSource will be augmented to specify AGGREGATED_DELTA_GRPC. This will replace and deprecate the existing ads field in `ConfigSource`. When used in `UdpaResourceName` resolution, the `ConfigSource` will act as an ADS transport if configured with this new enum value. Instead of pointing to a single shared ADS `ConfigSource` declared in the bootstrap, any resource authority (regardless of type) that maps to the `ConfigSource` will be multiplexed on the xDS stream specified by the `ConfigSource`.


## Discovery request/responses

`Discovery{Request,Response} `are not supported with the new naming scheme and are effectively deprecated for servers that make use of the scheme. The use of inlined entries in list collections allows simple management servers that do not wish to perform state tracking to perform state-of-the-world updates. This is achieved by sending the entire state, e.g. every listener inlined in a `ListenerCollection`. See 

<p id="gdcalert9" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: undefined internal link (link text: "delta and SotW examples"). Did you generate a TOC? </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert10">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>

[delta and SotW examples](#heading=h.hkc4btyff116) for a worked example of this.

A `DeltaDiscoveryRequest` will include a `UdpaResourceName` alternative to existing opaque resource name strings, deprecating the existing resource name format. In addition, it will be possible to ask for a collection of resources (replacing existing LDS/CDS wildcard). When individual resources are enumerated, they are expected to be delivered by the management server in the response. When a:



*   Listing collection is named, e.g. `udpa://foo.com/envoy.config.listeners.v3.ListenerCollection/bar`, its list of resources is returned. 
*   Glob collection is named, e.g. `udpa://foo.com/envoy.config.listeners.v3.Listener/bar/*`, its constituent resources are delivered.

Control planes may optionally support referring to the same resource via both list and glob collections.


```
message DeltaDiscoveryRequest {
  … 
  string type_url = 2 [deprecated = true];

  repeated string resource_names_subscribe = 3 [deprecated = true];

  repeated string resource_names_unsubscribe = 4 [deprecated = true];

  repeated UdpaResourceName udpa_resources_subscribe = N;

  repeated UdpaResourceName udpa_resources_unsubscribe = N+1;
  … 
}
```


The `type_url` field in both delta discovery requests and responses will no longer be required, since the type is present explicitly in the `UdpaResourceName`.

The `DeltaDiscoveryResponse` message is now:


```
message DeltaDiscoveryResponse {
  … 
  repeated Resource resources = 2;

  string type_url = 4 [deprecated = true];

  repeated string removed_resources = 6 [deprecated = true];

  repeated UdpaResourceName udpa_removed_resources = N;

  // In a future proposal, we leave room to add support for glob collection 
  // key frames. The complete_collection field below will reference 1 or more
  // glob collections URLs, e.g. [udpa://../foo/*, udpa://../bar/*] and
  // inform the client that any resource not explicitly mentioned in 
  // 'resources' in the specified glob collection is removed.
  // repeated UdpaResourceLocator complete_collections = M;
}
```


`udpa_removed_resources` is only used for glob collections. Since list collections are always subscribed to by clients, the management server does not need to make use of this field.

The resource payload in` DeltaDiscoveryResponse`s changes to a revised `Resource`:


```
message Resource {
  UdpaResourceName udpa_resource_name = 5;


  // This is mostly deprecated when working with UdpaResourceNames. However, 
  // it may be optionally used when delivering inline resources in list 
  // collections to name specific resources via #entry directives. 
  // Resource names must match [a-zA-Z0-9_-\./]+ when used as entry names.
  string name = 3;

  repeated string aliases = 4 [deprecated = true];

  string version = 1;

  // Payload is an explicit resource. If this is a UdpaResourceName, a 
  // client-side redirect is implied.
  google.protobuf.Any resource = 2;
}
```


Clients are expected to have knowledge ahead of time (via mechanisms not part of this proposal, e.g. support documentation) on whether the servers specified in the bootstrap supports the new conventions. In the case of Envoy, this will be the LDS and CDS servers. If this support exists, they can start specifying UDPA resource names in requests with the accompanying semantics. Management servers that return resources should be attentive to the use of the UDPA resource name, and when specifying further embedded resources, opt to use the UDPA resource name/locator if the target management server for the resource supports the new scheme. v3 management servers must continue to support both legacy opaque resource string names alongside the newer UDPA resource names (if at all).

It is an error for a `ConfigSource` to have an API type in the enclosed `ApiConfigSource` that is not in `{AGGREGATED_DELTA_GRPC, DELTA_GRPC}` when specifying an xDS transport. Additional API type specifiers will be added for HTTP and file transports.

Redirects are supported by having the `Any` wrapped inside `Resource` be a `UdpaResourceName`. Redirects are only supported for list collections. Glob collections may redirect any individual resource but the collection itself may not be redirected.


## Per-node resource qualification

We do not advocate that the entire Node message becomes part of the context params in a URI. Instead, we propose extending bootstrap with:


```
message Bootstrap {
  // A list of Node field names that will be included in the context 
  // parameters of the effective UdpaResourceName that is sent in a discovery
  // request. Any non-string field will have its JSON encoding set as the
  // context parameter value, with the exception of metadata, which will
  // be flattened (see example below).
  //
  // The node context parameter act as a base layer dictionary for the 
  // context parameters (i.e.
  // more specific resource specific context parameters will override). Field
  // names will be prefixed with "node." when included in context parameters.
  //
  // For example, if node_context_params is ["user-agent-name", "metadata"],
  // the implied context parameters might be:
  //
  // node.user-agent-name: envoy
  // node.metadata.foo: "{bar: baz}"
  // node.metadata.some: 42
  repeated string node_context_params = N;

  … extant Bootstrap …
}
```


These selected node context params will be included in every `UdpaResourceName`’s context params when generating a `DiscoveryRequest`, as described in the section on 

<p id="gdcalert10" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: undefined internal link (link text: "effective resource names"). Did you generate a TOC? </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert11">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>

[effective resource names](#heading=h.nnqgti1ico3k).


# Examples

We provide YAML examples below of discovery request/response sequences for various use cases. Resource names are abbreviated with `udpa://` schema representations.


## Singleton resource request

Client `Cluster` resource:


```
name: some-cluster
eds_cluster_config:
  udpa_resource_locator:
    name: >
udpa://some-authority/envoy.config.endpoint.v3.ClusterLoadAssignment/foo
  eds_config:
    authorities:
    - name: some-authority
    … rest of ConfigSource pointing at xDS management server … 
```


Client EDS `DeltaDiscoveryRequest` sent to xDS management server:


```
udpa_resources_subscribe:
- udpa://some-authority/envoy.config.endpoint.v3.ClusterLoadAssignment/foo
```


xDS management server `DeltaDiscoveryResponse` sent to client:


```
resources:
- udpa_resource_name: >
    udpa://some-authority/envoy.config.endpoint.v3.ClusterLoadAssignment/foo
  version: 1
  resource:
    "@type": >
      type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment
    … foo's ClusterLoadAssignment payload … 
```



## Collection resource request


### List collection

Client bootstrap:


```
dynamic_resources:
  lds_resource_set_locator:
    name: >
udpa://some-authority/envoy.config.listeners.v3.ListenerCollection/foo
  lds_config:
    authorities:
    - name: some-authority
    … rest of ConfigSource pointing at xDS relay proxy … 
```


Client A LDS `DeltaDiscoveryRequest` sent to xDS relay proxy:


```
udpa_resources_subscribe:
- >
udpa://some-authority/envoy.config.listeners.v3.ListenerCollection/foo
```


xDS management server `DeltaDiscoveryResponse` sent to client A:


```
resources:
- udpa_resource_name: >
udpa://some-authority/envoy.config.listeners.v3.ListenerCollection/foo
  version: 1
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.ListenerCollection
    - reference: udpa://some-authority/envoy.config.listeners.v3.Listener/bar
    - reference: udpa://some-authority/envoy.config.listeners.v3.Listener/baz
```


The `bar` and `baz` resources are then fetched with 

<p id="gdcalert11" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: undefined internal link (link text: "singleton resource requests"). Did you generate a TOC? </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert12">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>

[singleton resource requests](#heading=h.2rwk417x2h8).


### List collection with inlining

As with the previous list collection example, but rather than returning a list of references in the response, xDS management server `DeltaDiscoveryResponse` sends to client A:


```
resources:
- udpa_resource_name: >
udpa://some-authority/envoy.config.listeners.v3.ListenerCollection/foo
  version: 1
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.ListenerCollection
    - entry:
        name: bar
        version: 8.5.4
        resource:
          "@type": type.googleapis.com/envoy.config.listeners.v3.Listener
          … bar's ClusterLoadAssignment payload …
    - entry:
        # anonymous resource
        version: 3.9.0
        resource:
          "@type": type.googleapis.com/envoy.config.listeners.v3.Listener
          … baz's ClusterLoadAssignment payload …
```


Note that the first resource bar can be referenced as `udpa://some-authority/envoy.config.listeners.v3.ListenerCollection/foo#entry=bar`, while the second resource is anonymous and cannot be referenced outside the collection.


### Glob collection

Client bootstrap:


```
dynamic_resources:
  lds_resource_set_locator:
    name: >
udpa://some-authority/envoy.config.listeners.v3.Listener/my-listeners/*?node_type=ingress
  lds_config:
    authorities:
    - name: some-authority
    … rest of ConfigSource pointing at xDS relay proxy … 
```


Client A LDS `DeltaDiscoveryRequest` sent to xDS relay proxy (note the use of client capabilities):


```
udpa_resources_subscribe:
- >
udpa://some-authority/envoy.config.listeners.v3.Listener/my-listeners/*?node_type=ingress&client_features.envoy.config.no_bind_to_port=true
```


xDS management server `DeltaDiscoveryResponse` sent to client A:


```
resources:
- udpa_resource_name: >
udpa://some-authority/envoy.config.listeners.v3.Listener/my-listeners/foo?node_type=ingress&client_features.envoy.config.no_bind_to_port=true
  version: 1
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.Listener
    … foo's Listener payload … 
- udpa_resource_name: >
udpa://some-authority/envoy.config.listeners.v3.Listener/my-listeners/bar?node_type=ingress&client_features.envoy.config.no_bind_to_port=true
  version: 42
  resource:
    "@type": type.googleapis.com/envoy.service.discovery.v3.UdpaResourceName
    <udpa://some-authority/envoy.config.listeners.v3.Listener/baz>
```


In this example, `udpa://some-authority/envoy.config.listeners.v3.Listener/my-listeners/bar` was resolved to `udpa://some-authority/envoy.config.listeners.v3.Listener/baz.` The context parameters (ingress, client caps) were used to filter down the listener set for the client in a given collection (`my-listeners`). The client capability is a hypothetical cap indicating client support for UDP listeners.


## Delta & SotW

In this section we look at an example of resource delivery through the mechanisms of both delta and SotW style updates made available by this proposal.


### Delta

Consider the example of a server with two `Listener` objects `foo/bar` and `foo/baz`. This collection might be modeled as either a list or glob collection with delta xDS semantics. In both cases these are modeled as independent resources:



*   `udpa://.../envoy.config.listener.v3.Listener/foo/bar`
*   `udpa://.../envoy.config.listener.v3.Listener/foo/baz`

With list collections, the resources are referenced from a collection `udpa://.../envoy.config.listener.v3.ListenerCollection/my-listeners.`

An Envoy client will initially subscribe to the collection from its bootstrap with:


```
dynamic_resources:
  lds_resource_set_locator:
    name: udpa://.../envoy.config.listener.v3.ListenerCollection/my-listeners
```


Initially the response will be:


```
resources:
- udpa_resource_name: >
udpa://some-authority/envoy.config.listeners.v3.ListenerCollection/my-listeners
  version: 1
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.ListenerCollection
    - reference: udpa://some-authority/envoy.config.listeners.v3.Listener/foo/bar
    - reference: udpa://some-authority/envoy.config.listeners.v3.Listener/foo/baz
```


Both listeners will be fetched independently via additional delta xDS subscriptions. When they change value, they will be updated independently.

When the listener `foo/bar` is removed by the xDS management server, it will send an update to the Envoy with:


```
resources:
- udpa_resource_name: >
udpa://some-authority/envoy.config.listeners.v3.ListenerCollection/my-listeners
  version: 2
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.ListenerCollection
    - reference: udpa://some-authority/envoy.config.listeners.v3.Listener/foo/baz
```


The client will unsubscribe from `foo/bar`.

Later, when a new resource `foo/burp` is made available, the server will send:


```
resources:
- udpa_resource_name: >
udpa://some-authority/envoy.config.listeners.v3.ListenerCollection/my-listeners
  version: 3
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.ListenerCollection
    - reference: udpa://some-authority/envoy.config.listeners.v3.Listener/foo/baz
    - reference: udpa://some-authority/envoy.config.listeners.v3.Listener/foo/burp
```


And Envoy will subscribe to `foo/burp`.

For glob collections, the bootstrap would be:


```
dynamic_resources:
  lds_resource_set_locator:
    name: udpa://.../envoy.config.listener.v3.Listener/foo/*
```


The management server would then deliver two independent resources in a `DeltaDiscoveryResponse`:


```
resources:
- udpa_resource_name: udpa://some-authority/envoy.config.listeners.v3.Listener/foo/bar
  ...
- udpa_resource_name: 
udpa://some-authority/envoy.config.listeners.v3.Listener/foo/baz
  ...
```


These resources will be updated independently of one another as regular delta XDS resources. When the manager wishes to remove a resource `foo/bar`, it will respond with:


```
udpa_removed_resources:
- udpa://some-authority/envoy.config.listeners.v3.Listener/foo/bar
```



### SotW

Some management servers will be simple and not able to provide delta updates, or even able to manage subscriptions for individual resources in a collection. For example, if the server is taking a static configuration file for listeners emitted by a config pipeline, it may want to serve the entire file up as the listener SotW. In this case, inlined list collection entries will be used. Continuing the above example:

An Envoy client will initially subscribe to the collection from its bootstrap with:


```
dynamic_resources:
  lds_resource_set_locator:
    name: udpa://.../envoy.config.listener.v3.ListenerCollection/my-listeners
```


Initially the response will be:


```
resources:
- udpa_resource_name: >
udpa://some-authority/envoy.config.listeners.v3.ListenerCollection/my-listeners
  version: 1
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.ListenerCollection
    - entry: <Resource object for foo/bar>
    - entry: <Resource object for foo/baz>
```


The Envoy client now has the complete listener set state.

When the listener foo/bar is removed by the xDS management server, the SotW update for the collection will be:


```
resources:
- udpa_resource_name: >
udpa://some-authority/envoy.config.listeners.v3.ListenerCollection/my-listeners
  version: 1
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.ListenerCollection
    - entry: <Resource object for foo/baz>
```



## Redirects

In this example, a management server wants to delegate authority over some resource to another management server.

The config is as with the 

<p id="gdcalert12" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: undefined internal link (link text: "singleton resource request"). Did you generate a TOC? </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert13">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>

[singleton resource request](#heading=h.2rwk417x2h8) above.

Client EDS `DeltaDiscoveryRequest` sent to xDS management server:


```
udpa_resources_subscribe:
- udpa://some-authority/envoy.config.endpoint.v3.ClusterLoadAssignment/foo
```


xDS management server `DiscoveryResponse` sent to client:


```
resources:
- "@type": type.googleapis.com/envoy.service.discovery.v3.Resource
  udpa_resource_name: > udpa://some-authority/envoy.config.endpoint.v3.ClusterLoadAssignment/foo
  resource:
    "@type": type.googleapis.com/envoy.service.discovery.v3.UdpaResourceName
  <udpa://other-authority/envoy.config.endpoint.v3.ClusterLoadAssignment/bar> 
```


At this point, the client then sends a new `DiscoveryRequest` after resolving a `ConfigSource `for `other-authority:`


```
specifiers:
- resource_name: >
  udpa://other-authority/envoy.config.endpoint.v3.ClusterLoadAssignment/bar

… Singleton resource response … 
```



## Alternatives

In this example, an xDS-as-a-service control plane provided by management server X wants to specify on-premise control plane resources as failovers on network partition.

The xDS-as-a-service management server `DeltaDiscoveryResponse` sends to the client inside a `Cluster` resource:


```
name: some-cluster
type: EDS
eds_cluster_config:
  udpa_resource_locator:
    name: >
udpa://some-cloud-authority/envoy.config.endpoint.v3.ClusterLoadAssignment/foo#alt=udpa://some-onprem-authority/envoy.config.endpoint.v3.ClusterLoadAssignment/bar
  eds_config:
    authorities:
    - name: some-cloud-authority
    … rest of ConfigSource pointing at xDS management server … 
    authorities:
    - name: some-onprem-authority
    … rest of ConfigSource pointing at xDS management server … 
```


Client EDS `DeltaDiscoveryRequest` sends to xDS management server for `some-cloud-authority`:


```
udpa_resources_subscribe:
- >
udpa://some-cloud-authority/envoy.config.endpoint.v3.ClusterLoadAssignment/foo
```


Under normal conditions, the requested resource is returned as above. If the config source for `some-cloud-authority` is unreachable, then the client issues a new `DeltaDiscoveryRequest` to `some-onprem-authority`:


```
udpa_resources_subscribe:
- >
udpa://some-onprem-authority/envoy.config.endpoint.v3.ClusterLoadAssignment/bar
```



## VHDS

A VHDS example (this could be used for on-demand filter chains via SNI in the future as well).

Client `RouteConfiguration` config:


```
name: some-route-config
vhds:
  udpa_resource_locator:
    name: >
udpa://some-authority/envoy.config.route.v3.VirtualHost/virtual-host-table
  config_source:
    - some-authority
    … rest of ConfigSource pointing at xDS management server … 
```


When foo.com misses on the local virtual hosts, Envoy sends the following `DeltaDiscoveryRequest`:


```
udpa_resources_subscribe:
- >
udpa://some-authority/envoy.config.route.v3.VirtualHost/virtual-host-table?resource.host=foo.com
```


xDS management server `DeltaiscoveryResponse` sent to client:


```
resources:
- "@type": type.googleapis.com/envoy.service.discovery.v3.Resource
  udpa_resource_name: >
udpa://some-authority/envoy.config.route.v3.VirtualHost/virtual-host-table?resource.host=foo.com
 resource:
    "@type": type.googleapis.com/envoy.service.discovery.v3.UdpaResourceName
<udpa://some-authority/envoy.config.route.v3.VirtualHost/some-foo-resource>
```



## xDS relay proxy

In this example, two clients work with an xDS relay proxy to receive their configuration from some canonical xDS server.

Client A bootstrap:


```
dynamic_resources:
  lds_resource_set_locator:
    name: >
udpa://some-authority/envoy.config.listeners.v3.Listener/a-listeners/*
  lds_config:
    authorities:
    - name: some-authority
    … rest of ConfigSource pointing at xDS relay proxy … 
```


Client A LDS  `DeltaDiscoveryRequest` sent to xDS relay proxy:


```
udpa_resources_subscribe:
- udpa://some-authority/envoy.config.listeners.v3.Listener/a-listeners/*
```


Client B bootstrap:


```
dynamic_resources:
  lds_resource_set_locator:
    name: >
    udpa://some-authority/envoy.config.listeners.v3.Listener/b-listeners/*
  lds_config:
    authorities:
    - mame: some-authority
    … rest of ConfigSource pointing at xDS relay proxy … 
```


Client B LDS `DiscoveryRequest` sent to xDS relay proxy:


```
udpa_resources_subscribe:
- resource_name: >
  udpa://some-authority/envoy.config.listeners.v3.Listener/b-listeners/*
```


xDS relay proxy `DeltaDiscoveryRequest` sent to some `some-authority`:


```
udpa_resources_subscribe:
- udpa://some-authority/envoy.config.listeners.v3.Listener/a-listeners/*
- udpa://some-authority/envoy.config.listeners.v3.Listener/b-listeners/*
```


`some-authority DeltaDiscoveryResponse` sent to xDS relay proxy:


```
resources:
- "@type": type.googleapis.com/envoy.service.discovery.v3.Resource
  udpa_resource_name: >
    udpa://some-authority/envoy.config.listeners.v3.Listener/a-listeners/foo
  version: 1
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.Listener
    … foo's Listener payload … 
- "@type": type.googleapis.com/envoy.service.discovery.v3.Resource
  udpa_resource_name: >
    udpa://some-authority/envoy.config.listeners.v3.Listener/a-listeners/bar
  version: 42
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.Listener
    … bar's Listener payload … 
  udpa_resource_name: >
    udpa://some-authority/envoy.config.listeners.v3.Listener/b-listeners/baz
  version: 13
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.Listener
    … baz's Listener payload … 
```


xDS relay proxy `DeltaiscoveryResponse` sent to client A:


```
resources:
- "@type": type.googleapis.com/envoy.service.discovery.v3.Resource
  udpa_resource_name: >
    udpa://some-authority/envoy.config.listeners.v3.Listener/a-listeners/foo
  version: 1
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.Listener
    … foo's Listener payload … 
- "@type": type.googleapis.com/envoy.service.discovery.v3.Resource
  udpa_resource_name: >
    udpa://some-authority/envoy.config.listeners.v3.Listener/a-listeners/bar
  version: 42
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.Listener
    … bar's Listener payload … 
```


xDS relay proxy `DeltaiscoveryResponse` sent to client B:


```
resources:
- "@type": type.googleapis.com/envoy.service.discovery.v3.Resource
  udpa_resource_name: >
    udpa://some-authority/envoy.config.listeners.v3.Listener/b-listeners/baz
  version: 13
  resource:
    "@type": type.googleapis.com/envoy.config.listeners.v3.Listener
    … baz's Listener payload … 
```



# Open questions



*   Do we need to support dynamic `ConfigSource` delivery for redirects, where the client does not know all the authorities it might interact with in advance? 
    *   One use case is decomposing a control plane without mandating bootstrap changes.
    *    Any other use cases?
*   Should additional support be added for noting deletion of resources in the `Resource` wrapper?
*   Do we need filesystem path restrictions for file resources?
    *   Where are these configured (bootstrap?)? ConfigSource?
    *   What is the trust model?
    *   Are we worried about adding scope for directory traversal attacks?


# Roadmap

For Envoy, the following incremental roadmap is proposed:



1. Add `UdpaResourceNames` as alternatives for resource names across the v3 API. Assume existing ConfigSources are suitable for named authorities.
2. Extend discovery requests/responses to support `UdpaResourceName`s. Plumb the new resource names into discovery requests/response handling. This does not include support for directives such as alt/entry or redirects.
3. Add support for per-node resource qualification. Plumb this into discovery requests.
4. Add support for collections in discovery request/responses, demonstrate how this can be used for LDS/CDS. At this point there will be both filesystem and xDS transports implemented.
5. Add `ConfigSource` list to bootstrap and allow use as an alternative to inline `ConfigSource`.
6. Implement redirects, alt/entry directives. This is probably the most involved as it involves adding fallback behavior to Envoy for xDS resources and reference chasing.
7. Add native HTTP transport for xDS resources.
8. Migrate VHDS to the new naming scheme.

Moving beyond this proposal, there is likely to be future extensions to resource naming and response payloads that are needed to support stronger consistency models. These are beyond the scope of this document, but the extensible nature of URIs and proto3 messages described above should provide some room to grow here.

We currently plan to have 1-5 intercept Envoy 1.16.0. Step 6 should intercept 1.17.0. The remaining steps will be executed as needed.

In parallel to the above effort, the Istio xDS proxy effort, from which the above scheme was largely borrowed,  is likely to prove out various concepts prior to their inclusion in Envoy.


# Alternatives considered


## Collections


### Glob collections

Collections are not first-class types, instead resource naming indicates a collection via a trailing /*. Member resources must match the path structure of the parent collection.



1. Client requests `udpa://some-authority/envoy.config.listener.v3.Listener/foo/*.`
2. Server responds with resources [`udpa://some-authority/envoy.config.listener.v3.Listener/foo/bar, udpa://some-authority/envoy.config.listener.v3.Listener/foo/baz].`

Pros:



*   Aligns with v3 delta xDS and will support scalable LEDS implementation.
*   Matches filesystem glob semantics when mapping to a file transport.
*   Doesn’t require proto3 gymnastics to introduce generic collections.

Cons:



*   Forces a directory structure on path naming for resources. Where this is not the convention, will require the use of additional redirect resources. This becomes acute with federation and multiple administrative domains.
*   Redirects make human authored configuration unwieldy, likely to be an ergonomic failure for the API. Consider the case of a hand written configuration stored in S3 and using the HTTP native transport, or a local filesystem config.
*   Redirection of collections requires special case treatment.
*   HTTP native transport needs to diverge and will look more like a list collection (below).


### List collections

Collections are first-class proto3 messages, e.g.: 

 `message &lt;T>Collection {`


```
  repeated UdpaResourceName resources = 1;
}
```


A simple example of a listener fetch sequence is:



1. Client requests `udpa://some-authority/envoy.config.listener.v3.ListenerCollection/foo.`
2. Server responds with a list of resources [`udpa://some-authority/envoy.config.listener.v3.Listener/bar, udpa://some-authority/envoy.config.listener.v3.Listener/baz].`
3. Client requests both `udpa://some-authority/envoy.config.listener.v3.Listener/bar `and `udpa://some-authority/envoy.config.listener.v3.Listener/baz.`
4. Server responds with resources contents for `udpa://some-authority/envoy.config.listener.v3.Listener/bar `and `udpa://some-authority/envoy.config.listener.v3.Listener/baz`.

This involves two round-trips. As an optional/additional/future optimization, this proposal can allow the server to include the `bar` and `baz` resources in the same `DiscoveryResponse` as the `foo` collection in 2, reducing the round-trips to one.

Pros:



*   Collections can be mapped to native xDS, HTTP and filesystem in an intuitive way.
*   Everything is a resource; uniform handling of features such as redirects.
*   Low friction human authored configuration.
*   Flexible naming; all opaque identifiers are fully under the control of the server or config author.

Cons:



*   Additional round-trip required to fetch collection resources, or additional client complexity to support the single round-trip optimization. This introduces a latency vs. complexity tradeoff.
*   The collection resource list is atomic and won’t be subject to incremental update with delta xDS. This is likely to become a bottleneck for LEDS, for O(20k) endpoints in a group and 100 byte names, this is O(1MB) of data which might be shipped every few seconds.


### Mixed glob/list collections (preferred)

Provide both glob and list collections. That is, client can ask for either `udpa://some-authority/envoy.config.listener.v3.ListenerCollection/foo or udpa://some-authority/envoy.config.listener.v3.Listener/foo/*`. List collections would be the normal with glob collections reserved for special cases like LEDS.

Glob collections will not be available to HTTP or filesystem transports, since we do not currently plan on optimizing them for scale up configuration delivery (only scale out). HTTP and filesystem transports will stick to pure list collections.

Pros:



*   All pros from previous sections on glob/list resources.

Cons:



*   Redundant mechanisms for collections.
*   Additional xDS client and server complexity.


### List collections with stream optimized responses

The idea here is to stick to a pure list collection model. Clients will always request a list collection, e.g. `udpa://some-authority/envoy.config.listener.v3.ListenerCollection/foo`. On the xDS transport, the server can respond with either the collection resource list or with individual resources, eliding the collection resource list. This requires additional information to be made available to the client in order to reconcile the returned resource with the requested collection. 

This optimization will not be available to HTTP or filesystem transports, since we do not currently plan on optimizing them for scale up configuration delivery (only scale out). HTTP and filesystem transports will stick to pure list collections.


#### Parent collection reference (name)



1. Client requests `udpa://some-authority/envoy.config.listener.v3.ListenerCollection/foo.`
2. Server can respond with the full collection as 

<p id="gdcalert13" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: undefined internal link (link text: "above"). Did you generate a TOC? </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert14">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>

[above](#heading=h.1xfr9pprvqpe), or instead with resources [`udpa://some-authority/envoy.config.listener.v3.Listener/bar, udpa://some-authority/envoy.config.listener.v3.Listener/baz].` Inside the `Resource` message for `bar` and `baz` is a field providing the name of the parent collection, `udpa://some-authority/envoy.config.listener.v3.ListenerCollection/foo.`

Pros:



*   A single collection model that supports all the pros of list collections while also avoiding the problem of the resource list becoming a bottleneck for LEDS.

Cons:



*   Parent collection references will be costly on the wire. They will double the size of responses when resource payload is small. This overhead will be amortized as resources grow in size. It may be plausible to batch resources together in a single response at the expense of some additional complexity[^5].


*   Additional single hop delta xDS wire complexity due to dual mechanisms.


#### Parent collection reference (hash)

As with the parent collection reference above, but the server references the parent collection with H(parent collection name), where H is a hash function, providing a compact hash for the parent collection reference.



1. Client requests `udpa://some-authority/envoy.config.listener.v3.ListenerCollection/foo.`
2. Server can respond with the full collection as above, or instead with resources [`udpa://some-authority/envoy.config.listener.v3.Listener/bar, udpa://some-authority/envoy.config.listener.v3.Listener/baz].` Inside the Resource message for bar and baz is a field providing H(`udpa://some-authority/envoy.config.listener.v3.ListenerCollection/foo).`

Pros:



*   A single collection model that supports all the pros of list collections while also avoiding the problem of the resource list becoming a bottleneck for LEDS.

Cons:



*   Some additional single hop delta xDS wire complexity due to dual mechanisms.


#### Implied path



1. Client requests `udpa://some-authority/envoy.config.listener.v3.ListenerCollection/foo.`
2. Server can respond with the full collection as 

<p id="gdcalert14" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: undefined internal link (link text: "above"). Did you generate a TOC? </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert15">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>

[above](#heading=h.1xfr9pprvqpe), or instead with resources [`udpa://some-authority/envoy.config.listener.v3.Listener/foo/bar, udpa://some-authority/envoy.config.listener.v3.Listener/foo/baz].` The directory structure of the collection provides the path structure for the member resource.

Pros:



*   A single collection model that supports all the pros of list collections while also avoiding the problem of the resource list becoming a bottleneck for LEDS.
*   Wire efficient vs. parent collection references.

Cons:



*   Singleton resource identifier namespaces are coupled with the naming of collections. This is even worse than with glob collections, since the namespaces belong to different types, i.e. the naming in `envoy.config.listener.v3.ListenerCollection `influences the naming of `envoy.config.listener.v3.Listener`. This only affects LEDS though, other resource types still have naming flexibility with full list collections.


#### Glob collections with materialized resources

This alternative augments simple glob collections by allowing a concrete resource to be either a single `Resource` or a list of materialized (inline) `Resource`s. This provides easy to work with representations for human authored configurations while continuing to support scale up delta xDS. There is no need to introduce additional types of the form `&lt;T>Collection`.

Example URIs:



*   `udpa://x/envoy.v3.Listener/foo/bar/* `has glob semantics.
*   `udpa://x/envoy.v3.Listener/foo/bar/baz` is either a singleton `Listener` or a `repeated Resource` containing a materialized collection of listeners.

In the case where the identified resource is a `repeated Resourc`e then we deal with names of the resources contained within as identifiable fragments of the container but this is purely a runtime concern and they wouldn't need to have their own fully qualified udpa:// URIs, they can have simple flat names which would make these artifacts relatively portable.

So if:


```
udpa://x/envoy.v3.Listener/foo/bar/baz  is a repeated Resource
```


Then:

`udpa://x/envoy.v3.Listener/for/bar/baz#boo` is a named listener inside it

(forgive the overloading of fragments using '#")

You can't reload `'boo'` without reloading `'baz'` so it has that SoTW / materialized set property. Composition would still work if you wanted to refer to a single listener inside the container.

Pros:



*   Supports compact inline representation of resource collections, works well with human authored artifacts.
*   Maps to native xDS, HTTP and filesystem transports.
*   Some forms of collections can be redirected easily.

Cons:



*   Increased complexity of name resolution (need to add the additional #boo).
*   Some additional complexity due to the multiple representation forms and indexing methods.

There is an interplay between how [sets of] resources are named and how collections of resources are represented in discovery responses. In the alternatives below, we have a running example of two singleton listener resources A and B. 

We want to both be able to refer to A and B directly, e.g.:



*   `udpa://foo/listener/my-listeners/A`
*   `udpa://foo/listener/my-listeners/B`

This singleton reference capability is useful, for example when describing a resource inside a container, or when explicitly requesting a resource in a gRPC client that only knows about a single listener in LDS.

We also need to be able to refer to sets of resources, e.g. when an Envoy client wants to ask for all known listeners in LDS in some namespace. Below we provide some alternatives for how to refer to these _collections_ of listeners.


## Node context parameters

Rather than selecting `Node` fields to include in the resource context parameters, a distinct `map&lt;string, string>` could have been added to the bootstrap. The rationale for the above choice is:



*   Some `Node` fields will be useful as context parameters in different scenarios. In particular, user agent name/version seem likely to be candidates for this in the situation where a control plane needs to correct for buggy client implementations. Placing control of this in the bootstrap provides operator control on this selection, at the expense of creating a query parameter schema that needs to be consumed across a potential federated control plane. All management entities in this federated control plane must be capable of ignoring weak unknown query parameters.
*   We avoid any redundancy with existing Node information in the context parameters.
*   This scheme provides backwards compatibility with the existing use of `Node`.


# Appendix A: Why URIs?

The choice of a [canonical URI encoding scheme](#bookmark=id.xqg6f6prng5s) is motivated by a few considerations:



*   URIs are a [RFC standards based representation](https://tools.ietf.org/html/rfc3986) of resource identifiers. There are many tools, libraries and frameworks that understand URI structure. While a Protobuf representation of resource identity is easily understood by Envoy developers, the wider ecosystem is not necessarily conversant in Protobuf and is more likely to grok a URI form.
*   When mapping resource identities to alternative transports to the xDS transport, a URI structure aids in this transposition. It’s easy to see how `udpa://some-authority.com/listener/foo` can be served by a CDN `http://static.some-authority.com/listener/foo`.
*   URIs provide a compact human readable format to describe resource names. This is significant when working with hand crafted JSON or YAML representations of configuration, including in documentation and example configs.
*   Earlier versions of Envoy can make use of URIs as an opaque string and participate in a service mesh without needing to have awareness of the URI structure.

A limitation of the choice of URI as canonical representation is that use of query parameters and directives is needed for future growth, since the scheme and path are fixed. We are unaware of any challenges from this restriction at this point.


# Appendix B: HTTP and file transports

A `UdpaResourceName` identifies an xDS resource, but does not describe how it will be fetched on a given transport. In the earlier part of this document, the 

<p id="gdcalert15" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: undefined internal link (link text: "method of ConfigSource resolution"). Did you generate a TOC? </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert16">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>

[method of ConfigSource resolution](#heading=h.4iqy2glbz1r1) was introduced, with the expectation that a gRPC SotW or delta xDS transport would be used. In this section we describe how other transports are supported.


## File transport

The file transport in xDS is known to be used in a few scenarios:



*   When resources are needed prior to control plane availability (e.g. secrets).
*   When reliability is required in the face of potential control plane outages.
*   When resources are authored by local agents.
*   When providing configuration via version control systems such as git.

Envoy and xDS already have a primitive method of accessing `UdpaResourceName` resources on the filesystem via paths in `ConfigSource`. However, each authority can only be represented via a single path, so this only works when the `ConfigSource` is bundled with the resource name, which is severely limiting.

The recommended way to work with the filesystem is via a native file URI:

**_file:///{path/*}{_#&lt;resource name>}?**

In this representation, the path of the file containing the resource is provided explicitly, allowing the client to bypass `ConfigSource` lookup. Each file specifies precisely one resource (which may be a collection type, e.g. ListenerCollection). The schema of a file is `Resource `message in YAML format. Unlike the xDS transport, there is no explicit type or context parameters. The type is contained in the returned resource (and implied by the referencing resource). Context parameters have limited utility on a local filesystem.

An example file resource at `file:///a/file/system/path/some-listener.yaml` is:


```
name: foo
version: 1
resource:
  "@type": "envoy.config.listeners.v3.Listener"
    … foo's ClusterLoadAssignment payload … 
```


Glob collections are directories on the filesystem, e.g. `file:///a/file/system/path/my-listeners/* `has every file in the directory as a resource belonging to the collection `/a/file/system/path/my-listeners.`

List collections are regular named `Resource`s. Inlined resources can be reference with named anchors as per the xDS transport, e.g. `file:///a/file/system/path/my-listeners-collection.yaml#entry=foo.`


## HTTP transport

As with the file system, there are both simple and more sophisticated approaches to mapping xDS resource discovery to an HTTP transport.

In the simple variant, we use the existing support for REST-JSON subscriptions in [ConfigSource](https://github.com/envoyproxy/envoy/blob/ab32f5fd01ca8b23ee16dcffb55b1276e55bf1fa/api/envoy/config/core/v3/config_source.proto#L50) once the authority is resolved. This is not an HTTP native representation, since we’re effectively tunneling the gRPC transport through an HTTP/JSON representation, including the discovery request/responses. This is not likely to work with non-xDS aware HTTP servers, such as CDNs.

To overcome this limitation, we have a native URI transform from `udpa://&lt;some UDPA URI>` schema to `http[s]://`. This involves simply replacing the scheme with `http[s]` and ensuring that the authority is a DNS resolvable entity.

The following restrictions apply:



*   Only list collections are supported.
*   The HTTP server does not provide delta capabilities.
*   There is no ACK/NACK. It is expected that an out-of-band client status monitoring mechanism exists.

One of the advantages of the xDS transport is that it provides delta updates and client acceptance status, we expect that the HTTP transport will be used for scale out scenarios where the xDS transport is too costly, while the xDS transport will be used for scale up.

In the HTTP transport



*   Singleton and collection resources are the JSON/YAML specifying the contents.
    *   The JSON/YAML distinction will be encoded in the `content-type:` header
    *   The xDS type is in the URL, so there is no need to repeat this in headers or body.
    *   Resource versioning moves to HTTP [ETags](https://en.wikipedia.org/wiki/HTTP_ETag). 
*   List collections are also versioned with HTTP [ETags](https://en.wikipedia.org/wiki/HTTP_ETag). Explicit collection versioning was not necessary in the xDS transport as the pub-sub semantics guaranteed updates to the client whenever the collection changed.
*   Redirects will work via 302, pointing to the next HTTP URI for the xDS resource.
*   [Resource TTLs](https://github.com/envoyproxy/envoy/pull/10898) are expressed in `cache-control` headers.

To efficiently support this representation, deployments need to consider:



*   HTTP protocol; this will be most efficient with HTTP/2
*   The ability of the HTTP server to effectively mask and condition on query parameters. Support for custom cache keys is likely to be useful.
*   That this transport is explicitly designed for client scalability, i.e. having many clients, vs. xDS configuration scalability, i.e. having many resources, hence no delta support.
*   That there is a tradeoff to be made in whether resources are inlined inside a collection response. If they are inlined, the collection will need to be fetched (in full) more frequently if the resource changes frequently. If there are many resources, the number of HTTP requests can be minimized by inlining. Hence both the number of resources and their rate of change should be considered.
*   Whether to periodically poll for change or long poll. This will depend on the server capability with respect to etag semantics. If a TTL is provided in a fetched resource, polling might derive its period from the TTL.

An example sequence to fetch the listener set for an xDS client over HTTP from a CDN:

An xDS client has v1 of some listener collection and sends to the CDN HTTP server:


```
host: some-cache-server.com
:path: /envoy.config.listeners.v3.ListenerCollection/foo
:method: GET
if-none-match: v1
```


If there are no new updates to the collection available, the server may:



*   Respond with 304 (not modified).
*   Long poll until a new collection update is available.

The server will then respond on a change to the listener collection with:


```
:status: 200
content-type: text/yaml
etag: v2

- name: http://some-cache-server.com/envoy.config.listeners.v3.Listener/bar
- name: http://some-cache-server.com/envoy.config.listeners.v3.Listener/baz
- value:
    name: blurp
```


        `contents: &lt;YAML representation of blurb's resource contents>`

The client then issues a request for each resource, e.g.:


```
host: some-cache-server.com
:path: //envoy.config.listeners.v3.Listener/bar
:method: GET
if-none-match: <existing resource version for bar>
```


As with collections, if there are no new updates to the resource available, the server may:



*   Respond with 304 (not modified).
*   Long poll until a new collection update is available.

The server will then respond on a change to `bar` with:


```
:status: 200
content-type: text/yaml
cache-control: max-age=7200
etag: vN

<YAML representation of resource bar contents>
```


In this example, a TTL was set for `bar`, forcing an invalidation and refetch by the client on expiration.

Whenever the listener collection changes, its ETag will change, so when listener resource `blurp` changes, the entire listener collection will be refetched. Listener `bar` may change independently to the resource collection. In the above example, 3 HTTP request streams exist, one for the listener collection and one for each of `bar` and `baz`.

When fetching listener `baz`, the server may respond with:


```
:status: 302 Found
location: http://other-cache-server.com/v3-listener/foo/M
```


This will redirect as with[ xDS transport redirects](#bookmark=id.56ojmoo5i26i).


<!-- Footnotes themselves at the bottom. -->
## Notes

[^1]:
     With the exception of LRS, HDS and other Envoy APIs for streaming data back to the server (TAP, metrics, access logging, etc.).

[^2]:
     Although not by some other clients, e.g. gRPC, which request singleton resources.

[^3]:
     There's also the assumption that Node identity (encoded in Metadata) is equivalent to the identity of the resource in the control plane that will be used to configure it.
    While not an unreasonable assumption it's not one that is formalized within the API. It is formalized by omission but that is not a viable naming strategy long term.

[^4]:
     For further details of why URIs are significant, see [https://docs.google.com/document/d/1zZav-IYxMO0mP2A7y5XHBa9v0eXyirw1CxrLLSUqWR8/edit?ts=5ea703e8#heading=h.xt1ckjvpo85x](https://docs.google.com/document/d/1zZav-IYxMO0mP2A7y5XHBa9v0eXyirw1CxrLLSUqWR8/edit?ts=5ea703e8#heading=h.xt1ckjvpo85x).

[^5]:

     From earlier discussion with Louis: if we were to change DiscoveryRequest/Response so that only one collection is addressable per exchange, this requirement could be relaxed. This would require some changes to how DiscoveryRequest/Responses batch.

