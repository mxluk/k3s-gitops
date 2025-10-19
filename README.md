# k3s-gitops - GitOps Configuration

This repository manages the k3s cluster infrastructure using Argo CD.

## Prerequisites

1. k3s cluster running with Argo CD installed
2. Cloudflare Tunnel token
3. Wildcard DNS record: `*.mxluk.de` pointing to your Cloudflare Tunnel

## Initial Setup

### 1. Clone and prepare the repository

```bash
# Create the repository structure locally
mkdir -p k3s-gitops/{apps/{ingress-nginx,cloudflared,argocd-ingress},base}
cd k3s-gitops

# Copy all the artifact files into this structure
# (root-application.yaml, base/namespaces.yaml, apps/*/*)

# Initialize git
git init
git add .
git commit -m "Initial GitOps structure"

# Create GitHub repo and push
gh repo create k3s-gitops --private --source=. --remote=origin --push
# Or manually create on GitHub and:
# git remote add origin https://github.com/mxluk/k3s-gitops.git
# git push -u origin main
```

### 2. Update repository URLs

Replace `https://github.com/mxluk/k3s-gitops` in:
- `root-application.yaml`
- `apps/cloudflared/application.yaml`
- `apps/argocd-ingress/application.yaml`

### 3. Add your Cloudflare Tunnel token

Edit `apps/cloudflared/secret.yaml` and replace `YOUR_CLOUDFLARE_TUNNEL_TOKEN_HERE` with your actual token.

**Better: Encrypt with SOPS** (optional but recommended):

```bash
# Install SOPS and age
brew install sops age

# Generate an age key
age-keygen -o age-key.txt
# Save the public key that's printed

# Create .sops.yaml in repo root
cat > .sops.yaml <<EOF
creation_rules:
  - path_regex: .*secret.*\.yaml$
    age: age1... # your age public key here
EOF

# Encrypt the secret
sops --encrypt --in-place apps/cloudflared/secret.yaml

# Add to .gitignore
echo "age-key.txt" >> .gitignore

# Commit
git add .
git commit -m "Add Cloudflare secret (encrypted)"
git push
```

### 4. Configure Cloudflare Tunnel

In your Cloudflare Zero Trust dashboard:

1. Go to your tunnel settings
2. Add a public hostname:
   - **Subdomain**: `*`
   - **Domain**: `mxluk.de`
   - **Service**: `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local:80`

This routes all `*.mxluk.de` traffic to your NGINX ingress controller.

### 5. Apply the root application

```bash
# Make sure you're connected to the cluster
export KUBECONFIG=~/.kube/k3s-bootstrap/config

# Apply the root app (this bootstraps everything)
kubectl apply -f root-application.yaml

# Watch Argo CD sync everything
kubectl get applications -n argocd -w
```

### 6. Wait for everything to deploy

```bash
# Watch ingress-nginx
kubectl get pods -n ingress-nginx -w

# Watch cloudflared
kubectl get pods -n cloudflare -w

# Check Argo CD is accessible
curl -k https://argocd.mxluk.de
```

### 7. Access Argo CD

Once deployed, access Argo CD at: **https://argocd.mxluk.de**

Get the admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

## What This Sets Up

1. **NGINX Ingress Controller** - Routes HTTP/HTTPS traffic within the cluster
2. **Cloudflare Tunnel** - Securely exposes the cluster to the internet via `*.mxluk.de`
3. **Argo CD UI** - Accessible at `argocd.mxluk.de` with HTTPS

## Adding More Applications

To add new applications, create a new directory in `apps/` with an `application.yaml` pointing to your manifests or Helm chart. Argo CD will automatically pick it up.

Example:
```bash
mkdir -p apps/my-app
# Add manifests to apps/my-app/
git add apps/my-app
git commit -m "Add my-app"
git push
# Argo CD syncs automatically
```

## Troubleshooting

### Argo CD not syncing
```bash
kubectl get applications -n argocd
kubectl describe application root-app -n argocd
```

### Cloudflared not connecting
```bash
kubectl logs -n cloudflare deployment/cloudflared
# Check the token is correct in the secret
kubectl get secret cloudflared-token -n cloudflare -o yaml
```

### Ingress not working
```bash
kubectl get ingress -A
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```