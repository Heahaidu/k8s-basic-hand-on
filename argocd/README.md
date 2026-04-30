# Bước 1: Xóa password cũ trong argocd-secret
kubectl -n argocd patch secret argocd-secret \
  -p '{"data": {"admin.password": null, "admin.passwordMtime": null}}'
  
# Bước 2: Xóa initial secret nếu còn
kubectl -n argocd delete secret argocd-initial-admin-secret \
  --ignore-not-found

# Bước 3: Restart ArgoCD server
kubectl -n argocd rollout restart deployment argocd-server

# Bước 4: Chờ ready
kubectl -n argocd rollout status deployment argocd-server

# Bước 5: Lấy password mới được tạo tự động
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo