# Orbit Kubernetes: architecture and boundaries

## Responsibility

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

## Declared dependencies

The following dependency declarations come from the checked-in manifests. Optional groups and development dependencies are called out separately.

### `pyproject.toml`
- `orbit-discovery>=0.1.0a1,<0.2`
- Optional `kubernetes` group: `kubernetes-asyncio>=36.1,<37`.
- Optional `dev` group: `pytest>=8,<10`, `pytest-asyncio>=0.24,<2`, `ruff>=0.8,<1`, `mypy>=1.13,<2`, `build>=1.2,<2`.

Declared dependencies do not mean that optional providers or services are bundled with this package.

## Implementation layout

Representative implementation files in this checkout:

- `src/orbit_kubernetes/__init__.py`
- `src/orbit_kubernetes/discovery.py`

## Public contract and scope

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

## Boundary rules

Keep provider SDKs, credentials, transports, and provider-specific error translation in provider adapters. Keep reusable capability contracts in the matching capability package and lifecycle orchestration in Core. Apply the relevant layer for this repository and preserve the dependency direction shown above.
