# Route Trait Design

The main idea of route trait is to let users have an entrypoint to visit their App.

In k8s world, if you want to do so, you have to understand K8s [Serivce](https://kubernetes.io/docs/concepts/services-networking/service/)
, [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).
It's not easy to get all of these things work well.

The route trait will help you setup service, ingress automatically according your workload, along with the mTLS enabled.

The schema is also clean and easy to understand.

```go

// RouteSpec defines the desired state of Route
type RouteSpec struct {
	// WorkloadReference to the workload whose metrics needs to be exposed
	WorkloadReference runtimev1alpha1.TypedReference `json:"workloadRef,omitempty"`

	// Host is the host of the route
	Host string `json:"host"`
    
    // TLS indicate route trait will create SSL secret using cert-manager with specified issuer
	// If this is nil, route trait will use a selfsigned issuer
	TLS *TLS `json:"tls,omitempty"`

	// Rules contain multiple rules of route
	Rules []Rule `json:"rules"`
   
    // Provider indicate which ingress controller implementation the route trait will use, by default it's nginx-ingress
	Provider string `json:"provider,omitempty"`
}

// Rule defines to route rule
type Rule struct {
	// Path is location Path, default for "/"
	Path string `json:"path,omitempty"`

	
	// RewriteTarget will rewrite request from Path to RewriteTarget path.
	RewriteTarget string `json:"rewriteTarget,omitempty"`

	// CustomHeaders pass a custom list of headers to the backend service.
	CustomHeaders map[string]string `json:"customHeaders,omitempty"`

	// DefaultBackend will become the ingress default backend if the backend is not available
	DefaultBackend runtimev1alpha1.TypedReference `json:"defaultBackend,omitempty"`

	// Backend indicate how to connect backend service
	// If it's nil, will auto discovery
	Backend *Backend `json:"backend,omitempty"`
}

type TLS struct {
	IssuerName string `json:"issuerName,omitempty"`

	// Type indicate the issuer is ClusterIssuer or NamespaceIssuer
	Type IssuerType `json:"type,omitempty"`
}

type IssuerType string

const (
	ClusterIssuer   IssuerType = "ClusterIssuer"
	NamespaceIssuer IssuerType = "Issuer"
)

// Route will automatically discover podSpec and label for BackendService.
// If BackendService is already set, discovery won't work.
// If BackendService is not set, the discovery mechanism will work.
type Backend struct {
	// ReadTimeout used for setting read timeout duration for backend service, the unit is second.
	ReadTimeout int `json:"readTimeout,omitempty"`
	// SendTimeout used for setting send timeout duration for backend service, the unit is second.
	SendTimeout int `json:"sendTimeout,omitempty"`
	// BackendService specifies the backend K8s service and port
	BackendService *BackendServiceRef `json:"backendService,omitempty"`
}

type BackendServiceRef struct {
	// Port allow you direct specify backend service port.
	Port intstr.IntOrString `json:"port,omitempty"`
	// ServiceName allow you direct specify K8s service for backend service.
	ServiceName string `json:"serviceName,omitempty"`
}
```

Route Trait specifies a target workload by using `workloadRef`, in OAM system, this field will be filled automatically
by OAM runtime.

Besides `workloadRef`, one Route will have only one `host` and many rules. `host` is actually your app's visiting URL.
It's required and will be used to generate mTLS secrets.

Route Trait designed to be compatible with different ingress controller implementations, the `provider` field will allow
you to give a specified ingress controller type. Currently, only nginx-ingress is supported. 

The `tls` field allow you to specify a TLS for this route with an IssuerName, the IssuerName pointing to an Issuer Object
created by cert-manager. Cert-manager and ingress controller will handle certificate creation and binding.
If not specified, mTLS was disabled and you can only visit by http.

If no rule specified, route trait will create one rule automatically and match with the port.

For every rule, we will create an ingress. In the rule, you could specify `path`, `rewriteTarget`, `customHeaders`
and `defaultBackend`. All rules will using the same `tls`, `host` and `provider`.

`defaultBackend` will become the ingress default backend with K8s Object(apiVersion/kind/name).

`backend` of the rule is completely optional.
 
If backendService is specified, it will use it as backend of this rule. If not,
the route trait can automatically discovery backend settings from workload.

## Discovery mechanism

1. Check ChildResource of the workload, if there already has an existing K8s service match the Backend port, use it.
2. If there's no k8s service, this means we need to create one. In order to create K8s service, we need two information
`Container Port` and `Pod SelectorLabels`.
  - 2.1 Use [`PodSpecable` mechanism](https://github.com/crossplane/oam-kubernetes-runtime/blob/master/design/one-pager-podspecable-workload.md),
route trait will check `WorkloadDefinition` for podSpec field, with the `podSpec` field, we can easily find the container port.
      * If podSpecPath` is specified, we will use the workload labels as `Pod SelectorLabels`.
      * If `workload.oam.dev/podspecable: true` but no `podSpecPath`, will use `spec.Template` as `PodTemplate`, which means
      we can get `Pod SelectorLabels` from `spec.Template.Labels`.
  - 2.2 Use ChildResource: If No `PodSpecable` mechanism found in workload, we will continue discovery child resources of workload. If there
  is a valid `PodTemplate` structure in child resource, we will regard it as discovery target, use the same strategy like
  `workload.oam.dev/podspecable: true` but no `podSpecPath`.

