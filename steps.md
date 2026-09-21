
# Step 1 — Set up kubeconfig contexts for both clusters

## Add both clusters to your kubeconfig
aws eks update-kubeconfig \
  --name cluster-a \
  --region us-east-1 \
  --alias cluster-a

aws eks update-kubeconfig \
  --name cluster-b \
  --region us-east-1 \
  --alias cluster-b

## Verify
kubectl config get-contexts
## You should see both cluster-a and cluster-b


# Step 2 — Log in to ArgoCD on Cluster A

# Switch to cluster A context
kubectl config use-context cluster-a

# Get the ArgoCD server endpoint
# If using LoadBalancer:
ARGOCD_SERVER=$(kubectl get svc argocd-server \
  -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Get initial admin password
ARGOCD_PASSWORD=$(kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath='{.data.password}' | base64 -d)

# Login via CLI
argocd login "$ARGOCD_SERVER" \
  --username admin \
  --password "$ARGOCD_PASSWORD" \
  --insecure   # Remove if you have TLS configured

# Verify login
argocd account get-user-info


# Step 3 — Register Cluster B with ArgoCD on Cluster A

# Register Cluster B - argocd CLI uses your kubeconfig context
argocd cluster add cluster-b \
  --name cluster-b \
  --server "$ARGOCD_SERVER" \
  --insecure   # Remove if TLS configured

# You will be prompted to confirm - type 'y'


# Verify the cluster was registered

argocd cluster list

# Switch to cluster B to verify
kubectl config use-context cluster-b

kubectl get serviceaccount argocd-manager -n kube-system
kubectl get clusterrolebinding argocd-manager-role-binding


# Step 4 - Create the ArgoCD Application targeting Cluster B

# Get Cluster B's server URL from ArgoCD
CLUSTER_B_SERVER=$(argocd cluster list \
  --output json | \
  jq -r '.[] | select(.name=="cluster-b") | .server')

echo "$CLUSTER_B_SERVER"
# https://xxxx.gr7.us-east-1.eks.amazonaws.com


argocd app create falcon-operator-cluster-b \
  --repo https://github.com/your-org/gitops-repo \
  --path clusters/cluster-b/falcon-operator \
  --dest-server "$CLUSTER_B_SERVER" \
  --dest-namespace falcon-operator \
  --revision main \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true



# Step 5 — Trigger sync and verify

# Check application status
argocd app get falcon-operator-cluster-b

# Manually sync if needed (automated sync may take up to 3 mins)
argocd app sync falcon-operator-cluster-b

# Watch sync progress
argocd app wait falcon-operator-cluster-b --sync

# Check what was deployed on Cluster B
kubectl config use-context cluster-b
kubectl get all -n falcon-operator
