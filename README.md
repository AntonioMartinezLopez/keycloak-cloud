# Set Up Cluster 1 (Keycloak)

Install minikube, kubectl and HELM 3

Get the traefik chart and oauth2proxy chart
```bash
helm repo add traefik https://traefik.github.io/charts
helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
helm repo update
```

## start the cluster and get the ip adress


```bash
minikube start -p cluster1 --driver=virtualbox --memory=6000

cd test-cluster/cluster1

minikube ip -p cluster1
 -> ping 192.168.39.135

```

## traefik
---

### install traefik helm chart

```bash
helm install traefik traefik/traefik --version 21.1.0 --namespace traefik --values ./traefik/values.yaml --create-namespace
kubectl create -f traefik/service_dashboard.yaml -n traefik
```
### create self signed certificate for testing

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes -keyout tls-traefik.key -out tls-traefik.crt -subj "/CN=traefik.test" -addext "subjectAltName=DNS:oauth.traefik.test,DNS:traefik.test,DNS:www.traefik.test,IP:192.168.39.135"
kubectl create secret generic traefik-dashboard-secret --from-file=tls.crt=./tls-traefik.crt --from-file=tls.key=./tls-traefik.key --namespace traefik
```

## keycloak
---


### install keycloak - first, install postgres:

```bash
kubectl create ns auth
kubectl apply -f keycloak/postgres/0_configMap_postgres.yaml -n auth
kubectl apply -f keycloak/postgres/1_pv_postgres.yaml -n auth
kubectl apply -f keycloak/postgres/2_pvc_postgres.yaml -n auth 
kubectl apply -f keycloak/postgres/3_deployment_postgres.yaml -n auth
```

### then, install keycloak operator and a keycloak deployment

```bash
kubectl create -f keycloak/0_keycloaks.k8s.keycloak.org-v1.yaml -n auth
kubectl create -f keycloak/1_keycloakrealmimports.k8s.keycloak.org-v1.yaml -n auth
kubectl create -f keycloak/2_deployment_keycloak_operator.yaml -n auth
kubectl create -f keycloak/3_secret_keycloak.yaml -n auth
kubectl create -f keycloak/4_deployment_keycloak.yaml -n auth
```

### create self signed certificate for testing

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes -keyout tls-keycloak.key -out tls-keycloak.crt -subj "/CN=keycloak.test" -addext "subjectAltName=DNS:oauth.keycloak.test,DNS:keycloak.test,DNS:www.keycloak.test,IP:192.168.39.135"
```

### create secret and add ingress route for keycloak

```bash
kubectl create secret generic keycloak-dashboard-secret --from-file=tls.crt=./tls-keycloak.crt --from-file=tls.key=./tls-keycloak.key --namespace auth
kubectl create -f keycloak/5_ingressRoute_keycloak.yaml -n auth
```
 
## protect traefik dashboard
---

### secure application : create realm for admin ( is also used later for kubernetes dashboard)

```bash
kubectl apply -f traefik/auth/realm_admin.yaml -n auth
```
wait for approx. 5 min until migration job has finished


### build oauth2 proxy for given realm

```bash
helm install traefik-oauth2-proxy oauth2-proxy/oauth2-proxy -f traefik/auth/values.yaml --namespace=auth
```

### add ingress with forward auth middleware for traefik dashboard

```bash
kubectl create secret generic traefik-dashboard-secret --from-file=tls.crt=./tls-traefik.crt --from-file=tls.key=./tls-traefik.key --namespace auth
kubectl create -f traefik/ingressRoute_dashboard.yaml -n traefik
kubectl create -f traefik/auth/ingressRoute_auth_traefik.yaml -n auth
```

- browse 'traefik.test/dashboard/', a keycloak login should appear
- Login in with user: admin and pw:admin
- the traefik dashboard should now be visible
- logout with following link: https://traefik.test/oauth2/sign_out?rd=https%3A%2F%2Fkeycloak.test%2Frealms%2Fadmin%2Fprotocol%2Fopenid-connect%2Flogout



# Copy self signed certificate to certs folder of minikube

```bash
cp cluster1/tls-keycloak.crt ~/.minikube/certs/tls-keycloak.pem
```

# Change Folder to cluster 2

```bash
cd ../cluster2
```

# Setup Cluster 2 - Application environment

```bash
start -p cluster2 --extra-config=apiserver.authorization-mode=Node,RBAC --extra-config=apiserver.oidc-issuer-url=https://keycloak.test/realms/admin --extra-config=apiserver.oidc-username-claim=email --extra-config=apiserver.oidc-client-id=kubernetes-oauth2-proxy --extra-config=apiserver.oidc-groups-prefix=keycloak: --extra-config=apiserver.oidc-username-prefix=keycloak: --extra-config=apiserver.oidc-groups-claim=groups --driver=virtualbox --cpus=2 --memory=6000
```

add following host to /etc/hosts

'< ip cluster1 >  oauth.traefik.test traefik.test keycloak.test'

'< ip cluster2 >  oauth.traefik2.test traefik2.test oauth.kubernetes2.test kubernetes2.test'


## STEP 2 - setup traefik ##


Start traefik deployment:

```bash
helm install traefik traefik/traefik --version 21.1.0 --namespace traefik --values ./traefik/values.yaml --create-namespace

kubectl create -f traefik/service_dashboard.yaml -n traefik
```


Forwarding to dashboard

```bash
kubectl port-forward $(kubectl get pods --selector "app.kubernetes.io/name=traefik" --output=name) 9000:9000
```

or

```bash
kubectl port-forward -n traefik traefik-XXXXXXXXX-XXXXX 9000:9000
```


## STEP 5 - protect traefik dashboard

build oauth2 proxy for given realm

```bash
helm install traefik-oauth2-proxy oauth2-proxy/oauth2-proxy -f traefik/auth/values.yaml --namespace=traefik
kubectl create -f traefik/auth/ingressRoute_auth_traefik.yaml -n traefik
```

add ingress with forward auth middleware for traefik dashboard
```bash
kubectl create -f traefik/ingressRoute_dashboard.yaml -n traefik
```

- browse 'traefik.test/dashboard/', a keycloak login should appear
- Login in with user: admin and pw:admin
- the traefik dashboard should now be visible
- logout with following link: http://traefik2.test/oauth2/sign_out?rd=https%3A%2F%2Fkeycloak.test%2Frealms%2Fadmin%2Fprotocol%2Fopenid-connect%2Flogout


## STEP 6 - protect kubernetes dashboard

build oauth2 proxy for given realm (admin)

kubernetes dashboard: 
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```


```bash

helm install kubernetes-oauth2-proxy oauth2-proxy/oauth2-proxy -f kubernetes-dashboard/auth/values.yaml --namespace=kubernetes-dashboard
kubectl create -f kubernetes-dashboard/auth/ingressRoute_auth_kubernetes.yaml -n kubernetes-dashboard
```
add ingress with forward auth middleware for kubernetes dashboard
```bash
kubectl create -f kubernetes-dashboard/ingressRoute_dashboard.yaml -n kubernetes-dashboard
```

- browse 'kubernetes.test', a keycloak login should appear
- Login in with user: admin and pw:admin
- the kubernetes dashboard should now be visible
- logout with following link: http://kubernetes2.test/oauth2/sign_out?rd=https%3A%2F%2Fkeycloak.test%2Frealms%2Fadmin%2Fprotocol%2Fopenid-connect%2Flogout


# create new instance of an application (example whoami)

switch to keycloak cluster and create realm

```bash
cd ../whoami
minikube profile cluster1
kubectl create -f auth/realm_whoami.yaml -n auth
```

wait 5 min until realm instance has been initialized

switch back to customer cluster

```bash
minikube profile cluster2
```

initialize ingress and deployment

```bash
kubectl create ns whoami
kubectl create -f traefik-whoami.yaml -n whoami
kubectl create -f ingressRoute_auth_whoami.yaml -n whoami
```

logout: http://testwhoami.test/oauth2/sign_out?rd=https%3A%2F%2Fkeycloak.test%2Frealms%2Ftestwhoami%2Fprotocol%2Fopenid-connect%2Flogout


install ouath2-proxy

```bash
helm install whoami-oauth2-proxy oauth2-proxy/oauth2-proxy -f values.yaml --namespace=whoami
```

# extract realm settings

execute bash within keycloak server

```bash
kubectl exec -it -n auth pods/phxwi-kc-0 -- bash
```

extract realm as json

```bash
/opt/keycloak/bin/kc.sh export --dir ../data/admin --users realm_file --realm admin
```

copy realm specification from pod

```bash
kubectl exec -it -n auth pods/phxwi-kc-0 -- cat /opt/keycloak/data/admin/admin-realm.json >> admin_realm_updated.json
```


