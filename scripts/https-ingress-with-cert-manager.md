# Ingress with HTTPs, `nginx` & `istio`

---

## Table of Contents

1. Deploy `cert-manager`
2. Prepare `helm`
   * (Deprecated) Set `tiller` for `helm2`, which is the server-side component of `helm`
3. Deploy `nginx-ingress-controller` first
4. Set DNS
5. Configure `Issuer`(`cert-manager`) for Let's Encrypt
6. Deploy a TLS Ingress Resource

---
## Deploy `cert-manager`

```bash
# Install 'cert-manager' >= 1.0.0 to use the latest API.
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.yaml
```

## Prepare `helm`

### Get `helm`

```bash
curl -o get_helm.sh https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get && \
  chmod +x get_helm.sh && \
  ./get_helm.sh

helm init
```

### (Deprecated) Set `tiller` for `helm2`

```bash
# Only available with 'helm2'
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller  # --upgrade
```

then, the output is:

```bash
serviceaccount/tiller created
clusterrolebinding.rbac.authorization.k8s.io/tiller-cluster-rule created
$HELM_HOME has been configured at /home/pydemia/.helm.

Tiller (the Helm server-side component) has been updated to gcr.io/kubernetes-helm/tiller:v2.16.7 .
```

### Deploy `nginx-ingress-controller` first.

We use this repository: https://kubernetes.github.io/ingress-nginx  
[helm config values](https://github.com/kubernetes/ingress-nginx/blob/master/charts/ingress-nginx/values.yaml)

```bash
NAMESPACE="istio-system"  
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install \
  ingress-nginx \
  --namespace ${NAMESPACE} \
  ingress-nginx/ingress-nginx \
  --set metrics.enabled=true
```

then, the output is:

```bash
"ingress-nginx" has been added to your repositories
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
...Successfully got an update from the "nginx-stable" chart repository
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
NAME: ingress-nginx
LAST DEPLOYED: Mon Feb  8 06:19:06 2021
NAMESPACE: istio-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace istio-system get services -o wide -w ingress-nginx-controller'

An example Ingress that makes use of the controller:

  apiVersion: networking.k8s.io/v1beta1
  kind: Ingress
  metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
    name: example
    namespace: foo
  spec:
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                serviceName: exampleService
                servicePort: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
        - hosts:
            - www.example.com
          secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls

```

After a while, you can see the services as the following:

```bash
$ kubectl -n istio-system get svc -l app.kubernetes.io/name=ingress-nginx

NAME                                 TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   x.x.x.x      y.y.y.y          80:31273/TCP,443:32005/TCP   43h
ingress-nginx-controller-admission   ClusterIP      x.x.x.x      <none>           443/TCP                      43h
```

If you can use `LoadBalancer` type, an External-IP `y.y.y.y` is automatically allocated to `ingress-nginx-controller` for public access.  
This IP is used for Ingress IP.

### Set DNS


An SSL certificate is typically issued to a Fully Qualified Domain Name (FQDN) such as `www.example.com`.  
You should have an DOMAIN to assign the IP `y.y.y.y`.  
We assume you have an domain `example.com` owned by `user@example.com` in this case.  
We will use `*.example.com` for production and `*.stg.example.com` for staging.

### Configure `Issuer`(`cert-manager`) for Let's Encrypt

* `Issuer` for production

```yaml
# apiVersion: certmanager.k8s.io/v1alpha1  # < v0.11
# apiVersion: cert-manager.io/v1alpha2  # >= v0.11
# apiVersion: cert-manager.io/v1beta1  # >= v0.16
apiVersion: cert-manager.io/v1  # >= v1.0
kind: Issuer
metadata:
    name: letsencrypt-prod
spec:
    acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: user@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
        name: letsencrypt-prod
    solvers:
    # An empty 'selector' means that this solver matches all domains
    - selector: {}
      # Enable the HTTP-01 challenge provider
      http01:
        ingress:
            # matches the ingress have the annotation 'kubernetes.io/ingress.class: nginx'
            class:  nginx
```

* `Issuer` for staging

```yaml
# apiVersion: certmanager.k8s.io/v1alpha1  # < v0.11
# apiVersion: cert-manager.io/v1alpha2  # >= v0.11
# apiVersion: cert-manager.io/v1beta1  # >= v0.16
apiVersion: cert-manager.io/v1  # >= v1.0
kind: Issuer
metadata:
    name: letsencrypt-staging
spec:
    acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: user@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
        name: letsencrypt-staging
        # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
            class:  nginx
```

check it:

```bash
$ kubectl describe issuer letsencrypt-staging
Name:         letsencrypt-staging
Namespace:    istio-system
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"cert-manager.io/v1","kind":"Issuer","metadata":{"annotations":{},"name":"letsencrypt-staging","namespace":"default"},(...)}
API Version:  cert-manager.io/v1
Kind:         Issuer
Metadata:
  Cluster Name:
  Creation Timestamp:  2018-11-17T18:03:54Z
  Generation:          0
  Resource Version:    9092
  Self Link:           /apis/cert-manager.io/v1/namespaces/default/issuers/letsencrypt-staging
  UID:                 25b7ae77-ea93-11e8-82f8-42010a8a00b5
Spec:
  Acme:
    Email:  user@example.com
    Private Key Secret Ref:
      Key:
      Name:  letsencrypt-staging
    Server:  https://acme-staging-v02.api.letsencrypt.org/directory
    Solvers:
      Http 01:
        Ingress:
          Class:  nginx
Status:
  Acme:
    Uri:  https://acme-staging-v02.api.letsencrypt.org/acme/acct/7374163
  Conditions:
    Last Transition Time:  2018-11-17T18:04:00Z
    Message:               The ACME account was registered with the ACME server
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

### Deploy a TLS Ingress Resource

[Available annotations](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md)

* `Ingress` for production

```yml
# apiVersion: networking.k8s.io/v1  # >= k8s 1.16
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-prd
  namespace: istio-system
  annotations:
    # cert-manager.io/issuer  # >= v0.11
    cert-manager.io/issuer: letsencrypt-prd
    # cert-manager.io/cluster-issuer: letsencrypt-prd
    # letsencrypt-environment: "production"
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true" # "true"
    # kubernetes.io/tls-acme: "true"  # < Could not determine issuer for ingress due to bad annotations: failed to determine issuer name to be used for ingress resource
    nginx.ingress.kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:  # < placing a host in the TLS config will indicate a cert should be created
  - hosts:
    # wildcard domains as `*.example.com` is not allowed in Let's Encrypt.
    - example.com
    - app.example.com
    - dashboard.example.com
    secretName: ingress-tls-prd  # < cert-manager will store the created certificate in this secret.
    # `kubectl -n kubeflow describe certificates.cert-manager.io nginx-tls-prod`
  rules:
  - host: dashboard.airuntime.com  # www.example.com
    http:
      paths:
      # - path: /*
      #   backend:
      #     serviceName: istio-ingressgateway
      #     servicePort: 443
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
          # serviceName: istio-ingressgateway
          # servicePort: 80
```


* `Ingress` for staging

```yml
# apiVersion: networking.k8s.io/v1  # >= k8s 1.16
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-stg
  namespace: istio-system
  annotations:
    # cert-manager.io/issuer  # >= v0.11
    cert-manager.io/issuer: letsencrypt-stg
    # cert-manager.io/cluster-issuer: letsencrypt-stg
    # letsencrypt-environment: "production"
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true" # "true"
    # kubernetes.io/tls-acme: "true"  # < Could not determine issuer for ingress due to bad annotations: failed to determine issuer name to be used for ingress resource
    nginx.ingress.kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:  # < placing a host in the TLS config will indicate a cert should be created
  - hosts:
    # wildcard domains as `*.example.com` is not allowed in Let's Encrypt.
    - dev.example.com
    - app.dev.example.com
    - dashboard.dev.example.com
    secretName: ingress-tls-stg  # < cert-manager will store the created certificate in this secret.
    # `kubectl -n kubeflow describe certificates.cert-manager.io nginx-tls-prod`
  rules:
  - host: dashboard.airuntime.com  # www.example.com
    http:
      paths:
      # - path: /*
      #   backend:
      #     serviceName: istio-ingressgateway
      #     servicePort: 443
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
          # serviceName: istio-ingressgateway
          # servicePort: 80
```

After a while, `Certificate` ingress-tls-stg is automatically configured in this sequence:

1. `certificaterequests` ingress-tls-stg-xxxx
2. `orders` ingress-tls-stg-xxxx-yyyy
3. `challenges` ingress-tls-stg-xxxx-yyyy-zzzz (as many as `spec.tls.hosts` in Ingress)
4. `certificates` ingress-tls-stg


