# argocd
** Run on linux laptop - needs minikube and docker **
systemctl start docker
minikube start --memory=8192 --cpus=3  --driver=docker -p gitops
minikube addons enable metrics-server -p gitops
minikube dashboard &
k config set-context minikube
k get pods
eval $(minikube docker-env -p gitops)
minikube addons enable ingress -p gitops
kubectl create namespace argocd
# Install argocd #
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
sleep 60
# Install BGD APP #
cat <<EOF > bgd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgd-app
  namespace: argocd
spec:
  destination:
    namespace: bgd
    server: https://kubernetes.default.svc
  project: default
  source:
    path: apps/bgd/overlays/bgd
    repoURL: https://github.com/redhat-developer-demos/openshift-gitops-examples
    targetRevision: minikube
  syncPolicy:
    automated:
      prune: true
      selfHeal: false
    syncOptions:
    - CreateNamespace=true
EOF
k apply -f bgd-app.yaml
echo "$(minikube ip) bgd.devnation" | sudo tee -a /etc/hosts
google-chrome http://bgd.devnation:8080

# To access argocd GUI #
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
minikube -p gitops service list | grep argocd
argoPass=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d) ; echo $argoPass
google-chrome `minikube service list | grep "argocd-server " | awk '{print $(NF-1)}'`

#kubectl port-forward svc/argocd-server -n argocd 8080:443
