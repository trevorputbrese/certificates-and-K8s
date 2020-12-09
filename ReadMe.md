The following provides instructions on how to install VMware's open source Harbor Container Registry into a Kubernetes cluster along with an official CA Signed certificate for the Harbor deployment, issued by Let's Encrypt.

Notes:  
Step 4 is only required in TKC based K8s clusters
Step 7 installs an nginx ingress controller into the K8s cluster.  This will be used to expose the Harbor registry externally
Step 8 installs Cert Manager into the K8s cluster.


w/tkg use Metallb for v7wk8s skip to step 4
  
1.   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
2.   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
3.   kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
  
Start w/ TKG-S

4.   kubectl apply -f tanzu-cr.yml
5.   helm repo add stable https://charts.helm.sh/stable
6.   helm repo add bitnami https://charts.bitnami.com/bitnami
7.   helm install nginx-ingress stable/nginx-ingress --set controller.config.proxy-body-size=1024m
8.   kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.yaml
9.   kubectl apply -f acme.yml
10.  kubectl annotate sc "YOUR-STORAGE-CLASS" storageclass.kubernetes.io/is-default-class=true
11.  modify harbor-values.yml to set global.storageClass
12.  helm install harbor bitnami/harbor --version 7.1.0 -f harbor-values.yml
13.  navigate to harbor.YOUR.DOMAIN in browser
