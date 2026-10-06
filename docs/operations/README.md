# Orbit Kubernetes: operations and security

This guide organizes runtime behavior documented by the package. It does not certify production readiness. Verify provider/client versions, permissions, transport security, limits, and failure behavior in the target environment before release.

## Configuration surface

Environment names found in the package README:

The package README does not name `ORBIT_*` variables. Use its typed constructors and application configuration, and confirm exact runtime inputs in the implementation before deployment.

Use the package README's constructor and deployment examples. Store credentials in a secret manager and avoid logging credentials, raw provider errors, request data, or opaque cursors.

## Lifecycle, failure behavior, and limits

`orbit-kubernetes` is the Kubernetes provider adapter for Orbit capability packages. The first
adapter implements `orbit_discovery.ServiceDiscovery`; Core and `orbit-discovery` remain free of
Kubernetes SDK dependencies.

This package is pre-alpha and supports Python 3.11–3.14. The Kubernetes client is an optional
extra:

```bash
python -m pip install 'orbit-kubernetes[kubernetes]'
```

Orbit also provides a separately installable Go process-plugin variant, `orbit-kubernetes-go`, for
deployments that prefer a standalone Go provider process. Both variants implement the same
`orbit.discovery.v1` contract. Select one explicitly in application wiring: the Python variant
registers `KubernetesServiceDiscovery`; the Go variant is supervised by Core and exposed through
`orbit-discovery[process]`'s `ProcessServiceDiscovery`. Core remains Python in both cases.

## Behavior and limits

- Reads Kubernetes `discovery.k8s.io/v1` EndpointSlices, not the legacy Endpoints API. EndpointSlice
  is Kubernetes' scalable backend representation and supports IPv4 and IPv6.
- A configured named Service port is required. Only TCP endpoints with that port name are returned.
- Explicitly unready endpoints are filtered. Kubernetes defines an unset `ready` condition as
  ready, so unset values remain eligible.
- Results are deterministic immutable `ServiceInstance` snapshots, deduplicated by host and port.
  The adapter does not choose an instance, connect to it, or perform health checks.
- Refreshes are coalesced per service using bounded lock striping. The cache, API request timeout,
  pages, page size, total endpoints, and cache capacity have finite limits.
- API failures become `DiscoveryError` without copying provider error details or credentials into
  messages. Task cancellation propagates. A missing Service resolves to an empty tuple.
- This adapter does not watch EndpointSlices. Lookups reflect the Kubernetes API at the time of
  the last cache refresh; choose a short cache TTL to limit staleness.

The Kubernetes identity should have read-only `list` permission for EndpointSlices in the selected
namespace. Avoid cluster-wide roles for this adapter. Network egress policy still needs to validate
discovered addresses before an application connects to them; discovery results are not an SSRF
defense.

## Production validation

Validate startup/shutdown cleanup, timeout and cancellation behavior, concurrency and payload bounds where applicable, secret rotation and least-privilege access, data durability, backup/restore, and failover against the selected provider. Do not infer distributed or durable guarantees from an in-process API or fake-client tests.
