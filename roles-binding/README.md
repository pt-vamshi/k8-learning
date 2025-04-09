# Kubernetes Cert-Based User Access with Namespace RBAC

This guide explains how to set up a Kubernetes user (`john`) with certificate-based authentication, limited to a specific namespace (`dev-team`) using Role-Based Access Control (RBAC).

---

## 🛠 Prerequisites

- Minikube or any local Kubernetes cluster
- `kubectl`
- `openssl`

---
kubectl apply -f john-csr.yaml
kubectl certificate approve john-csr
kubectl get csr john-csr -o jsonpath='{.status.certificate}' | base64 -d > john.crt



kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml

kubectl config set-credentials john \
  --client-certificate=john.crt \
  --client-key=john.key \
  --embed-certs=true

kubectl config set-context john-context \
  --cluster=minikube \
  --namespace=dev-team \
  --user=john

kubectl config use-context john-context


kubectl get pods -n dev-team         # ✅ should work
kubectl get pods -n kube-system      # ❌ should be forbidden


kubectl config use-context minikube


