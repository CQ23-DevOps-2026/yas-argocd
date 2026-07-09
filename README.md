# Kịch bản kiểm thử và cấu hìnháp dụng với từng yêu cầu.
# 1. Kịch Bản Kiểm Thử Mutual TLS (mTLS)

Có, nên bổ sung **pod không có sidecar** để gửi request plaintext. Đây là test case rất đẹp cho mTLS `STRICT`.

Ý tưởng:

- `curl-search` hoặc `deployment/search` có sidecar → gọi `product` được.
- Pod `curl-plain` **không có sidecar** → gửi plaintext vào `product` → bị mTLS `STRICT` chặn.

Dưới đây là test plan đã bổ sung.

**Test Plan: Kiểm Thử mTLS STRICT Trong Istio Service Mesh**

**1. Mục Tiêu**

Kiểm tra cơ chế mTLS của Istio trong namespace `dev`.

Mục tiêu kiểm thử:

- Xác nhận workload trong namespace `dev` có Istio sidecar.
- Apply mTLS `STRICT`.
- Kiểm tra service trong mesh gọi nhau thành công bằng mTLS.
- Tạo pod không có sidecar để gửi plaintext request.
- Xác nhận plaintext request bị chặn khi mTLS `STRICT` đang bật.
- Sau demo, rollback về `PERMISSIVE`.

**2. File Sử Dụng**

```text
service-mesh/dev/00-namespace.yaml
service-mesh/dev/01-mtls.yaml
service-mesh/dev/02-mtls-strict-demo.yaml
```

| File | Mục đích |
|---|---|
| `00-namespace.yaml` | Bật Istio injection cho namespace `dev` |
| `01-mtls.yaml` | Cấu hình mTLS `PERMISSIVE` |
| `02-mtls-strict-demo.yaml` | Cấu hình mTLS `STRICT` để demo |

**3. Chuẩn Bị**

Kiểm tra namespace:

```bash
kubectl get ns dev --show-labels
```

Kết quả mong muốn:

```text
istio-injection=enabled
```

Kiểm tra pod có sidecar:

```bash
kubectl get pods -n dev
```

Kết quả mong muốn với các service trong mesh:

```text
2/2
```

**4. Apply mTLS STRICT**

```bash
kubectl apply -f service-mesh/dev/02-mtls-strict-demo.yaml
```

Kiểm tra:

```bash
kubectl get peerauthentication -n dev
```

Kiểm tra chi tiết:

```bash
kubectl get peerauthentication default -n dev -o yaml
```

Kết quả mong muốn:

```yaml
mtls:
  mode: STRICT
```

**5. Test Case 1: Service Có Sidecar Gọi Nhau Thành Công**

Mục tiêu: kiểm tra service trong mesh vẫn gọi nhau được khi bật mTLS `STRICT`.

Lệnh:

```bash
kubectl exec deployment/search -n dev -- wget -qO- http://product/product/storefront/products/featured
```

Hoặc:

```bash
kubectl exec -n dev curl-search -c curl-search -- curl -i http://product/product/storefront/products/featured
```

Kết quả mong muốn:

- Request thành công, trả về dữ liệu từ `product`.
- Không bị lỗi mTLS.
- Không bị `403 Forbidden`.

Ý nghĩa:

> `search` và `product` đều có sidecar Istio nên traffic giữa hai service được mã hóa bằng mTLS và được chấp nhận.

**6. Test Case 2: Tạo Pod Plaintext Không Có Sidecar**

Tạo pod curl không có sidecar bằng annotation:

```bash
kubectl run curl-plain \
  -n dev \
  --image=curlimages/curl \
  --restart=Never \
  --overrides='{
    "metadata": {
      "annotations": {
        "sidecar.istio.io/inject": "false"
      }
    }
  }' \
  -- sleep 172800
```

Kiểm tra pod:

```bash
kubectl get pod curl-plain -n dev
```

Kết quả mong muốn:

```text
curl-plain   1/1   Running
```

`1/1` nghĩa là pod **không có Istio sidecar**, nên request gửi ra là plaintext.

**7. Test Case 3: Plaintext Request Bị Chặn Bởi mTLS STRICT**

Từ pod `curl-plain`, gọi vào `product`:

```bash
kubectl exec -n dev curl-plain -- curl -i http://product/product/storefront/products/featured
```

Kết quả mong muốn có thể là một trong các dạng sau:

```text
Recv failure: Connection reset by peer
```

hoặc:

```text
Empty reply from server
```

hoặc:

```text
upstream connect error
```

hoặc request timeout.

Ý nghĩa:

> Pod `curl-plain` không có sidecar Istio nên không thể thiết lập mTLS với `product`. Khi namespace `dev` đang ở chế độ mTLS `STRICT`, request plaintext bị từ chối.

**8. Test Case 4: Rollback Về PERMISSIVE Và Test Lại Plaintext**

Apply lại mTLS `PERMISSIVE`:

```bash
kubectl apply -f service-mesh/dev/01-mtls.yaml
```

Kiểm tra:

```bash
kubectl get peerauthentication default -n dev -o yaml
```

Kết quả mong muốn:

```yaml
mtls:
  mode: PERMISSIVE
```

Gọi lại từ pod plaintext:

```bash
kubectl exec -n dev curl-plain -- curl -i http://product/product/storefront/products/featured
```

Kết quả mong muốn:

- Request không còn bị lỗi mTLS.
- Có thể trả về dữ liệu, `200`, `404`, hoặc lỗi ứng dụng.
- Quan trọng là không còn lỗi connection reset/empty reply do mTLS STRICT.

Ý nghĩa:

> Khi chuyển về `PERMISSIVE`, service chấp nhận cả mTLS và plaintext, nên pod không có sidecar có thể gửi request vào service.

**9. Dọn Pod Test Sau Khi Demo**

Nếu muốn dọn sau demo:

```bash
kubectl delete pod curl-plain -n dev
```

**10. Bảng Tổng Hợp**

| Test case | Lệnh | Kết quả mong muốn |
|---|---|---|
| Kiểm tra sidecar | `kubectl get pods -n dev` | Pod mesh có `2/2` |
| Apply STRICT | `kubectl apply -f service-mesh/dev/02-mtls-strict-demo.yaml` | mTLS chuyển sang `STRICT` |
| Mesh gọi mesh | `search` gọi `product` | Request thành công |
| Tạo plaintext pod | `kubectl run curl-plain ... inject=false` | Pod chạy `1/1` |
| Plaintext gọi product khi STRICT | `curl-plain` gọi `product` | Bị reset, timeout hoặc empty reply |
| Rollback PERMISSIVE | `kubectl apply -f service-mesh/dev/01-mtls.yaml` | mTLS về `PERMISSIVE` |
| Plaintext gọi lại khi PERMISSIVE | `curl-plain` gọi `product` | Request đi qua, không lỗi mTLS |

**11. Kết Luận**

Kịch bản kiểm thử chứng minh mTLS `STRICT` của Istio hoạt động đúng. Khi các service đều có sidecar, traffic service-to-service vẫn hoạt động vì được mã hóa bằng mTLS. Ngược lại, pod `curl-plain` không có sidecar gửi request plaintext vào `product` sẽ bị chặn khi namespace ở chế độ `STRICT`. Sau khi rollback về `PERMISSIVE`, plaintext request có thể đi qua, phù hợp với môi trường phát triển.

# 2. Kịch Bản Kiểm Thử Authorization Policy (S2S)

Mục tiêu của kịch bản này là kiểm tra Istio `AuthorizationPolicy` có kiểm soát đúng giao tiếp service-to-service hay không. Cụ thể, kiểm thử service `search` được phép gọi vào service `product`, trong khi service không được cấp quyền như `payment` sẽ bị chặn.

**2.1. Điều kiện chuẩn bị**

Các cấu hình Istio đã được apply trong namespace `dev`:

```bash
kubectl apply -f service-mesh/dev/00-namespace.yaml
kubectl apply -f service-mesh/dev/02-mtls-strict-demo.yaml
kubectl apply -f service-mesh/dev/04-authorization-policy.yaml
```

Kiểm tra các policy:

```bash
kubectl get peerauthentication -n dev
kubectl get authorizationpolicy -n dev
```

Kiểm tra các pod đã có sidecar Istio:

```bash
kubectl get pods -n dev
```

Các pod trong mesh nên có trạng thái container dạng `2/2`.

**2.2. Chính sách truy cập của service `product`**

Trong file `04-authorization-policy.yaml`, service `product` được cấu hình policy `product-access`.

Policy này chỉ cho phép các service account sau gọi vào `product`:

```text
storefront-bff
backoffice-bff
search
cart
order
```

Vì vậy, kịch bản kiểm thử sẽ gồm hai trường hợp:

| Trường hợp | Service account | Gọi tới service | Kết quả mong muốn |
|---|---|---|---|
| Được phép | `search` | `product` | Request được chấp nhận |
| Bị chặn | `payment` | `product` | `403 Forbidden` |

**2.3. Test case 1: `search` được phép gọi `product`**

Sử dụng deployment `search` để gọi API của service `product`:

```bash
kubectl exec deployment/search -n dev -- wget -qO- http://product/product/storefront/products/featured
```

Hoặc dùng pod curl với service account `search`:

```bash
kubectl run curl-search \
  -n dev \
  --image=curlimages/curl \
  --restart=Never \
  --overrides='{"spec":{"serviceAccountName":"search"}}' \
  -- sleep 172800
```

Gọi API:

```bash
kubectl exec -n dev curl-search -c curl-search -- curl -i http://product/product/storefront/products/featured
```

Kết quả mong muốn:

```text
HTTP/1.1 200 OK
```

Hoặc trả về dữ liệu JSON danh sách sản phẩm nổi bật.

Nếu endpoint trả `404` hoặc lỗi ứng dụng nhưng không phải `403 RBAC`, vẫn chứng minh request đã được Istio cho đi qua. Tuy nhiên để demo đẹp, nên dùng endpoint thật:

```text
/product/storefront/products/featured
```

**Giải thích:**

Service account `search` nằm trong danh sách `principals` được phép truy cập `product`, nên request từ `search` sang `product` được Istio cho phép.

**2.4. Test case 2: `payment` không được phép gọi `product`**

Tạo pod curl dùng service account `payment`:

```bash
kubectl run curl-payment \
  -n dev \
  --image=curlimages/curl \
  --restart=Never \
  --overrides='{"spec":{"serviceAccountName":"payment"}}' \
  -- sleep 172800
```

Kiểm tra pod:

```bash
kubectl get pod curl-payment -n dev
```

Gọi cùng endpoint của service `product`:

```bash
kubectl exec -n dev curl-payment -c curl-payment -- curl -s -i http://product/product/storefront/products/featured
```

Kết quả mong muốn:

```text
HTTP/1.1 403 Forbidden
RBAC: access denied
```

**Giải thích:**

Service account `payment` không nằm trong danh sách `principals` được phép gọi service `product`. Vì vậy Istio chặn request trước khi request đi vào ứng dụng `product`.

**2.5. Kiểm tra service account của pod test**

Nếu kết quả không đúng như mong muốn, kiểm tra pod đang dùng service account nào:

```bash
kubectl get pod curl-search -n dev -o jsonpath='{.spec.serviceAccountName}'
```

```bash
kubectl get pod curl-payment -n dev -o jsonpath='{.spec.serviceAccountName}'
```

Kết quả mong muốn:

```text
search
```

và:

```text
payment
```

**2.6. Kết quả đánh giá**

| Test case | Lệnh kiểm thử | Kết quả | Đánh giá |
|---|---|---|---|
| `search` gọi `product` | `kubectl exec deployment/search -n dev -- wget -qO- http://product/product/storefront/products/featured` | Trả về dữ liệu hoặc HTTP success | Đạt |
| `payment` gọi `product` | `kubectl exec -n dev curl-payment -c curl-payment -- curl -i http://product/product/storefront/products/featured` | `403 Forbidden`, `RBAC: access denied` | Đạt |

**2.7. Kết luận**

Kịch bản kiểm thử cho thấy Istio `AuthorizationPolicy` đã hoạt động đúng. Service `search` được phép gọi vào `product` vì service account `search` nằm trong danh sách được cấp quyền. Ngược lại, service account `payment` không được khai báo trong policy nên request bị chặn với lỗi `RBAC: access denied`. Điều này chứng minh hệ thống có thể kiểm soát giao tiếp service-to-service dựa trên identity của workload trong Kubernetes.

# 3. Kiểm thử retry policy với httpbin

## **3.1 Cấu hình retry**

File sử dụng:

```text
service-mesh/dev/05-virtual-service.yaml
```

Cấu hình chính:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: httpbin-retry
  namespace: dev
spec:
  hosts:
    - "httpbin"
  http:
    - route:
        - destination:
            host: httpbin.dev.svc.cluster.local
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: "5xx,connect-failure,refused-stream"
```

Ý nghĩa:

- `attempts: 3`: Istio thử gửi request tối đa 3 lần.
- `perTryTimeout: 2s`: mỗi lần thử có timeout 2 giây.
- `retryOn: 5xx,connect-failure,refused-stream`: retry khi gặp lỗi server hoặc lỗi kết nối.

## **3.2. Điều kiện trước khi test**

Apply service `httpbin`:

```bash
kubectl apply -f service-mesh/dev/06-httpbin.yaml
```

Apply cấu hình retry:

```bash
kubectl apply -f service-mesh/dev/05-virtual-service.yaml
```

Kiểm tra tài nguyên:

```bash
kubectl get pod,svc -n dev | grep httpbin
kubectl get virtualservice httpbin-retry -n dev
```

Kiểm tra pod client:

```bash
kubectl get pod curl-search -n dev
```

Pod client cần chạy trong namespace `dev` và có sidecar Istio, trạng thái thường là `2/2`.

## **3.3. Test case 1: Kiểm tra httpbin hoạt động bình thường**

Lệnh kiểm thử:

```bash
kubectl exec -n dev curl-search -c curl-search -- curl -i http://httpbin/get
```

Kết quả mong muốn:

```text
HTTP/1.1 200 OK
```

Ý nghĩa:

> Service `httpbin` hoạt động bình thường và client có thể gọi tới service này trong mesh.

## **3.4. Test case 2: Kiểm tra retry khi gặp lỗi 5xx**

Mở terminal theo dõi log của `httpbin`:

```bash
kubectl logs -n dev deployment/httpbin -f
```

Ở terminal khác, gửi request tới endpoint trả lỗi `500`:

```bash
kubectl exec -n dev curl-search -c curl-search -- curl -i http://httpbin/status/500
```

Hoặc dùng lệnh gọn hơn để lấy HTTP code:

```bash
kubectl exec -n dev curl-search -c curl-search -- curl -s -o /dev/null -w "HTTP_CODE=%{http_code}\n" http://httpbin/status/500
```

Kết quả mong muốn ở client:

```text
HTTP_CODE=500
```

Kết quả mong muốn ở log `httpbin`:

```text
GET /status/500
GET /status/500
GET /status/500
```

Ý nghĩa:

> Mặc dù kết quả cuối cùng vẫn là `500` do endpoint `/status/500` luôn trả lỗi, log của `httpbin` cho thấy request được gửi nhiều lần. Điều này chứng minh Istio đã thực hiện retry theo cấu hình `VirtualService`.

## **3.5. Test case 3: Kiểm tra cấu hình VirtualService**

Lệnh kiểm tra:

```bash
kubectl get virtualservice httpbin-retry -n dev -o yaml
```

Kết quả cần có:

```yaml
retries:
  attempts: 3
  perTryTimeout: 2s
  retryOn: 5xx,connect-failure,refused-stream
```

Ý nghĩa:

> Cấu hình retry đã được áp dụng cho traffic đến service `httpbin`.

## **3.6. Bảng tổng hợp kết quả**

| Test case | Lệnh kiểm thử | Kết quả mong muốn | Đánh giá |
|---|---|---|---|
| Kiểm tra httpbin hoạt động | `curl http://httpbin/get` | `HTTP/1.1 200 OK` | Đạt |
| Kiểm tra retry lỗi 5xx | `curl http://httpbin/status/500` | Client nhận `500`, log ghi nhiều request | Đạt |
| Kiểm tra VirtualService | `kubectl get virtualservice httpbin-retry -o yaml` | Có `attempts: 3`, `retryOn: 5xx` | Đạt |

## **3.7. Kết luận**

Kịch bản kiểm thử cho thấy cơ chế retry của Istio hoạt động đúng. Khi service `httpbin` trả về lỗi `500`, Istio Envoy proxy tự động retry request theo cấu hình trong `VirtualService`. Kết quả cuối cùng vẫn là `500` vì endpoint luôn trả lỗi, nhưng log của service cho thấy có nhiều request được gửi đến `httpbin`, chứng minh retry đã được kích hoạt.