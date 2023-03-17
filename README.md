## STEP 1 ##
Install minikube, kubectl and HELM 3

add following host to /etc/hosts

'< minikube ip > oauth.traefik.test traefik.test oauth.whoami.test whoami.test keycloak.test oauth.kubernetes.test kubernetes.test'

kubernetes dashboard: 
```bash
minikube dashboard
```


## STEP 2 - setup traefik ##

Get the traefik chart
```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

Start traefik deployment:

```bash
helm install traefik traefik/traefik --version 21.1.0 --namespace traefik --values ./traefik/values.yaml --create-namespace
```


Forwarding to dashboard

```bash
kubectl port-forward $(kubectl get pods --selector "app.kubernetes.io/name=traefik" --output=name) 9000:9000
```

or

```bash
kubectl port-forward -n traefik traefik-XXXXXXXXX-XXXXX 9000:9000
```

initially:
```bash
helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
```


## STEP 3 - setup keycloak ##

install keycloak

first, install postgres:
```bash
kubectl create ns auth
kubectl apply -f keycloak/postgres/0_configMap_postgres.yaml -n auth
kubectl apply -f keycloak/postgres/1_pv_postgres.yaml -n auth
kubectl apply -f keycloak/postgres/2_pvc_postgres.yaml -n auth 
kubectl apply -f keycloak/postgres/3_deployment_postgres.yaml -n auth
```

then, install keycloak operator and a keycloak deployment
```bash
kubectl create -f keycloak/0_keycloaks.k8s.keycloak.org-v1.yaml -n auth
kubectl create -f keycloak/1_keycloakrealmimports.k8s.keycloak.org-v1.yaml -n auth
kubectl create -f keycloak/2_deployment_keycloak_operator.yaml -n auth
kubectl create -f keycloak/3_secret_keycloak.yaml -n auth
kubectl create -f keycloak/4_deployment_keycloak.yaml -n auth
kubectl create -f keycloak/5_ingressRoute_keycloak.yaml -n auth
```

- Visit the keycloak admin console: keycloak.test
- from kubernetes dashboard ($minikube dashboard), read the auto-set admin password from the environmental variables within the keycloak deployment.
- This password is used to log in to the admin console.



## STEP 4 - setup whoami service  ##

install whoami service

```bash
kubectl create ns whoami
kubectl create -f whoami/traefik-whoami.yaml -n whoami
```

secure application : create realm
```bash
kubectl apply -f whoami/auth/realm_whoami.yaml -n auth
```
wait for approx. 5 min until migration job has finished

build oauth2 proxy for given realm
```bash
helm install whoami-oauth2-proxy oauth2-proxy/oauth2-proxy -f whoami/auth/values.yaml --namespace=auth
kubectl create -f whoami/auth/ingressRoute_auth_whoami.yaml -n auth
```

- browse 'whoami.test/notsl', a keycloak login should appear
- Login in with user: testuser and pw:test
- the whoami output should now be visible
- logout with following link: http://whoami.test/oauth2/sign_out?rd=http%3A%2F%2Fkeycloak.test%2Frealms%2Fwhoami%2Fprotocol%2Fopenid-connect%2Flogout



## STEP 5 - protect traefik dashboard

secure application : create realm for admin ( is also used later for kubernetes dashboard)
```bash
kubectl apply -f traefik/auth/realm_admin.yaml -n auth
```
wait for approx. 5 min until migration job has finished


build oauth2 proxy for given realm

```bash
helm install traefik-oauth2-proxy oauth2-proxy/oauth2-proxy -f traefik/auth/values.yaml --namespace=auth
kubectl create -f traefik/auth/ingressRoute_auth_traefik.yaml -n auth
```

add ingress with forward auth middleware for traefik dashboard
```bash
kubectl create -f traefik/ingressRoute_dashboard.yaml -n traefik
```

- browse 'traefik.test/dashboard/', a keycloak login should appear
- Login in with user: admin and pw:admin
- the traefik dashboard should now be visible
- logout with following link: http://traefik.test/oauth2/sign_out?rd=http%3A%2F%2Fkeycloak.test%2Frealms%2Fadmin%2Fprotocol%2Fopenid-connect%2Flogout


## STEP 6 - protect kubernetes dashboard

build oauth2 proxy for given realm (admin)

```bash
helm install kubernetes-oauth2-proxy oauth2-proxy/oauth2-proxy -f kubernetes-dashboard/auth/values.yaml --namespace=auth
kubectl create -f kubernetes-dashboard/auth/ingressRoute_auth_kubernetes.yaml -n auth
```
add ingress with forward auth middleware for kubernetes dashboard
```bash
kubectl create -f kubernetes-dashboard/ingressRoute_dashboard.yaml -n kubernetes-dashboard
```

- browse 'kubernetes.test', a keycloak login should appear
- Login in with user: admin and pw:admin
- the kubernetes dashboard should now be visible
- logout with following link: http://kubernetes.test/oauth2/sign_out?rd=http%3A%2F%2Fkeycloak.test%2Frealms%2Fadmin%2Fprotocol%2Fopenid-connect%2Flogout