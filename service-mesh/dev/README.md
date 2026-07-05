# Istio Service Mesh Configuration & Testing Guide - Dev Environment

This directory contains the Kubernetes manifests to configure Istio Service Mesh features for the YAS application in the `dev` namespace.

## 1. Configured Manifests
*   `00-namespace.yaml`: Defines the `dev` namespace with the `istio-injection: enabled` label to automate sidecar injection via GitOps.
*   `01-mtls.yaml`: Configures the namespace mTLS mode to `PERMISSIVE`. This allows internal microservices to use strict mTLS (via DestinationRule) while letting external Nginx Ingress Controller access endpoints like product images and Swagger API docs via plaintext.
*   `01a-destination-rule.yaml`: Explicitly configures client-side mTLS (`mode: ISTIO_MUTUAL`) for all services in the `dev` namespace to ensure encrypted service-to-service communication.
*   `02-test-client.yaml`: Deploys a simple `test-client` Pod running with the `default` service account to act as an unauthorized client.
*   `03-authorization-policy.yaml`: Enforces strict zero-trust access control policies:
    *   Allows public Ingress access to `storefront-bff`, `backoffice-bff`, `dev-swagger-ui`, and `media`.
    *   Allows public access to Swagger API docs (`/v3/api-docs`) of all services.
    *   `product` service: Allows only `storefront-bff`, `backoffice-bff`, and `search`.
    *   `search` service: Allows only `storefront-bff`.
    *   `cart` service: Allows only `storefront-bff` and `order`.
    *   `order` service: Allows only `storefront-bff` and `backoffice-bff`.
    *   `payment`, `inventory`, `tax` services: Allow only `order` (and `backoffice-bff` for inventory).
    *   Other general backend services: Allow only `storefront-bff` and `backoffice-bff`.
*   `04-retry-virtualservice.yaml`: Configures automatic request retries (3 attempts, 2s timeout) for the `cart`, `payment`, `inventory`, and `tax` services.

---

## 2. Deploying via Argo CD
Since these manifests are registered in `dev-applications.yaml`, Argo CD will automatically sync this folder.

To manually trigger a rollout restart on the VPS to ensure sidecars are injected:
```bash
kubectl rollout restart deployment -n dev
```

---

## 3. Testing Scenarios (Demo Script)

### Scenario A: Zero-Trust Authorization Policy (Access Control)

1.  **Test 1 (Authorized dependency - ALLOW):**
    Verify that the `search` service is allowed to communicate with `product` service:
    ```bash
    kubectl exec -n dev deploy/search -c search -- curl -s -o /dev/null -w "%{http_code}\n" http://product:80/actuator/health
    ```
    *Expected output:* **`200`** (or relevant app HTTP code).

2.  **Test 2 (Unauthorized - DENY):**
    Verify that the `test-client` (using the `default` service account) is blocked from calling the `product` service:
    ```bash
    kubectl exec -n dev test-client -- curl -ivs http://product:80/actuator/health
    ```
    *Expected output:* **`HTTP 403 Forbidden`** with response `RBAC: access denied` from Envoy.

3.  **Test 3 (Strict isolation - DENY):**
    Verify that `search` service is blocked from calling `cart` service (since `cart` only allows `storefront-bff` and `order`):
    ```bash
    kubectl exec -n dev deploy/search -c search -- curl -ivs http://cart:80/actuator/health
    ```
    *Expected output:* **`HTTP 403 Forbidden`** (RBAC: access denied).

---

### Scenario B: Observe mTLS and Topology in Kiali
1.  Access Kiali Dashboard via NodePort or Port-forwarding.
2.  Interact with the YAS web storefront to generate traffic.
3.  Go to Kiali **Graph**, choose **`dev`** namespace:
    *   Verify the flow: `storefront-bff` -> `product`/`cart`/`search` and `order` -> `tax`/`payment`/`inventory`.
    *   Verify that each connecting line displays a **Padlock icon** (mTLS STRICT).

---

### Scenario C: Test Auto-Retry Policy via Fault Injection
We can simulate HTTP 500 errors on the `tax` service to demonstrate that Istio automatically retries and masks transient errors:

1.  Open `04-retry-virtualservice.yaml` and **uncomment** the `DEMO FAULT INJECTION` section for the `tax` service.
2.  Apply the change (or let Argo CD sync it).
3.  Check the Envoy logs of the `order` pod or run a curl query from `order` to `tax`:
    ```bash
    kubectl exec -n dev deploy/order -c order -- curl -ivs http://tax:80/actuator/health
    ```
    *   Even though `tax` is throwing 500 errors 50% of the time, the request will succeed most of the time because Envoy automatically retries up to 3 times behind the scenes!
    *   You can verify the retries in the Envoy sidecar logs:
        ```bash
        kubectl logs -n dev deploy/order -c istio-proxy --tail=100
        ```
