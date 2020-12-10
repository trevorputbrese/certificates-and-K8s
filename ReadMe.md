### SUMMARY ###
The following provides instructions on how to install VMware's open source Harbor Container Registry into a Kubernetes cluster along with an official CA Signed certificate for the Harbor deployment, issued by Let's Encrypt via automation through JetStack's Cert Manager.

### NOTES & CAVEATS ###
-These instructions and the accompanying yaml files assume your domain is hosted at AWS/Route53.  If you are using another DNS provider you will need to edit the acme.yaml file appropriately.  You will find instructions for other providers here:  https://cert-manager.io/docs/configuration/acme/dns01/  
-Step 3 installs an nginx ingress controller into the K8s cluster.  This will be used to expose the Harbor registry externally  
-Step 4 installs Cert Manager into the K8s cluster. Cert Manager is a certificate controller that automates the certificate issuance and assignment process in a K8s cluster.  It requires certificate "issuers", in this case we're using LetsEncrypt as the CA Authority which will issue a 90 day cert.  Note, I'm using an "Issuer" as opposed to a "ClusterIssuer" so I've deployed everything in the same namespace (the default ns).  
-Step 5 configures the "issuer" (again, that's LetsEncrypt).  In the Cert Manager documentation the issuer is often referred to as ACME, but technically that is the protocol that LetsEncrypt uses to automate certificate processes.  LetsEncrypt root certs are installed in just about all OS's "trusted cert" stores, so it's a "real CA" to your computer.  There's a process, called a "challenge" by which your ownership of the domain name must be validated.  Let's Encrypt reaches out to an public facing ACME server to complete the challenge.  There are two types of challenges:  HTTP01 and DNS01.  The instructions below use DNS01 which is easier if you don't have a web server with a public IP (see acme.yaml for DNS01 challenge details).  At a high level, the DNS01 challenge works as follows:  the "Issuer" in your cluster (that's LetsEncrypt) will use the AWS creds (stored as a secret) to make an API call to AWS Route 53 to create a DNS TXT record.  A public facing ACME server will look for that DNS TXT record to validate you own the domain before issuing a certificate.
-Step 8 and beyond install Harbor into the cluster.


### INSTRUCTIONS ###
These instructions assume you are using vSphere 7 with Tanzu and have just deployed a new workload cluster:

1.   After authenticating for the first time to your workload cluster, create a ClusterRoleBinding and ClusterRole that uses the Privileged PSP (only necessary in a new cluster):  
`kubectl apply -f https://raw.githubusercontent.com/trevorputbrese/tkc/master/tkc-priveledged-cluster-role.yaml`    <br />

2.   Add the stable Helm repo (this has the nginx chart is here) and the Bitnami repo (which has the chart we use to install Harbor):
`helm repo add stable https://charts.helm.sh/stable`  
`repo add bitnami https://charts.bitnami.com/bitnami`  

3.   Install nginx ingress into the cluster using Helm:  
`helm install nginx-ingress stable/nginx-ingress --set controller.config.proxy-body-size=1024m`

4.   Install Cert Manager into the cluster:  
`kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.yaml`

5.   Apply the acme.yaml file in this repo.  You will first need to edit the file add your own AWS credentials (instructions on how to create IAM role and creds are here:  https://cert-manager.io/docs/configuration/acme/dns01/route53/).  You will also need to modify your the acme.yaml file to use your own domain name.  This yaml will create a secret to store your AWS credentials and also create the Cert Manager "issuer" and "certificate" resources in your cluster:  
`kubectl apply -f acme.yaml`  

6.  Make sure you have a default storage class in your cluster.  If you don't:  
`kubectl annotate sc "YOUR-STORAGE-CLASS" storageclass.kubernetes.io/is-default-class=true`

7.  Modify the harbor_values.yaml to set global.storageClass.  Also modify the file to set externalURL, ingress.hosts.core, and ingress.hosts.notary to your own domain name.  

8.  Install Harbor using Helm:  
`helm install harbor bitnami/harbor -f https://raw.githubusercontent.com/trevorputbrese/certificates-and-K8s/main/harbor_values.yaml`

9.  navigate to harbor.YOUR.DOMAIN in browser and click on the "https" (or the lock icon) to view the certificate and confirm its issued by LetsEncrypt root cert (and not a self-signed Harbor cert)
