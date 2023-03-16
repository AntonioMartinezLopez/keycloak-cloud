## STEP 1 ##
Install minikube, kubectl and HELM 3

add following host to /etc/hosts

'< minikube ip > oauth.whoami.test whoami.test keycloak.test k8s-dashboard.test'

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
















## STEP 3 ##
add ingressroutes to minikube dashboard
```bash
kubectl create -f ingressRoute_dashboard.yaml -n kubernetes-dashboard
kubectl create -f service_dashboard.yaml -n kubernetes-dashboard
```

## STEP 4 ##
add following host to /etc/hosts

'<minikube ip>  test.example.com whoami.com'

## STEP 5 ##

install app1 (ngninx)
```bash
kubectl create ns app1
kubectl create -f deployment_app1.yaml -f service_app1.yaml -f ingressRoute_app1.yaml -n app1
```
curl or browse 'test.example.com', the default nginx page should be shown

## STEP 6 ##

install whoami service

```bash
kubectl create ns whoami
kubectl create -f traefik-whoami.yaml -n whoami
```

curl or browse 'whoami/notsl', the default nginx page should be shown

## STEP 7 ##

install keycloak

first, install postgres:
```bash
kubectl create ns auth
kubectl apply -f pv_postgres.yaml 
kubectl apply -f pvc_postgres.yaml -n auth 
kubectl apply -f configMap_postgres.yaml -n auth
kubectl apply -f deployment_postgres.yaml -n auth
```

then, install keycloak operator and a keycloak deployment
```bash
kubectl create -f 0_keycloaks.k8s.keycloak.org-v1.yaml -n auth
kubectl create -f 1_keycloakrealmimports.k8s.keycloak.org-v1.yaml -n auth
kubectl create -f 2_deployment_keycloak_operator.yaml -n auth
kubectl create -f 3_secret_keycloak.yaml -n auth
kubectl create -f 4_deployment_keycloak.yaml -n auth
```

from kubernetes dashboard, read the auto-set admin password from the environmental variables within the keycloak deployment.
This password is used to log in to the admin console.


## STEP 8 ##

install Oauth2 proxy for every application

initially:
```bash
helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
```

and then per application
helm install my-release oauth2-proxy/oauth2-proxy -n <namespace>