`helm`/`kubectl` luôn dùng **context hiện tại** trong kubeconfig — không phải biết "cluster nào" theo cách bạn nghĩ, mà là **context nào đang active**.

## Kiểm tra context hiện tại

```bash
kubectl config current-context
```

## Xem tất cả context có trong kubeconfig

```bash
kubectl config get-contexts
```

Output ví dụ:
```
CURRENT   NAME                                          CLUSTER
*         arn:aws:eks:ap-southeast-1:xxx:cluster/test    test
          arn:aws:eks:ap-southeast-1:xxx:cluster/prod    prod
```

Dấu `*` = context đang active = nơi `helm install` sẽ chạy vào.

## Chuyển context (chọn đúng cluster)

```bash
kubectl config use-context arn:aws:eks:ap-southeast-1:xxx:cluster/prod
```

Sau đó `helm install ...` sẽ apply lên `prod`.

## Cách an toàn hơn: chỉ định context trực tiếp trong lệnh helm

Tránh việc nhớ phải `switch context` trước (dễ nhầm cluster):

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --kube-context arn:aws:eks:ap-southeast-1:xxx:cluster/test
```

`kubectl` cũng hỗ trợ tương tự:
```bash
kubectl get pods --context arn:aws:eks:ap-southeast-1:xxx:cluster/prod
```

## Khi `aws eks update-kubeconfig` cho 2 cluster

Mỗi lần chạy sẽ **thêm context mới** vào `~/.kube/config` (không xóa context cũ), và **set context đó thành current** — nên nếu bạn vừa update-kubeconfig cho cluster B, context active sẽ là B, dù trước đó đang là A.

→ **Luôn check `kubectl config current-context` trước khi `helm install`** nếu làm việc với nhiều cluster, hoặc dùng `--kube-context` để chỉ định rõ ràng, tránh deploy nhầm cluster.