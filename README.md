# Description
This project contains the instructions of how to self host [n8n.io](https://n8n.io/) in the GCP GKE environment.

This accompanies a Youtube video I recorded on the subject.

TODO: link to video

# Prerequisites
[Install the Google Cloud SDK](https://cloud.google.com/sdk/docs/install?authuser=3#deb)
```sh
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates gnupg curl sudo

curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

sudo apt-get update
sudo apt-get install google-cloud-cli

gcloud version

```

[Installing kubectl-auth-changes-in-gke](https://cloud.google.com/blog/u/3/products/containers-kubernetes/kubectl-auth-changes-in-gke)
```sh
gcloud components install gke-gcloud-auth-plugin
gke-gcloud-auth-plugin --version
```

[Installing kubectl](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#install_kubectl)
```sh
gcloud components install kubectl
kubectl version
```

Installing postgresql-client
```sh
sudo apt install postgresql-client
psql --version
psql -h localhost -p 5432 -U admin -d n8n

Similarly, install other commands you see present in the doc if necessary

```

# Instructions

## GCP GKE (Free Tier) Deployment
Cerating a GKE Autopilot cluster is a trivial process with [this guide](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-an-autopilot-cluster)

## Prerequisites
These items need to be configured prior to installing N8N in GKE
* External domain name: own domain name with an ability to create/modify records ([CloudDNS](https://cloud.google.com/dns))
* [Managed Certificate Map](https://cloud.google.com/certificate-manager/docs/maps): for enabling a transport layer encryption on your endpoint (https://...) at Application Load Balancer
* [IAP (Identity-Aware Proxy)](https://cloud.google.com/security/products/iap?hl=en): for securing your login screen with Google IdP SSO and enabling authentication
* [CloudArmor](https://cloud.google.com/security/products/armor?hl=en) for protection against DDoS attacks and such

### Managed Certificate Map
The most managable way to do SSL certificates for your domain is to use [GCP Certificate Maps](https://cloud.google.com/certificate-manager/docs/maps)

```sh
# cert: *.iliazlobin.com
gcloud certificate-manager dns-authorizations create iliazlobin-com --domain="iliazlobin.com"
gcloud certificate-manager dns-authorizations describe iliazlobin-com
gcloud certificate-manager dns-authorizations delete iliazlobin-com

gcloud certificate-manager certificates create iliazlobin-com \
  --domains="*.iliazlobin.com" \
  --dns-authorizations=iliazlobin-com
gcloud certificate-manager certificates describe iliazlobin-com

# cert: *.kube.iliazlobin.com
gcloud certificate-manager dns-authorizations create kube-iliazlobin-com-auth --domain="kube.iliazlobin.com"
gcloud certificate-manager dns-authorizations describe kube-iliazlobin-com-auth
# gcloud certificate-manager dns-authorizations list
# gcloud certificate-manager dns-authorizations delete kube-iliazlobin-com

gcloud certificate-manager certificates create kube-iliazlobin-com-cert \
  --domains="kube.iliazlobin.com,*.kube.iliazlobin.com" \
  --dns-authorizations=kube-iliazlobin-com-auth
gcloud certificate-manager certificates describe kube-iliazlobin-com-cert
# gcloud certificate-manager certificates list
# gcloud certificate-manager certificates delete kube-iliazlobin-com-cert

gcloud certificate-manager maps create kube-iliazlobin-com-map
gcloud certificate-manager maps describe kube-iliazlobin-com-map

gcloud certificate-manager maps entries create n8n-kube-iliazlobin-com-map-entry \
  --map=kube-iliazlobin-com-map \
  --hostname="n8n.kube.iliazlobin.com" \
  --certificates=kube-iliazlobin-com-cert

```

## N8N Installation to GKE
This chapter covers a full installation process in a scripted manner:
* Secrets for PostgreSQL
* PostgreSQL database with storage volume claims
* N8N service paired with PostgreSQL
* Testing / port-forwarding
* Gateway API backed by Global Application Load Balancer
* HTTP Routes to the service endpoint
* SSL/TLS activation (HTTPS + certs)
* Authentication with IAP

### Applications and services
```sh
# git clone git@github.com:n8n-io/n8n-kubernetes-hosting.git

kubectl apply -f n8n-gke-autopilot/n8n/_namespace.yaml
kubectl get ns

cp -r n8n-gke-autopilot/n8n/secrets-template n8n-gke-autopilot/n8n/secrets
# kubectl delete -f n8n-gke-autopilot/n8n/secrets
kubectl apply -f n8n-gke-autopilot/n8n/secrets
kubectl -n n8n get secret

# kubectl delete -f n8n-gke-autopilot/n8n/data
kubectl apply -f n8n-gke-autopilot/n8n/data
kubectl -n n8n get deployment
kubectl -n n8n describe deployment postgres

# kubectl delete -f n8n-gke-autopilot/n8n/app
kubectl apply -f n8n-gke-autopilot/n8n/app
kubectl -n n8n get n8n
kubectl -n n8n describe n8n postgres


# confirm all pvc are bound (not pending)
kubectl -n n8n get pvc

kubectl apply -f n8n-kubernetes-hosting/namespace.yaml
kubectl apply -f n8n-kubernetes-hosting

kubectl apply -f n8n-kubernetes-hosting/postgres-secret.yaml
# kubectl delete -f n8n-kubernetes-hosting/postgres-deployment.yaml
# kubectl delete -f n8n-kubernetes-hosting/postgres-claim0-persistentvolumeclaim.yaml
kubectl apply -f n8n-kubernetes-hosting/postgres-claim0-persistentvolumeclaim.yaml
kubectl apply -f n8n-kubernetes-hosting/postgres-deployment.yaml

# kubectl get nodes
# kubectl get node gk3-iz-kube-prod-pool-2-34b067d9-9dwz -o yaml
# kubectl -n n8n get pvc
# kubectl -n n8n get pv

# deployments
kubectl -n n8n get deployments
kubectl -n n8n describe deployment n8n
# kubectl delete -f n8n-kubernetes-hosting/n8n-deployment.yaml
# kubectl delete -f n8n-kubernetes-hosting/n8n-claim0-persistentvolumeclaim.yaml
kubectl apply -f n8n-kubernetes-hosting/n8n-claim0-persistentvolumeclaim.yaml
kubectl apply -f n8n-kubernetes-hosting/n8n-deployment.yaml

kubectl -n n8n get deployments postgres -o yaml
# kubectl apply -f n8n-kubernetes-hosting/postgres-deployment.yaml

# services
kubectl -n n8n get pods
kubectl -n n8n get services
kubectl -n n8n get endpoints
kubectl -n n8n describe services n8n
kubectl delete -f n8n-kubernetes-hosting/n8n-service.yaml
kubectl apply -f n8n-kubernetes-hosting/n8n-service.yaml

# iap-proxy
kubectl apply -f n8n-kubernetes-hosting/n8n-iap-secret.yaml

# backendconfig
kubectl -n n8n get backendconfig
kubectl -n n8n describe backendconfig n8n
kubectl apply -f n8n-kubernetes-hosting/n8n-backendconfig.yaml

```

### Ingress
```sh
# ingress  (backed by Classic Global Application Load Balancer)
# kubectl delete -f n8n-gke-autopilot/n8n/ingress
kubectl apply -f n8n-gke-autopilot/n8n/ingress
kubectl -n n8n get ingress
kubectl -n n8n describe ingress n8n-ingress

```

### Gateway
```sh
# gateway (backed by Global Application Load Balancer)
# kubectl delete -f n8n-gke-autopilot/gateway
kubectl apply -f n8n-gke-autopilot/gateway
kubectl -n gateway get gateway
kubectl -n gateway describe gateway external-gateway

# routing
# kubectl delete -f n8n-gke-autopilot/n8n/routing
kubectl apply -f n8n-gke-autopilot/n8n/routing
kubectl -n n8n get httproute
kubectl -n n8n describe httproute n8n-route

```

## Testing
These comamnds will help you run some useful checks against your exposed HTTP/HTTPS endpoint and DNS records.

```sh
nslookup iliazlobin.com
nslookup n8n.iliazlobin.com
nslookup n8n.kube.iliazlobin.com

nmap -Pn -p 80,443,8080 n8n.iliazlobin.com
nmap -Pn -p 80,443,8080 n8n.kube.iliazlobin.com

# curl -v http://34.49.228.168:8080/
curl -v http://34.49.228.168/
curl -v http://34.49.228.168/healtz
curl -I https://34.49.228.168/healtz
curl -v http://n8n.iliazlobin.com/
curl -v http://n8n.kube.iliazlobin.com/
curl -v https://n8n.kube.iliazlobin.com/

echo | openssl s_client -servername hostname -connect n8n.iliazlobin.com:443 2>/dev/null | openssl x509 -noout -subject
echo | openssl s_client -connect 34.49.228.168:443 2>/dev/null
echo | openssl s_client -connect 34.49.228.168:443 2>/dev/null | openssl x509 -noout -subject

openssl s_client -servername n8n.iliazlobin.com -connect n8n.iliazlobin.com:443 2>/dev/null | openssl x509 -noout -subject
echo | openssl s_client -servername google.com -connect google.com:443 2>/dev/null | openssl x509 -noout -subject
echo | openssl s_client -connect 151.101.194.186:443 2>/dev/null | openssl x509 -noout -subject

```


## Troubleshooting
All commands you need to investigate the situation if something doesn't go quite smooth.

```sh
# port-forwarding to services for local testing
kubectl -n n8n get svc
kubectl -n n8n port-forward svc/postgres-service 5432:5432
kubectl -n n8n port-forward svc/n8n-service 5678:5678

# connect to postgres (since proxied locally, doesn't require password, trusts are configurable in pg_hba.com)
PGPASSWORD=PASSWORD psql -h localhost -p 5432 -U admin -d n8n
PGPASSWORD=PASSWORD psql -h localhost -p 5432 -U user -d n8n

# simple postgres commands
# \l
# \c n8n
# \dt

# exec
kubectl -n n8n get pod
kubectl -n n8n exec -it postgres-79d9d6f5f9-cmgk2 -- bash
cat /var/lib/postgresql/data/pgdata/pg_hba.conf

# port-forwarding to PODS for local testing (pod name always changes)
# kubectl -n n8n get pod
# kubectl -n n8n port-forward n8n-648b5c64cf-289qv 5678:5678
# kubectl -n n8n port-forward postgres-6dbd765c97-7t222 5432:5432

# check logs on deployments (aggregate from all pods)
kubectl -n n8n logs deployment/postgres
kubectl -n n8n logs deployment/n8n

# check logs on individual pods
# kubectl -n n8n logs -c volume-permissions n8n-648b5c64cf-289qv
# kubectl -n n8n logs -c n8n n8n-648b5c64cf-289qv
# kubectl -n n8n logs postgres-6dbd765c97-7t222

# check load balancer specs
gcloud compute url-maps list
gcloud compute url-maps describe gkegw1-575w-gateway-external-gateway-cm5jpi635k5h --global

# check certificate maps
gcloud compute target-http-proxies list
gcloud compute target-https-proxies list

gcloud compute target-http-proxies create http-proxy --url-map=gkegw1-575w-gateway-external-gateway-cm5jpi635k5h
gcloud compute target-http-proxies describe http-proxy

# gcloud compute target-http-proxies create http-proxy --url-map=gkegw1-575w-gateway-external-gateway-cm5jpi635k5h
# gcloud compute target-http-proxies delete http-proxy

# check backend service specs
gcloud compute backend-services list
gcloud compute backend-services describe gkegw1-575w-n8n-n8n-service-5678-qx1m60r8pnj3

# check health check policies
gcloud compute health-checks list
gcloud compute health-checks describe gkegw1-575w-n8n-n8n-service-5678-qx1m60r8pnj3 --region us-central1

# check NEGs specs
gcloud compute network-endpoint-groups list
gcloud compute network-endpoint-groups describe k8s1-4f97364b-n8n-n8n-service-5678-9a0ce1b7 --zone us-central1-f

```

# About
* [x.com/iliazlobin](https://x.com/iliazlobin)
* [linkedin.com/in/iliazlobin](https://www.linkedin.com/in/iliazlobin)
* [youtube.com/@iliazlobin](https://www.youtube.com/@iliazlobin)
* [mailto:contact@iliazlobin.com](mailto:contact@iliazlobin.com)
