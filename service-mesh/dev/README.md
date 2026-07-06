# Istio Service Mesh - Dev

This directory is intentionally minimal while the service mesh setup is being rebuilt manually.

## Current Scope

* `00-namespace.yaml`: Ensures the `dev` namespace has `istio-injection: enabled`.
* `01-mtls.yaml`: Enables namespace-level mTLS in `PERMISSIVE` mode.
* `02-mtls-strict-demo.yaml`: Enables namespace-level mTLS in `STRICT` mode for demos, while leaving public ingress-facing workloads in `PERMISSIVE` mode.
* `03-destination-rule.yaml`: Configures `product` client-side traffic to use `ISTIO_MUTUAL`.
* `04-authorization-policy.yaml`: Adds basic service-to-service allow rules for the core product/search/cart/order/checkout flow, plus public OpenAPI docs access for Swagger UI.

## Manual Apply

Apply the namespace label first:

```bash
kubectl apply -f service-mesh/dev/00-namespace.yaml
```

Apply the baseline mTLS policy:

```bash
kubectl apply -f service-mesh/dev/01-mtls.yaml
```

For a STRICT mTLS demo, apply:

```bash
kubectl apply -f service-mesh/dev/02-mtls-strict-demo.yaml
```

Apply the simple DestinationRule demo:

```bash
kubectl apply -f service-mesh/dev/03-destination-rule.yaml
```

Apply the basic AuthorizationPolicy demo:

```bash
kubectl apply -f service-mesh/dev/04-authorization-policy.yaml
```

Restart workloads after enabling injection or changing sidecar-related behavior:

```bash
kubectl rollout restart deployment -n dev
```

## Verification

Check that the namespace still has sidecar injection enabled:

```bash
kubectl get ns dev --show-labels
```

Check the active mTLS policy:

```bash
kubectl get peerauthentication -n dev
```

`PERMISSIVE` is the baseline mode for normal development because it allows mesh workloads to use mTLS while keeping plaintext callers such as Nginx Ingress working during the rebuild.

`STRICT` is useful for the demo because plaintext calls to internal mesh workloads are rejected. A `DestinationRule` is not required at this step when Istio auto mTLS is enabled and both source and destination workloads have sidecars.

The DestinationRule is included as a small explicit demo for `ISTIO_MUTUAL` traffic to `product`.

The basic AuthorizationPolicy intentionally protects only internal backend services and uses real application service accounts. Public OpenAPI docs are allowed so Swagger UI can load `/v3/api-docs`. Public entry services such as `storefront-bff`, `backoffice-bff`, `swagger-ui`, and `media` are left open while ingress behavior is being tested.
