---
title: "Using Vault as a Certificate Manager on OpenShift"
tags: 
  - vault
  - openshift
  - hashicorp
  - helm
  - certificate
  - pki
---

<center>
<img align="center" src="/assets/images/vault_cm_ocp1.png" alt=""> 
</center>


In this post, we will proceed to try on some of the features that Vault can offer as a secrets manager. One of the feature is that Vault is able to be configured as a certifcate manager. This enables your services to establish their identity and communicate securely over the network with other services or clients internal or external to the cluster.

Jetstack's cert-manager enables Vault's PKI secrets engine to dynamically generate X.509 certificates within OpenShift through an Issuer interface.
In this guide, we will configure the PKI secrets engine and OpenShift authentication. Then install Jetstack's cert-manager, configure it to use Vault, and request a certificate.

Most of the steps documented below are taken from Vault documentation [here][vault-certmanager]. We are using Red Hat OpenShift as the Kubernetes plaform for the steps below.

[vault-certmanager]: https://learn.hashicorp.com/vault/kubernetes/cert-manager

## Prerequisites

The following are required prior to the setup:

  * Running OpenShift cluster (I am using OpenShift 4.4 in this example)
  * Helm CLI installed (Refer [here][helm-ocp-install] to learn how to install Helm CLI on OpenShift 4)
  * Vault running on OpenShift (refer to previous blog entry [here][vault-ocp])
  
  [helm-ocp-install]: https://docs.openshift.com/container-platform/4.4/cli_reference/helm_cli/getting-started-with-helm-on-openshift-container-platform.html
  [vault-ocp]: https://raylaijh.github.io/vault-ocp/
  

## Configure PKI secrets engine

First, enable the PKI secrets engine at the default path on Vault.

```css
$ oc exec -it vault-0 -- /bin/sh
/ $
```

```css
$ vault secrets enable pki
Success! Enabled the pki secrets engine at: pki/
```

By default the KPI secrets engine sets the time-to-live (TTL) to 30 days. A certificate can have its lease extended to ensure certificate rotation on a yearly basis (8760h). Tuning of this TTL can be done easily on Vault.

```css
$ vault secrets tune -max-lease-ttl=8760h pki
Success! Tuned the secrets engine at: pki/
```

Generate a self-signed certificate valid for `8760h`

```css
$ vault write pki/root/generate/internal \
    common_name=example.com \
    ttl=8760h
Key              Value
---              -----
certificate      -----BEGIN CERTIFICATE-----
## ...
-----END CERTIFICATE-----
expiration       1619120269
issuing_ca       -----BEGIN CERTIFICATE-----
## ...
-----END CERTIFICATE-----
serial_number    65:37:b5:b3:91:6c:7b:d8:33:22:03:28:b1:58:ff:be:8a:72:a4:c0
```

Configure the PKI secrets engine certificate issuing and certificate revocation list (CRL) endpoints to use the Vault service in the default namespace.

```css
$ vault write pki/config/urls \
    issuing_certificates="http://vault.default:8200/v1/pki/ca" \
    crl_distribution_points="http://vault.default:8200/v1/pki/crl"
Success! Data written to: pki/config/urls
```

Configure a role named `example-dot-com` that enables the creation of certificates example.com domain with any subdomains. The role, `example-dot-com`, is a logical name that maps to a policy used to generate credentials. 

```css
$ vault write pki/roles/example-dot-com \
    allowed_domains=example.com \
    allow_subdomains=true \
    max_ttl=72h
Success! Data written to: pki/roles/example-dot-com
```
Next, we will also need to define the policy in Vault which maps to all these paths where OpenShift will trigger. This will be attached to the OpenShift service account used later on. Here, we create the policy `pki`, that enables read access to the PKI secrets engine path. These paths enable the token to view all the roles created for this PKI secrets engine and access the sign and issues operations for the example-dot-com role.

```css
$ vault policy write pki - <<EOF
path "pki*"                        { capabilities = ["read", "list"] }
path "pki/roles/example-dot-com"   { capabilities = ["create", "update"] }
path "pki/sign/example-dot-com"    { capabilities = ["create", "update"] }
path "pki/issue/example-dot-com"   { capabilities = ["create"] }
EOF
Success! Uploaded policy: pki
```


## Configure Kubernetes authentication

Enable the Kubernetes authentication method

```css
$ vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
```

Configure the Kubernetes authentication method to use the service account token, the location of the Kubernetes host, and its certificate.

```css
$ vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
Success! Data written to: auth/kubernetes/config
```
The `token_reviewer_jwt` and `kubernetes_ca_cert` reference files written to the container by Kubernetes. The environment variable `KUBERNETES_PORT_443_TCP_ADDR` references the internal network address of the Kubernetes host.

Finally, create a Kubernetes authentication role named `issuer` that binds the `pki` policy with a Kubernetes service account named `issuer`. This `isser` service account will be used by Jetstack cert-manager later. The role connects the Kubernetes service account, `issuer`, in the `default` namespace with the `pki` Vault policy. The token generated by Vault, issued to service account `issuer` will be vaild for 20 minutes.

```css
$ vault write auth/kubernetes/role/issuer \
    bound_service_account_names=issuer \
    bound_service_account_namespaces=default \
    policies=pki \
    ttl=20m
Success! Data written to: auth/kubernetes/role/issuer
```

Exit from the `vault-0` pod

```css
$ exit
```


## Deploy Cert Manager

Jetstack's cert-manager is a Kubernetes add-on that automates the management and issuance of TLS certificates from various issuing sources. Vault can be configured as one of those sources. To understand more about this cert-manager, check out the links below.


* [https://github.com/jetstack/cert-manager][link1]
* [https://cert-manager.io/docs/installation/kubernetes/][link2]

[link1]: https://github.com/jetstack/cert-manager
[link2]: https://github.com/jetstack/cert-manager

Install Jetstack's cert-manager's version 0.14.3 resources.

```css
$ oc apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.3/cert-manager.crds.yaml
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
```

Create a project named `cert-manager` to host the cert-manager.

```css
$ oc new-project cert-manager
Now using project "cert-manager" on server "https://api.cluster-sgp-3a38.sgp-3a38.example.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app ruby~https://github.com/sclorg/ruby-ex.git

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
```

We will add the Helm repository of `Jetstack` so as to install `cert-manager`

```css
$ helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jetstack" chart repository
Update Complete. ⎈ Happy Helming!⎈

$ helm install cert-manager \
    --namespace cert-manager \
    --version v0.14.3 \
   jetstack/cert-manager
NAME: cert-manager
## ...
```

We should be able to see the `cert-manager` pods running in the cert-manager project

```css
$ oc get pods 
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-77fcb5c94-zx2kc              1/1     Running   0          10s
cert-manager-cainjector-8fff85c75-gtw4j   1/1     Running   0          10s
cert-manager-webhook-54647cbd5-pddrt      1/1     Running   0          10s
```

## Configure an issuer and generate a certificate

The cert-manager enables you to define Issuers that interface with the Vault certificate generating endpoints. These Issuers are invoked when a Certificate is created.

Create the `issuer` service account in the `default` project
```css
$ oc create serviceaccount issuer -n default
serviceaccount/issuer created
```

The service account generated a secret that is required by the `Issuer`.

Get all the secrets in the `default` namespace. Look for the secret name required by the `issuer` service account. It should be named similar to `issuer-token-xxxxx`. Capture the secret name with a variable `ISSUER_SECRET_REF`.

```css
$ oc get secrets -n default
...
default-token-mlm2n           kubernetes.io/service-account-token   3      13d
issuer-token-lmzpj            kubernetes.io/service-account-token   3      47s
sh.helm.release.v1.vault.v1   helm.sh/release.v1                    1      28m
vault-token-749nd             kubernetes.io/service-account-token   3      28m
...
```

```css
$ ISSUER_SECRET_REF=$(oc get serviceaccount issuer -o json | jq -r ".secrets[].name")
```

Create an Issuer custom resource for cert-manager to consume. Name it `issuer.yml` Then create this object with `oc create -f` command.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: vault-issuer
  namespace: default
spec:
  vault:
    server: http://vault.default
    path: pki/sign/example-dot-com
    auth:
      kubernetes:
        mountPath: /v1/auth/kubernetes
        role: issuer
        secretRef:
          name: $ISSUER_SECRET_REF
          key: token
```
```css
$ oc create -f issuer.yml
issuer.cert-manager.io/vault-issuer created
```

Generate a certificate named `example-com`. Create a file, say `certificate.yml` with the contents below. Then create the object with `oc create -f`. 

```css
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: vault-issuer
  commonName: www.example.com
  dnsNames:
  - www.example.com
```
```css
$ oc create -f certificate.yml
certificate.cert-manager.io/example-com created
```

As seen above, the `Certificate`, named example-com, requests from Vault the certificate through the `Issuer`, named `vault-issuer`. The common name and DNS names are names within the allowed domains for the configured Vault endpoint as defined earlier on in the `pki` policy.

We can check if the cerificate is being issued successfully by describing the `certificate` object in the `default` project.

```css
$ oc describe certificate example-com -n default

Name:         example-com
Namespace:    default
## ...
Events:
  Type    Reason        Age   From          Message
  ----    ------        ----  ----          -------
  Normal  GeneratedKey  9m   cert-manager  Generated a new private key
  Normal  Requested     9m   cert-manager  Created new CertificateRequest resource "example-com-1055631450"
  Normal  Issued        10s   cert-manager  Certificate issued successfully
 
```

The `certificate` custom resource also gives information with regards to the expiry details of the certificate issued.

```css
...
Status:
  Conditions:
    Last Transition Time:  2020-07-24T06:22:49Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2020-07-29T06:22:39Z
...
```

As seen above, Vault can be configured in conjunction with a certificate manager like Jetstack's cert-manager. Cert-manager can help to request and issue certificates, obtained dynamically from Vault. Besides creation, certificates can also be revoked and removed. Automated tracking, renewal and issuance of certifcates is definitely very useful in administrating various applications within a Kubernetes cluster. 
