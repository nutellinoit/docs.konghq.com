---
name: TLS Handshake Modifier
publisher: Kong Inc.

desc: Requests a client to present its client certificate
description: |
  The plugin requests, but does not require the client certificate. No validation
  of the client certificate is performed. If a client certificate exists,
  the plugin makes the certificate available to other plugins acting on this request.  

  This plugin must be used in conjunction with the TLS Metadata Headers plugin.

enterprise: true
plus: false # need to update this once we verify that this plugin is in Konnect post-3.0 update
cloud: false # need to update this once we verify that this plugin is in Konnect post-3.0 update
type: plugin
categories:
  - security
kong_version_compatibility:
  enterprise_edition:
    compatible:
      - 3.0.x
params:
  name: tls-handshake-modifier
  service_id: true
  consumer_id: false
  route_id: true
  protocols:
    - name: https
    - name: grpcs
    - name: tls
  dbless_compatible: 'yes'
  konnect_examples: false # need to update this once we verify that this plugin is in Konnect post-3.0 update
  config:
    - name: tls_client_certificate
      required: false
      default: REQUEST
      datatype: string
      description: |
        Indicates the TLS handshake preference. The only option is `REQUEST`, which
        requests the client certificate.
---

## Client certificate request

Client certificates are requested in the `ssl_certificate_by_lua` phase where {{site.base_gateway}} does not
have access to `route` and `workspace` information. Due to this information gap, {{site.base_gateway}} asks for
the client certificate on every handshake if the `tls-handshake-modifier` plugin is configured on any route or service.

In most cases, the failure of the client to present a client certificate doesn't affect subsequent
proxying if that route or service does not have the `tls-handshake-modifier` plugin applied. The exception is where
the client is a desktop browser, which prompts the end user to choose the client cert to send and
leads to user experience issues rather than proxy behavior problems.

To improve this situation, Kong builds an in-memory map of SNIs from the configured {{site.base_gateway}} routes that should present a client
certificate. To limit client certificate requests during a handshake while ensuring the client
certificate is requested when needed, the in-memory map is dependent on all the routes in
{{site.base_gateway}} having the SNIs attribute set. When no routes have SNIs set, {{site.base_gateway}} must request
the client certificate during every TLS handshake:

- On every request irrespective of workspace when the plugin is enabled in global workspace scope.
- On every request irrespective of workspace when the plugin is applied at the service level
  and one or more of the routes *do not* have SNIs set.
- On every request irrespective of workspace when the plugin is applied at the route level
  and one or more routes *do not* have SNIs set.
- On specific requests only when the plugin is applied at the route level and all routes have SNIs set.

The result is all routes must have SNIs if you want to restrict the handshake request
for client certificates to specific requests.
