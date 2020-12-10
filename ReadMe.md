### SUMMARY ###
The following provides instructions on how to install VMware's open source Harbor Container Registry into a Kubernetes cluster along with an official CA Signed certificate for the Harbor deployment, issued by Let's Encrypt via automation through JetStack's Cert Manager.

### NOTES ###
-Step 4 is only required in newly created TKC K8s clusters  
-Step 7 installs an nginx ingress controller into the K8s cluster.  This will be used to expose the Harbor registry externally  
-Step 8 installs Cert Manager into the K8s cluster. Cert Manager is a certificate controller that automates the certificate issuance and assignment process in a K8s cluster.  It requires certificate "issuers", in this case we're using LetsEncrypt as the CA Authority which will issue a 90 day cert.  
-Step 9 Configures the "issuer" which is LetsEncrypt.  In Cert Manager documentation the issuer is referred to as ACME, but technically that is the protocol that LetsEncrypt uses to automate certificate process.  LetsEncrypt root certs are installed in just about all OS's "trusted cert" stores, so it's a real CA to your computer.  There's a process, called a "challenge" by which your ownership of the domain name must be validated.  Let's Encrypt reaches out to an public facing ACME server to complete the challenge, of which there are two types:  HTTP01 and DNS01.  The instructions below use DNS01 which is easier if you don't have a web server with a public facing IP (see acme.yaml for DNS01 challenge details).  In short, LE will use some AWS creds I've added to my cluster (as a secret in "acme.yaml") to make an API call to AWS cloud to create a DNS TXT record that ACME server will look for to validate you own the domain.  
-Step 10 and beyond install Harbor into the cluster.


### INSTRUCTIONS ###
If you're using a TKG cluster with Metallb.  If you're using vSphere with Tanzu skip to step 4)
  
1.   Create a MetalLB namespace:  
`kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml`  

2.   Apply the following yaml to deploy metallb:  
 `kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml`  
 
3.   Apply the following yaml to create a metallb secret:  
`kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"`  
  
Start here if you're using vSphere with Tanzu and have a newly created workload cluster:

4.   After authenticating for the first time to your workload cluster, create a ClusterRoleBinding and ClusterRole that uses the Privileged PSP (only necessary is a new cluster):  
`kubectl apply -f https://raw.githubusercontent.com/trevorputbrese/tkc/master/tkc-priveledged-cluster-role.yaml`    <br />

5.   Add the stable Helm repo (nginx chart is here):  
`helm repo add stable https://charts.helm.sh/stable`  

6.   Add the Bitnami repo to Helm:  
`repo add bitnami https://charts.bitnami.com/bitnami`

7.   Install nginx ingress into the cluster using Helm:  
`helm install nginx-ingress stable/nginx-ingress --set controller.config.proxy-body-size=1024m`

8.   Install Cert Manager into the cluster:  
`kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.yaml`

9.   Apply the acme.yml (it will need to be edited with your credentials) to create the secret, certificate, and issuer resource in yoru cluster:  
`kubectl apply -f acme.yml`

10.  Make sure you have a default storage class in your cluster.  If you don't:  
`kubectl annotate sc "YOUR-STORAGE-CLASS" storageclass.kubernetes.io/is-default-class=true`

11.  modify harbor-values.yml to set global.storageClass  

12.  Install Harbor using Helm:  
`helm install harbor bitnami/harbor --version 7.1.0 -f harbor-values.yml`

13.  navigate to harbor.YOUR.DOMAIN in browser
