# inlets-http-server

This is the helm chart for an inlets PRO HTTP server

Use it to deploy one or more inlets PRO HTTP tunnel servers to your Kubernetes cluster using a wildcard domain.

# Use your Kubernetes cluster for inlets-pro HTTP exit-servers

Install [arkade](https://arkade.dev/), which is used in the tutorial to install Kubernetes software.

```bash
curl -sLS https://dl.arkade.dev | sh        # Move to /usr/local/bin/
curl -sLS https://dl.arkade.dev | sudo sh   # Moved automatically.
```

Install helm with `arkade get helm`.

You also need to add the helm chart repository:

```bash
$ helm repo add inlets-pro https://inlets.github.io/inlets-pro/charts/
$ helm repo update
```

## Setup cert-manager, Ingress and a DNS01 certificate

```bash
arkade install cert-manager
arkade install ingress-nginx
```

Note that all the resources we will create will be within the `inlets` namespace. cert-manager and ingress-nginx can reside in their own respective namespaces.

Now create a DNS01 issuer for your preferred cloud:

```bash
export EMAIL="you@example.com"
export ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export REGION="eu-central-1"

cat > issuer.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: inlets
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: $EMAIL
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - dns01:
        route53:
          region: $REGION
          accessKeyID: $ACCESS_KEY_ID
          secretAccessKeySecretRef:
            name: prod-route53-credentials-secret
            key: secret-access-key
EOF
```

See other [DNS01 options here](https://cert-manager.io/docs/configuration/acme/dns01/)

Then create a wildcard certificate:

```bash
export DOMAIN=inlets.example.com

cat > certificate.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-inlets-cert
  namespace: inlets
spec:
  secretName: wildcard-inlets-cert
  issuerRef:
    name: letsencrypt-prod
    kind: Issuer
  commonName: '*.$DOMAIN'
  dnsNames:
  - '*.$DOMAIN'
EOF
```

Whatever you place here will be the prefix to any tunnels you create.

I.e. `openfaas.inlets.example.com` and `prometheus.inlets.example.com`

## Setup the tunnel server with helm

Create a secret:

```bash
export NAME=client1

export TOKEN=$(head -c 16 /dev/random | shasum | cut -d" " -f1)
kubectl create secret generic -n inlets inlets-$NAME-token \
  --from-literal token=$TOKEN
```

Create a `values.yaml` and customise the `controlPlaneIngress` with the domain you want the inlets PRO client to connect to.

Then update `dataPlaneIngresses` with any services that you want to expose to the Internet from the tunnel. If you don't want to expose anything then change it to: `dataPlaneIngresses: {}`.

```yaml
controlPlaneIngress:
  domain: client1.exit.o6s.io
  annotations:
    kubernetes.io/ingress.class: "nginx"
  secretName: wildcard-inlets-cert

dataPlaneIngresses:
  - domain: prometheus.exit.o6s.io
    annotations:
      kubernetes.io/ingress.class: "nginx"
    secretName: wildcard-inlets-cert
  - domain: faas.exit.o6s.io
    annotations:
      kubernetes.io/ingress.class: "nginx"
    secretName: wildcard-inlets-cert
fullnameOverride: ""
```

Above: `values-live.yaml`

Since we are using a wildcard TLS record (`wildcard-inlets-router-cert`), this needs to be set as the `secretName`.

Then install the chart:

```bash
export NAME=client1

helm upgrade --namespace inlets \
  --install client1 inlets-pro/inlets-http-server \
  --set tokenSecretName=inlets-$NAME-token \
  -f values-live.yaml
```

Now connect a client:

```bash
# Query $TOKEN
# export $TOKEN=$(kubectl get secret  -n inlets inlets-$NAME-token -o jsonpath={.data.token}|base64 --decode)

inlets-pro http client \
  --token $TOKEN \
  --upstream faas.exit.o6s.io=http://127.0.0.1:8080 \
  --upstream prometheus.exit.o6s.io=http://127.0.0.1:9090 \
  --url wss://client1.exit.o6s.io \
  --auto-tls=false \
  --license-file ~/.inlets/LICENSE
```

Access your tunnelled services:

* https://faas.exit.o6s.io
* https://prometheus.exit.o6s.io
