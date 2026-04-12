---
author: "CRH"
date: "2026-04-04"
modified: "2026-04-04"
description: "Stop creating TLS secrets by hand. cert-manager automates certificate issuance and renewal in Kubernetes."
tags: ["kubernetes", "k8s", "tls", "cert-manager", "lets-encrypt", "gateway-api", "cloud-native"]
title: "Kubernetes for Cloud Engineers: Automating TLS with cert-manager"
type: "post"
bag: true
draft: false
---

*This is part two of a series for new operations and cloud engineers getting into Kubernetes. In [part one](/post/2026-04-04_k8s-tls-default-cert/k8s-tls-default-cert/) we covered how to set default TLS certificates on your ingress controllers. That works, but it means you're managing certs by hand. Let's fix that.*

---

## Why cert-manager?

In the last post, we created TLS secrets manually. That's fine for getting started. It's not fine for production. Certificates expire. People forget to renew them. And then you're getting paged at 3am because your site is serving a browser warning.

[cert-manager](https://cert-manager.io/) is a Kubernetes-native certificate management controller. It watches your cluster for resources that need certificates, talks to a Certificate Authority (usually Let's Encrypt), handles the challenge/response dance, stores the resulting cert as a Kubernetes secret, and renews it before it expires. All automatically. You configure it once and move on with your life.

It's one of those tools that, once installed, you forget it's there. That's the highest compliment I can give infrastructure software.

## Installing cert-manager

Two options. Pick whichever matches your deployment style.

### Option 1: Helm (recommended)

```bash
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.20.0 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

# NAME: cert-manager
# NAMESPACE: cert-manager
# STATUS: deployed
```

### Option 2: Static manifests

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.20.0/cert-manager.yaml
```

Both methods install the same thing: the cert-manager controller, webhook, cainjector, and six CRDs. Those CRDs are the building blocks you'll use for everything else.

```bash
# Verify everything is running
kubectl get pods -n cert-manager

# NAME                                       READY  STATUS
# cert-manager-5c4b5f7b9-xk2lq              1/1    Running
# cert-manager-cainjector-7f694c-m8p         1/1    Running
# cert-manager-webhook-7cd8c8-9tn2f          1/1    Running
```

Three pods running. That's what you want to see.

## Issuers: telling cert-manager where to get certificates

cert-manager doesn't know anything about Let's Encrypt out of the box. You need to tell it where to go and how to prove you own your domains. That's what Issuers do.

There are two flavors:

- **Issuer** is namespace-scoped. It can only issue certs for resources in the same namespace.
- **ClusterIssuer** is cluster-scoped. It works across all namespaces.

For most setups, you want a ClusterIssuer. One config, whole cluster. If you have teams that need isolated certificate management per namespace, use an Issuer. But start simple.

## Staging first. Always.

Let's Encrypt has two environments: staging and production. They work identically, but staging issues certificates signed by a fake CA that browsers don't trust. The certificates themselves are structurally real. They just won't show a green lock.

Why does this matter? Because Let's Encrypt production has [rate limits](https://letsencrypt.org/docs/rate-limits/). Tight ones. 50 certificates per registered domain per week. 5 duplicate certificates per week. If you mess up your config and keep retrying, you can lock yourself out for days.

Staging has much more generous limits. So you should always get your pipeline working against staging first, then switch to production once you know everything is wired up correctly.

### Create a staging ClusterIssuer

```yaml
# staging-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
```

```bash
kubectl apply -f staging-issuer.yaml
# clusterissuer.cert-manager.io/letsencrypt-staging created

# Check that it registered successfully
kubectl get clusterissuer letsencrypt-staging

# NAME                  READY  AGE
# letsencrypt-staging   True   30s
```

`READY: True` means cert-manager registered an account with Let's Encrypt staging. If you see `False`, run `kubectl describe clusterissuer letsencrypt-staging` and read the events.

### Create the production ClusterIssuer

Same thing, different ACME server URL:

```yaml
# production-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
```

Use separate `privateKeySecretRef` names for staging and production. The ACME accounts are completely independent. You can't reuse keys between environments.

## HTTP-01 vs DNS-01: how you prove domain ownership

When you request a certificate, Let's Encrypt needs proof that you actually own the domain. There are two ways to do this, and which one you pick depends on your setup.

### HTTP-01

Let's Encrypt sends an HTTP request to `http://yourdomain.com/.well-known/acme-challenge/<token>`. cert-manager spins up a temporary pod and Ingress (or HTTPRoute) to serve the response. Once validated, the temp resources get cleaned up.

This is the simpler option. It works if:
- Port 80 is publicly reachable on your cluster
- You don't need wildcard certificates

It does not work if:
- You're behind a firewall with no public HTTP access
- You need `*.example.com` certs

### DNS-01

Instead of an HTTP request, Let's Encrypt checks for a specific TXT record at `_acme-challenge.yourdomain.com`. cert-manager talks to your DNS provider's API to create and clean up the record.

This is the only way to get wildcard certificates. It's also the right choice for internal domains that aren't publicly accessible. The tradeoff is more setup. You need to give cert-manager API credentials for your DNS provider, and DNS propagation delays can slow things down.

**Built-in DNS providers:** Cloudflare, Route53, Google Cloud DNS, Azure DNS, DigitalOcean, Akamai, ACMEDNS, and RFC-2136. There are webhook extensions for 30+ more.

## Wiring it up with Ingress

If you're using the traditional Ingress API, cert-manager makes this really clean. Add an annotation to your Ingress resource, include a `tls` block, and cert-manager handles the rest.

```yaml
# my-app-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: my-app
            port:
              number: 80
  tls:
  - hosts:
    - app.example.com
    secretName: app-example-com-tls
```

That's it. cert-manager sees the annotation, reads the `tls` block, creates a Certificate resource, kicks off the ACME challenge, and stores the signed certificate in the `app-example-com-tls` secret. Your ingress controller picks up the secret and starts serving HTTPS.

When the cert is 30 days from expiration, cert-manager renews it automatically.

<div class="k8s-callout">

**Staging first, remember.** Point the annotation at `letsencrypt-staging` until you see a valid (but untrusted) cert in your browser. Then swap it to `letsencrypt-prod`. You'll need to delete the old secret so cert-manager issues a fresh one from production.

</div>

Use `cert-manager.io/cluster-issuer` for ClusterIssuers and `cert-manager.io/issuer` for namespace-scoped Issuers. Mix them up and nothing will happen. No error, no cert. Just silence. Ask me how I know.

## Wiring it up with Gateway API

If you followed the first post in this series, you know Gateway API is where things are heading. cert-manager supports it, but it needs a little extra configuration.

### Enable Gateway API support

Gateway API support is not on by default. You need to opt in.

```bash
helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager \
  --set crds.enabled=true \
  --set config.enableGatewayAPI=true
```

Make sure the Gateway API CRDs are installed in your cluster too:

```bash
kubectl apply --server-side \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/standard-install.yaml
```

If you installed the CRDs after cert-manager was already running, restart it so it picks them up:

```bash
kubectl rollout restart deployment cert-manager -n cert-manager
```

### Annotate your Gateway

The pattern is similar to Ingress. Add the annotation, reference a secret in the listener's `certificateRefs`, and cert-manager fills in the rest.

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    hostname: app.example.com
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: app-example-com-tls
```

cert-manager sees the annotation and the listener config, creates a Certificate for `app.example.com`, and stores the result in the `app-example-com-tls` secret. The Gateway picks it up and starts terminating TLS.

Three things must be true for cert-manager to act on a Gateway listener:
1. The `hostname` is not empty
2. The TLS `mode` is `Terminate` (not `Passthrough`)
3. There's at least one entry in `certificateRefs`

Miss any of those and cert-manager quietly ignores the listener.

## Wildcard certificates with DNS-01

This is where DNS-01 earns its keep. Let's say you want a single cert that covers `*.example.com` so every subdomain is automatically secured. HTTP-01 can't do this. Only DNS-01 can.

I'll use Cloudflare as the DNS provider since it's common (and because I have a love/hate relationship with them that we've already discussed on the [software page](/software/)).

### Step 1: Create a Cloudflare API token

In the Cloudflare dashboard, create an API token with these permissions:
- **Zone / DNS / Edit**
- **Zone / Zone / Read**

Scope it to the zone you need. Don't use a global API key if you can avoid it.

### Step 2: Store the token in your cluster

```yaml
# The secret MUST be in the cert-manager namespace for ClusterIssuers
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token
  namespace: cert-manager
type: Opaque
stringData:
  api-token: your-cloudflare-api-token-here
```

That namespace bit is important. When you use a ClusterIssuer, cert-manager looks for referenced secrets in the namespace where cert-manager itself is installed. Not the namespace of your Certificate. Not the namespace of your app. The cert-manager namespace. This trips up everyone at least once.

### Step 3: Create a DNS-01 ClusterIssuer

```yaml
# dns-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod-dns
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-dns-account-key
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare-api-token
            key: api-token
```

### Step 4: Request the wildcard cert

```yaml
# wildcard-cert.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-example-com
  namespace: default
spec:
  secretName: wildcard-example-com-tls
  issuerRef:
    name: letsencrypt-prod-dns
    kind: ClusterIssuer
  dnsNames:
  - "*.example.com"
  - example.com
```

Include both `*.example.com` and `example.com` in the `dnsNames`. The wildcard only covers subdomains, not the bare domain itself.

```bash
kubectl apply -f wildcard-cert.yaml
# certificate.cert-manager.io/wildcard-example-com created

# Watch it work
kubectl get certificate wildcard-example-com -w

# NAME                    READY  SECRET                        AGE
# wildcard-example-com    False  wildcard-example-com-tls      5s
# wildcard-example-com    True   wildcard-example-com-tls      47s
```

DNS-01 is slower than HTTP-01 because of DNS propagation. Give it a minute or two. If it takes more than five minutes, start troubleshooting (see below).

## When things go wrong

They will. Here's how to figure out what happened.

cert-manager has a chain of resources that it creates when processing a certificate request. Follow the chain from top to bottom:

```bash
# 1. Is the Certificate ready?
kubectl get certificates -A

# 2. What does the CertificateRequest say?
kubectl get certificaterequest -A
kubectl describe certificaterequest <name> -n <namespace>

# 3. Is the Issuer healthy?
kubectl get clusterissuer
kubectl describe clusterissuer <name>

# 4. For Let's Encrypt: check the ACME Order and Challenge
kubectl get orders -A
kubectl get challenges -A
kubectl describe challenge <name> -n <namespace>

# 5. Check the cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager --tail=100
```

Most problems fall into a handful of categories:

**Challenge not reachable (HTTP-01).** Port 80 isn't open, or the temporary solver Ingress isn't being picked up by your controller. Check that `ingressClassName` in your solver config matches your actual ingress controller.

**DNS propagation timeout (DNS-01).** The TXT record was created but Let's Encrypt can't see it yet. If your cluster's DNS resolver is slow, you can point cert-manager at public resolvers:

```bash
helm upgrade cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager \
  --set 'extraArgs={--dns01-recursive-nameservers-only,--dns01-recursive-nameservers=1.1.1.1:53\,9.9.9.9:53}'
```

**Secret in the wrong namespace.** The number one DNS-01 headache. ClusterIssuers look for API token secrets in the cert-manager namespace. Not your app namespace. Not default.

**Rate limited.** You hit Let's Encrypt production limits. Switch back to staging, wait for the rate limit window to pass (usually 7 days), fix whatever caused the excessive requests, and try again.

**Issuer not ready.** `kubectl describe clusterissuer` will tell you why. Usually it's a bad email address, an unreachable ACME server, or a malformed `privateKeySecretRef`.

**Nothing happens at all.** Check the annotation name. `cert-manager.io/cluster-issuer` is not the same as `cert-manager.io/issuer`. Using the wrong one for your Issuer type produces zero errors and zero certificates. It's the most frustrating failure mode because there's nothing in the logs.

## A note on certificate lifetimes

Let's Encrypt has been shortening certificate lifetimes. They started at 90 days, moved to 64, and are now pushing toward 45-day certificates. This doesn't matter much if you're using cert-manager because it handles renewal automatically (default is 30 days before expiration). But it does mean that if cert-manager breaks and you don't notice, you have less runway before things go red.

Monitor your certificates. At minimum, set up an alert on cert-manager pod health. Ideally, also alert on certificates that haven't renewed within their expected window.

## Wrapping up

Here's the workflow once everything is configured:

1. Deploy an Ingress or Gateway with the cert-manager annotation
2. cert-manager creates a Certificate resource
3. cert-manager talks to Let's Encrypt, completes the challenge
4. The signed certificate lands in a Kubernetes secret
5. Your ingress controller or gateway picks it up
6. cert-manager renews it before expiration

No cron jobs. No manual `certbot renew`. No calendar reminders. Just certificates that work.

If you're running through this series from the beginning, you now have default TLS certs on your ingress controllers and automated certificate management with Let's Encrypt. That's a solid foundation.

Next up in this series: **DNS automation with external-dns.** Because if cert-manager removes the manual work from certificates, external-dns does the same thing for DNS records.
