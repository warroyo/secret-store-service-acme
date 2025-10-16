

## Setup a sample intermediate CA

This step is only required if you don't have an official CA that can sign an intermediate cert as an issuing cert. see notes [here](https://openbao.org/docs/secrets/pki/considerations/#be-careful-with-root-cas) about why we do this. 


The easiest way to do this is to use step-ca. you can install the CLI from [here](https://smallstep.com/docs/step-cli/installation/).


1. make directory structure 

```bash
mkdir -p certs
```

2. Generate the root CA

```bash
cd certs
step certificate create "My Root CA" root_ca.crt root_ca.key --profile root-ca --not-after 87600h
```

3. Generate the intermediate

```bash
step certificate create "My Intermediate CA" intermediate_ca.crt intermediate_ca.key \
  --ca root_ca.crt \
  --ca-key root_ca.key \
  --profile intermediate-ca \
  --not-after 43800h
```

4. verify it, this should return no errors

```bash
step certificate verify \
  --roots root_ca.crt \
  intermediate_ca.crt
```

4. create a cert bundle that we will use with OpenBao

```bash
#decrypt the intermediate 
openssl ec -in intermediate_ca.key -out intermediate.key.dec
#create bundle
cat intermediate_ca.crt > cert_bundle.pem
cat intermediate.key.dec >>cert_bundle.pem
cat root_ca.crt >>cert_bundle.pem
```

## Setup the OpenBao PKI endpoint

This will setup an endpoint on OpenBao that uses the intermediate CA from the previous step as a issuing cert.

1. enable the PKI endpoint at a specific path
```bash
bao secrets enable -path=pki_int pki
```

2. upload the intermediate cert bundle as an issuer
```bash
bao write pki_int/issuers/import/bundle pem_bundle=@cert_bundle.pem
```

3. set default issuer

```bash
#get the issuer id
bao list pki_int/issuers
# set the default
bao write pki_int/config/issuers default=<issuer_id>
```

4. set the URL config

```bash
bao write pki_int/config/urls issuing_certificates="https://<secret-store-ip>:8200/v1/pki_int/ca" crl_distribution_points="https://<secret-store-ip>:8200/v1/pki_int/crl"
```

5. update cluster urls

```bash
bao write pki_int/config/cluster path=https://<secret-store-ip>:8200/v1/pki_int
```

6. create a role to issue certs

```bash
bao write pki_int/roles/acme \
    allow_any_name=true \
    allow_subdomains=true max_ttl=72h
```

7. test cert issuance

```bash
bao write pki_int/issue/acme  common_name=blah.example.com
```

## Integrating with cert manager

This will enable acme on the pki endpoint and use cert manager to generate a cert for an application in a workload cluster using http01 challenge

For this to work DNS needs to exist, in this setup I am using contour for ingress on the app and have a wildcard DNS entry setup in from of contour.  To see more on this check out [this argocd app](https://github.com/warroyo/vks-argocd-examples/blob/main/secret-store-issuer-full/app.yml) that deploys contour, cert manager, an issuer that points at the secret store service endpoint, and a sample app with ingress.


1. Enable acme on the pki endpoint

```bash
#enable headers
bao secrets tune \
      -passthrough-request-headers=If-Modified-Since \
      -allowed-response-headers=Last-Modified \
      -allowed-response-headers=Location \
      -allowed-response-headers=Replay-Nonce \
      -allowed-response-headers=Link \
      pki_int
#enable acme
bao write pki_int/config/acme enabled=true
```

2. If you using argocd:

```bash
#deploy the app that installs the pre-reqs like contour cert manager etc.
kubectl apply -f https://raw.githubusercontent.com/warroyo/vks-argocd-examples/refs/heads/main/contour-ingress/app.yml -n <argo-ns>

# get the IP for contour and create a DNS entry

#deploy the issuer and app

kubectl apply -f https://raw.githubusercontent.com/warroyo/vks-argocd-examples/refs/heads/main/secret-store-issuer-full/app.yml -n <argo-ns>

```



3. If not using argocd you can follow these steps. you will need to edit sample-app's ingress to match what you set up in DNS so that the http-01 challenge will work.

```bash
# deploy the package repo for VKS

kubectl apply -f https://raw.githubusercontent.com/warroyo/vks-argocd-examples/refs/heads/main/vks-standard-repo/source/repo.yml

# deploy the rbac needed for package installs

kubectl apply -f https://raw.githubusercontent.com/warroyo/vks-argocd-examples/refs/heads/main/package-rbac/source/rbac.yml

# deploy the cert manager package

kubectl apply -f https://raw.githubusercontent.com/warroyo/vks-argocd-examples/refs/heads/main/cert-manager/source/cert-manager.yml -n infra-packages


# deploy the cert manager issuer

kubectl apply -f https://raw.githubusercontent.com/warroyo/vks-argocd-examples/refs/heads/main/cert-manager/issuers/secret-store-acme/issuer.yml

# deploy contour, at this point you should get the IP and add DNS

kubectl apply -f https://raw.githubusercontent.com/warroyo/vks-argocd-examples/refs/heads/main/contour-ingress/source/contour.yml -n infra-packages

# deploy the app
kubectl apply -f https://github.com/warroyo/vks-argocd-examples/blob/main/secret-store-issuer-full/source/sample-app.yml
