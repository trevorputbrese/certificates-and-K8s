### SUMMARY ###
The following provides instructions on how to install VMware's open source Harbor Container Registry into a Kubernetes cluster along with an official CA Signed certificate for the Harbor deployment, issued by Let's Encrypt via automation through JetStack's Cert Manager.

### NOTES ###
-Step 4 is only required in newly created TKC K8s clusters  
-Step 7 installs an nginx ingress controller into the K8s cluster.  This will be used to expose the Harbor registry externally  
-Step 8 installs Cert Manager into the K8s cluster. Cert Manager is a certificate controller that automates the certificate issuance and assignment process in a K8s cluster.  It requires certificate "issuers", in this case we're using LetsEncrypt as the CA Authority which will issue a 90 day cert.  
-Step 9 Configures the "issuer" which is LetsEncrypt.  In Cert Manager documentation the issuer is referred to as ACME, but technically that is the protocol that LetsEncrypt uses to automate certificate process.  LetsEncrypt root certs are installed in just about all OS's "trusted cert" stores, so it's a real CA to your computer.  There's a process, called a "challenge" by which your ownership of the domain name must be validated.  Let's Encrypt reaches out to an public facing ACME server to complete the challenge, of which there are two types:  HTTP01 and DNS01.  The instructions below use DNS01 which is easier if you don't have a web server with a public facing IP (see acme.yaml for DNS01 challenge details).  In short, LE will use some AWS creds I've added to my cluster (as a secret in "acme.yaml") to make an API call to AWS cloud to create a DNS TXT record that ACME server will look for to validate you own the domain.  
-Step 10 and beyond install Harbor into the cluster.


### INSTRUCTIONS ###
w/tkg use Metallb for v7wk8s skip to step 4
  
1.   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml  
2.   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml  
3.   kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"  
  
Start w/ TKG-S

4.   Run the following command in a TKGS cluster to create a ClusterRoleBinding and ClusterRole that uses the Privileged PSP:   
`kubectl apply -f https://raw.githubusercontent.com/trevorputbrese/tkc/master/tkc-priveledged-cluster-role.yaml`    
5.   helm repo add stable https://charts.helm.sh/stable  
6.   helm repo add bitnami https://charts.bitnami.com/bitnami  
7.   helm install nginx-ingress stable/nginx-ingress --set controller.config.proxy-body-size=1024m  
8.   kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.yaml  
9.   kubectl apply -f acme.yml
10.  kubectl annotate sc "YOUR-STORAGE-CLASS" storageclass.kubernetes.io/is-default-class=true
11.  modify harbor-values.yml to set global.storageClass
12.  helm install harbor bitnami/harbor --version 7.1.0 -f harbor-values.yml
13.  navigate to harbor.YOUR.DOMAIN in browser
