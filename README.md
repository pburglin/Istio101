# Istio101
Getting started with Istio

source: https://istio.io/docs/setup/kubernetes/prepare/platform-setup/minikube/

```bash
-- 1. setup Kubernetes

minikube start --memory=8192 --cpus=4 --kubernetes-version=v1.13.0

-- 2. setup Istio

-- download latest stable Istio release from https://github.com/istio/istio/releases/latest

curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.1.7 sh -

cd istio-1.1.7

export PATH=$PWD/bin:$PATH
-- validate we can call istioctl from any path
istioctl

-- setup NodePort since we don’t have a real load balancer
vi install/kubernetes/istio-demo.yaml
-- replace ‘LoadBalancer’ with ‘NodePort’
-- replace port for http2 from 31380 to 30080

-- install Helm Kubernetes package manager
brew install kubernetes-helm
helm init
helm repo add istio.io https://storage.googleapis.com/istio-release/releases/1.1.7/charts/

helm repo update

-- prob not needed
helm install install/kubernetes/helm/istio-init --name istio-init --namespace istio-system
helm install install/kubernetes/helm/istio --name istio --namespace istio-system

-- enable addons
vi install/kubernetes/helm/istio/values.yaml
-- enable Grafana, ServiceGraph, Kiali, Tracing
helm upgrade istio install/kubernetes/helm/istio


kubectl config use-context minikube

kubectl create namespace istio-system

-- install all Istio CRDs:
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done

kubectl apply -f install/kubernetes/istio-demo.yaml

kubectl get nodes

kubectl get svc -n istio-system

kubectl get pods -n istio-system

kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)

kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

-- K8s dashboard UI:
-- https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
-- https://github.com/kubernetes/dashboard/wiki/Creating-sample-user
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml

kubectl proxy
-- http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/service?namespace=default

-- access sample app:
-- https://istio.io/docs/tasks/traffic-management/ingress/#determining-the-ingress-ip-and-ports

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')

export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')

export INGRESS_HOST=$(minikube ip)

export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"

echo http://${GATEWAY_URL}/productpage
-- http://192.168.99.100:31380/productpage

-- not needed:
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get gateway


-- Prometheus:

-- create some traffic to sample app:
for i in {1..10}; do curl -o /dev/null -s -w "%{http_code}\n" http://${GATEWAY_URL}/productpage; done

kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090

-- open Prometheus:
http://localhost:9090


-- Kiali

KIALI_USERNAME=$(read -p 'Kiali Username: ' uval && echo -n $uval | base64)
KIALI_PASSPHRASE=$(read -sp 'Kiali Passphrase: ' pval && echo -n $pval | base64)
NAMESPACE=istio-system

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: kiali
  namespace: $NAMESPACE
  labels:
    app: kiali
type: Opaque
data:
  username: $KIALI_USERNAME
  passphrase: $KIALI_PASSPHRASE
EOF

kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001

-- open Kiali:
http://localhost:20001/kiali/console

-- generate traffic to web app
brew install watch
watch -n 0.1 curl -o /dev/null -s -w %{http_code} $GATEWAY_URL/productpage


-- Grafana
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000

-- open grafana
http://localhost:3000/d/DvXBsvZZz/istio-mesh-dashboard?orgId=1


-- tracing w/ Jaeger

kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686  &

http://localhost:16686


-- uninstall everything:
kubectl delete -f install/kubernetes/istio-demo.yaml

-- delete all Istio CRDs:
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl delete -f $i; done
```
