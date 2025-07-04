# GEP-91: Client Certificate Validation for TLS terminating at the Gateway

* Issue: [#91](https://github.com/kubernetes-sigs/gateway-api/issues/91)
* Status: Implementable

(See definitions in [GEP States](../overview.md#gep-states).)

## TLDR

This GEP proposes a way to validate the TLS certificate presented by the frontend client to the server
(Gateway in this case) during a [TLS Handshake Protocol][].

## Goals

* Define an API field to specify the CA Certificate within the Gateway configuration that can be used as a trust anchor to validate the certificates presented by the client.
This use case has been highlighted in the [TLS Configuration GEP][] under segment 1 and in the [Gateway API TLS Use Cases][] document under point 7.
* Ensure the configuration mitigates the authentication bypass risks associated with HTTP/2 connection coalesing as described in [GEP-3567](https://gateway-api.sigs.k8s.io/geps/gep-3567/#interaction-with-client-cert-validation).
* Supporting a mode where validating client certificates is optional, useful for debugging and migrating to strict TLS.

## Non-Goals
* Define other fields that can be used to verify the client certificate such as the Certificate Hash.

### API

* Introduce a `FrontendValidation` field of type `FrontendTLSValidation` within [GatewayTLSConfig][] that can be used to validate the peer (frontend) with which the TLS connection is being made.
* Introduce a `caCertificateRefs` field within `FrontendTLSValidation` that can be used to specify a list of CA Certificates that can be used as a trust anchor to validate the certificates presented by the client.
* This new field is separate from the existing [BackendTLSPolicy][] configuration. [BackendTLSPolicy][] controls TLS certificate validation for connections *from* the
  Gateway to the backend service.  This proposal adds the ability to validate the TLS certificate presented by the *client* connecting to the Gateway (the
  frontend). These two validation mechanisms operate independently and can be used simultaneously.
* Also introduce a `ObjectReference` structure that can be used to specify `caCertificateRefs` references.

#### GO

```go
// ObjectReference identifies an API object including its namespace.
//
// The API object must be valid in the cluster; the Group and Kind must
// be registered in the cluster for this reference to be valid.
//
// References to objects with invalid Group and Kind are not valid, and must
// be rejected by the implementation, with appropriate Conditions set
// on the containing object.
type ObjectReference struct {
	// Group is the group of the referent. For example, "gateway.networking.k8s.io".
	// When unspecified or empty string, core API group is inferred.
	Group Group `json:"group"`

	// Kind is kind of the referent. For example "ConfigMap" or "Service".
	Kind Kind `json:"kind"`

	// Name is the name of the referent.
	Name ObjectName `json:"name"`

	// Namespace is the namespace of the referenced object. When unspecified, the local
	// namespace is inferred.
	//
	// Note that when a namespace different than the local namespace is specified,
	// a ReferenceGrant object is required in the referent namespace to allow that
	// namespace's owner to accept the reference. See the ReferenceGrant
	// documentation for details.
	//
	// Support: Core
	//
	// +optional
	Namespace *Namespace `json:"namespace,omitempty"`
}

type GatewayTLSConfig struct {
    ......
    // FrontendValidation holds configuration information for validating the frontend (client).
    // Setting this field will require clients to send a client certificate
    // required for validation during the TLS handshake. In browsers this may result in a dialog appearing 
    // that requests a user to specify the client certificate.
    // The maximum depth of a certificate chain accepted in verification is Implementation specific.
    FrontendValidation *FrontendTLSValidation `json:"frontendValidation,omitempty"`
}

// FrontendTLSValidation holds configuration information that can be used to validate
// the frontend initiating the TLS connection
type FrontendTLSValidation struct {
    // CACertificateRefs contains one or more references to
    // Kubernetes objects that contain TLS certificates of
    // the Certificate Authorities that can be used
    // as a trust anchor to validate the certificates presented by the client.
    //
    // A single CA certificate reference to a Kubernetes ConfigMap
    // has "Core" support.
    // Implementations MAY choose to support attaching multiple CA certificates to
    // a Listener, but this behavior is implementation-specific.
    //
    // Support: Core - A single reference to a Kubernetes ConfigMap
    // with the CA certificate in a key named `ca.crt`.
    //
    // Support: Implementation-specific (More than one reference, or other kinds
    // of resources).
    //
    // References to a resource in a different namespace are invalid UNLESS there
    // is a ReferenceGrant in the target namespace that allows the certificate
    // to be attached. If a ReferenceGrant does not allow this reference, the
    // "ResolvedRefs" condition MUST be set to False for this listener with the
    // "RefNotPermitted" reason.
    //
    // +kubebuilder:validation:MaxItems=8
    // +kubebuilder:validation:MinItems=1
    CACertificateRefs []ObjectReference `json:"caCertificateRefs,omitempty"`
}

```

#### YAML

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: client-validation-basic
spec:
  gatewayClassName: acme-lb
  listeners:
  - name: foo-https
    protocol: HTTPS
    port: 443
    hostname: foo.example.com
    tls:
      certificateRefs:
      - kind: Secret
        group: ""
        name: foo-example-com-cert
      frontendValidation:
        caCertificateRefs:
        - kind: ConfigMap
          group: ""
          name: foo-example-com-ca-cert
```

## Deferred

This section highlights use cases that may be covered in a future iteration of this GEP

* Using system CA certificates as the trust anchor to validate the certificates presented by the frontend client.
* Supporting an optional `subjectAltNames` field within `FrontendTLSValidation` that can be used to specify one or more alternate names to verify the subject identity in the certificate presented by the client. This field falls under Authorization, the initial focus here is on Client Authentication and will be revisited when Authorization is tackled as a whole in the project.
* Specifying the verification depth in the client certificate chain. This is being deferred because the default verification depth differs across implementations.

## Existing support in Implementations

This feature is already widely supported by implementations that conform to the Gateway API. 
The table below summarizes current support. Please feel free to add any implementations that are missing.
This GEP aims to standardize this behavior as an official part of the upstream specification.

| Implementation                      | Support                                                                                                                                                     | Granularity                  | Inline vs Policy |
|------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------|------------------|
| Acnodal EPIC                       |                                                                                                                                                             |                              |                  |
| Airlock Microgateway               | [SidecarGateway.spec.applications[].downstream.tls.clientCertificate](https://docs.airlock.com/microgateway/latest/index/api/crds/sidecar-gateway/v1alpha1/index.html#sidecargatewayspecapplicationsdownstreamtls) | Per Listener                 | Inline           |
| Amazon Elastic Kubernetes Service  |                                                                                                                                                             |                              |                  |
| Apache APISIX                      | [ApisixTls.Client.CASecret](https://apisix.apache.org/docs/ingress-controller/tutorials/mtls/#mutual-authentication)                                       | Per SNI                      |                  |
| Avi Kubernetes Operator            |                                                                                                                                                             |                              |                  |
| Azure Application Gateway for Containers | [FrontendTLSPolicy](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/api-specification-kubernetes#alb.networking.azure.io/v1.FrontendTLSPolicy) | Per Gateway & Per Listener  | Policy           |
| Cilium                             |                                                                                                                                                             |                              |                  |
| Contour                            | [HTTPProxy.Spec.VirtualHost.Tls.ClientValidation.CASecret](https://projectcontour.io/docs/v1.17.1/config/tls-termination/)                                 | Per SNI                      | Inline           |
| Easegress                          |                                                                                                                                                             |                              |                  |
| Emissary Ingress                   | [TlSContext.Spec.CASecret](https://www.getambassador.io/docs/emissary/latest/howtos/client-cert-validation)                                                | Per SNI                      | Policy           |
| Envoy Gateway                      | [ClientTrafficPolicy.Spec.TLS.ClientValidation](https://gateway.envoyproxy.io/docs/api/extension_types/#clientvalidationcontext)                           | Per Gateway & Per Listener  | Policy           |
| Flomesh Service Mesh               |                                                                                                                                                             |                              |                  |
| Gloo Gateway                       | [VirtualService.Spec.SSLConfig.SecretRef](https://docs.solo.io/gloo-edge/latest/guides/security/tls/server_tls/#configuring-downstream-mtls-in-a-virtual-service) | Per SNI                      | Inline           |
| Google Cloud Service Mesh          |                                                                                                                                                             |                              |                  |
| Google Kubernetes Engine           |                                                                                                                                                             |                              |                  |
| HAProxy Ingress                    |                                                                                                                                                             |                              |                  |
| HAProxy Kubernetes Ingress Controller | [ca-file](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/security/authentication/client-certificate-authentication/#sidebar)      | Per SNI                      | Inline           |
| HashiCorp Consul                   | [file-system-certificate](https://developer.hashicorp.com/consul/docs/north-south/api-gateway/secure-traffic/encrypt)                                      | Per Listener                 | Policy           |
| Istio                              | [Gateway.Spec.Servers.TLS.Mode](https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/#configure-a-mutual-tls-ingress-gateway)      | Per Server                  | Inline           |
| kgateway                           |                                                                                                                                                             |                              |                  |
| Kong Kubernetes Ingress Controller | [mTLS Plugin](https://docs.konghq.com/hub/kong-inc/mtls-auth/)                                                                                              | Per HTTP Proxy (Host/Port)   | Policy           |
| Kong Gateway Operator              | [mTLS Plugin](https://docs.konghq.com/hub/kong-inc/mtls-auth/)                                                                                              | Per HTTP Proxy (Host/Port)   | Policy           |
| Kuma                               |                                                                                                                                                             |                              |                  |
| Linkerd                            |                                                                                                                                                             |                              |                  |
| LiteSpeed Ingress Controller       |                                                                                                                                                             |                              |                  |
| LoxiLB                             |                                                                                                                                                             |                              |                  |
| NGINX Gateway Fabric               | [ingressMTLS](https://docs.nginx.com/nginx-ingress-controller/configuration/policy-resource/#ingressmtls)                                                  | Per Listener                 | Policy           |
| ngrok Kubernetes Operator          | [TrafficPolicy.Terminate-TLS.Config.MutualTLSCertificateAuthorities](https://ngrok.com/docs/traffic-policy/actions/terminate-tls/#configuration-reference) | Per Endpoint (Host:Port)     | Policy           |
| STUNner                            |                                                                                                                                                             |                              |                  |
| Traefik Proxy                      | [TLSOption.Spec.ClientAuth](https://doc.traefik.io/traefik/https/tls/#client-authentication-mtls)                                                          | Per EntryPoint               | Inline           |
| Tyk                                | [Enable Client Certificate](https://tyk.io/docs/basic-config-and-security/security/mutual-tls/client-mtls/#why-use-mutual-tls)                             | Per Gateway                  | Policy           |
| WSO2 APK                           | [Authentication.Spec.Default.AuthTypes.MTLS](https://apk.docs.wso2.com/en/latest/catalogs/crds/authentication_types/#dp.wso2.com/v1alpha2.MutualSSLConfig) | Per API                      | Policy           |
| Ingress-NGINX                      | [nginx.ingress.kubernetes.io/auth-tls-verify-client](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#client-certificate-authentication) | Per Ingress                  | Inline           |

## References

[TLS Handshake Protocol]: https://www.rfc-editor.org/rfc/rfc5246#section-7.4
[Certificate Path Validation]: https://www.rfc-editor.org/rfc/rfc5280#section-6
[GatewayTLSConfig]: /references/spec/#gateway.networking.k8s.io/v1.GatewayTLSConfig
[BackendTLSPolicy]: ../../api-types/backendtlspolicy.md
[TLS Configuration GEP]: ../gep-2907/index.md
[Gateway API TLS Use Cases]: https://docs.google.com/document/d/17sctu2uMJtHmJTGtBi_awGB0YzoCLodtR6rUNmKMCs8/edit?pli=1#heading=h.cxuq8vo8pcxm
