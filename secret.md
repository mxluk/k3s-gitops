# 1. Create the cloudflared secret
kubectl create namespace cloudflare
kubectl create secret generic cloudflared-token \
  --namespace=cloudflare \
  --from-literal=token='YOUR_CLOUDFLARE_TUNNEL_TOKEN_HERE'

# 2. Force delete and recreate ingress-nginx application
kubectl delete application ingress-nginx -n argocd

# Wait 10 seconds
sleep 10

# Force refresh root-app
kubectl patch application root-app -n argocd -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}' --type=merge


kubectl create namespace longhorn-system
kubectl create secret generic longhorn-s3-secret \
  --namespace=longhorn-system \
  --from-literal=AWS_ACCESS_KEY_ID='YOUR_ACCESS_KEY' \
  --from-literal=AWS_SECRET_ACCESS_KEY='YOUR_SECRET_KEY' \
  --from-literal=AWS_ENDPOINTS='https://nbg1.your-objectstorage.com/' \
  --from-literal=CRYPTO_KEY_PROVIDER='secret' \
  --from-literal=CRYPTO_KEY_CIPHER='aes-256-cfb' \
  --from-literal=CRYPTO_PBKDF2_ITER='10000'