## Install the Cert Manager

Add the Helm repository

This repository is the only supported source of cert-manager charts. There are some other mirrors and copies across the internet, but those are entirely unofficial and could present a security risk.

Notably, the "Helm stable repository" version of cert-manager is deprecated and should not be used.

```shell
helm repo add jetstack https://charts.jetstack.io --force-update
```
Update your local Helm chart repository cache:
```shell
helm repo update
```
Install CustomResourceDefinitions

cert-manager requires a number of CRD resources, which can be installed manually using kubectl, or using the installCRDs option when installing the Helm chart. Both options are described below and will achieve the same result but with varying consequences. You should consult the CRD Considerations section below for details on each method.

Installing CRDs with kubectl (recommended for production installations)
```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.crds.yaml
```
Install cert-manager

To install the cert-manager Helm chart, use the Helm install command as described below.

```shell
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.14.5 \
  # --set installCRDs=true
```

Configure a Let's Encrypt Issuer

We'll set up two issuers for Let's Encrypt in this example: staging and production.

Create this definition locally and update the email address to your own. This email is required by Let's Encrypt and used to notify you of certificate expiration and updates.
```shell
nano staging-issuer.yaml
```
```
# staging-issuer.yaml
apiVersion: cert-manager.io/v1
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
            ingressClassName: nginx
```
```shell
kubectl create -f staging-issuer.yaml
# expected output: issuer.cert-manager.io "letsencrypt-staging" created
```
Also create a production issuer and deploy it. As with the staging issuer, you will need to update this example and add in your own email address.
```shell
nano production-issuer.yaml
```
```
# production-issuer.yaml
apiVersion: cert-manager.io/v1
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
    # Enable the HTTP-01 challenge provider
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```
```shell
kubectl create -f production-issuer.yaml
# expected output: issuer.cert-manager.io "letsencrypt-prod" created
```
Both of these issuers are configured to use the HTTP01 challenge provider.

Check on the status of the issuers after you create it:
```shell
kubectl describe issuer letsencrypt-staging
kubectl describe issuer letsencrypt-prod
```

```shell
kubectl get issuer.cert-manager.io
```
```
#Output
NAME                  READY   AGE
letsencrypt-prod      True    102m
letsencrypt-staging   True    105m
```
