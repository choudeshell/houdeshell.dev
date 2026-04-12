---
author: "CRH"
date: "2026-04-04"
modified: "2026-04-04"
description: "How to set the default TLS certificate for your Kubernetes ingress controllers and Gateway API implementations."
tags: ["kubernetes", "k8s", "tls", "ingress", "gateway-api", "cloud-native"]
title: "Kubernetes for Cloud Engineers: Default TLS Certificates"
type: "post"
bag: true
draft: false
---

*This is the first post in a series for new operations and cloud engineers getting started with Kubernetes. Whether you're running K3s on a Raspberry Pi, EKS in AWS, AKS in Azure, or anything in between — the concepts are the same. Let's get into it.*

---

## The problem

You've got a Kubernetes cluster. You've got an ingress controller routing traffic. You hit your app over HTTPS and... you get a browser warning about an untrusted self-signed certificate. Or worse, you've got ten services and you're copy-pasting the same TLS secret into every single Ingress resource.

The fix: **set a default TLS certificate** on your ingress controller. One cert to rule them all. Any request that doesn't match a more specific TLS config falls back to this default. Clean, simple, done.

Every controller does this slightly differently. Let's walk through each one.

---

## First — create your TLS secret

This part is the same regardless of which controller you're running. You need a Kubernetes TLS secret containing your certificate and private key.

```bash
# Create the TLS secret from your cert and key files
kubectl create secret tls default-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n default

# secret/default-tls created
```

Your cert file should contain the full chain: leaf → intermediate → root. If you're using cert-manager or Let's Encrypt, this is handled for you. If you're doing it by hand — get the order right or you'll be debugging trust chain errors at 2am.

---

## Ingress Controllers (Legacy Ingress API)

<div class="k8s-callout warning">

**Heads up: Ingress NGINX is retired.** The `kubernetes/ingress-nginx` project — the one most of us grew up on — officially reached end-of-life on **March 24, 2026**. The repository is now read-only. No more releases, no bug fixes, **no security patches**. Your existing deployments won't break overnight, but you're flying without a safety net.

The Kubernetes Steering Committee and Security Response Committee issued a [joint statement](https://kubernetes.io/blog/2026/01/29/ingress-nginx-statement/) in January 2026 urging migration. The recommended path forward is **Gateway API**, which has been GA since October 2023 and now has 20+ conformant implementations.

If you're starting fresh — skip to the [Gateway API section](#gateway-api) below. If you're migrating, check out `ingress2gateway` — the official migration tool that now supports 30+ annotation conversions.

**Note:** NGINX Inc.'s commercial ingress controller (F5/NGINX) is a completely separate codebase and remains actively maintained. Don't confuse the two.

</div>

### Ingress NGINX (kubernetes/ingress-nginx)

Even though it's retired, you'll encounter this in the wild for a while. The default cert is set via a controller argument.

**With Helm:**

```yaml
# values.yaml
controller:
  extraArgs:
    default-ssl-certificate: "default/default-tls"
```

```bash
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  -f values.yaml \
  -n ingress-nginx
```

**Without Helm** — patch the controller deployment directly:

```yaml
# Add to the container args in the ingress-nginx-controller Deployment
spec:
  containers:
  - name: controller
    args:
    - /nginx-ingress-controller
    - --default-ssl-certificate=default/default-tls
```

The format is `namespace/secret-name`. Without this flag, NGINX generates a self-signed certificate for the catch-all server — that's the browser warning you're seeing.

**Gotcha:** If your Ingress resource has a `tls:` section but no `secretName`, NGINX uses this default cert and forces an HTTPS redirect. If the `tls:` section is missing entirely, it still serves the default cert but does *not* redirect. Use `force-ssl-redirect: "true"` in the ConfigMap if you want to enforce HTTPS everywhere.

---

### HAProxy Ingress

There are two HAProxy ingress projects — the community [haproxy-ingress](https://haproxy-ingress.github.io/) and the one from [HAProxy Technologies](https://github.com/haproxytech/kubernetes-ingress). Both support the same flag pattern.

**With Helm (HAProxy Technologies):**

```yaml
# values.yaml
controller:
  defaultTLSSecret:
    secretNamespace: default
    secret: default-tls
```

**With controller args (both projects):**

```text
--default-ssl-certificate=default/default-tls
```

```bash
helm upgrade haproxy-ingress haproxytech/kubernetes-ingress \
  --set controller.defaultTLSSecret.secretNamespace=default \
  --set controller.defaultTLSSecret.secret=default-tls \
  -n haproxy-ingress
```

Without a default cert configured, both HAProxy implementations generate a self-signed fake certificate — same story as NGINX.

**Note:** HAProxy Technologies controller v1.11+ automatically enables QUIC (HTTP/3) when you set a default TLS cert. If you don't want that yet, add `--disable-quic`.

---

### Istio Ingress Gateway

Istio doesn't use the Kubernetes Ingress API at all. It has its own `Gateway` CRD (not the same as Gateway API — I know, the naming is painful).

**Important:** The TLS secret **must live in the same namespace as the Istio ingress gateway pod** — typically `istio-system`.

```bash
kubectl create secret tls default-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n istio-system

# secret/default-tls created
```

Then create the Gateway resource:

```yaml
# istio-gateway.yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: default-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: default-tls
    hosts:
    - "*.example.com"
```

```bash
kubectl apply -f istio-gateway.yaml
# gateway.networking.istio.io/default-gateway created
```

**Gotchas:**
- `credentialName` must exactly match the Kubernetes secret name. No `namespace/name` format here — it just looks in its own namespace.
- Istio doesn't have a single "default certificate" concept. You configure TLS per-server block on the Gateway. To make it act as a default, use a wildcard host like `*.example.com`.
- Wrong namespace is the #1 debugging headache. If TLS isn't working, check the secret namespace first.

---

## Gateway API

Gateway API is the future. It's the Kubernetes-native standard from SIG-Network, GA since October 2023, and every major ingress controller is building an implementation. If you're starting fresh — start here.

The big difference: there's no `--default-ssl-certificate` flag. TLS is configured **declaratively** on the Gateway resource itself, per-listener. Each HTTPS listener explicitly references a certificate via `certificateRefs`.

### NGINX Gateway Fabric

NGINX Gateway Fabric is NGINX's Gateway API implementation — completely separate from the retired Ingress NGINX project.

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: default-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
  - name: https
    hostname: "*.example.com"
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: default-tls
```

```bash
kubectl apply -f gateway.yaml
# gateway.gateway.networking.k8s.io/default-gateway created
```

Then attach your HTTPRoutes to the HTTPS listener:

```yaml
# route.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
spec:
  parentRefs:
  - name: default-gateway
    sectionName: https
  hostnames:
  - "app.example.com"
  rules:
  - backendRefs:
    - name: my-app
      port: 80
```

**With cert-manager** — add an annotation and cert-manager handles the rest:

```yaml
# gateway.yaml — cert-manager will create and manage the secret
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: default-gateway
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    hostname: "app.example.com"
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: default-tls
```

---

### HAProxy (Gateway API)

The beauty of Gateway API is that it's a standard. The Gateway resource looks almost identical regardless of the underlying implementation — you just swap the `gatewayClassName`.

```yaml
# gateway.yaml — same pattern, different class
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: default-gateway
spec:
  gatewayClassName: haproxy
  listeners:
  - name: https
    hostname: "*.example.com"
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: default-tls
```

```bash
kubectl apply -f gateway.yaml
# gateway.gateway.networking.k8s.io/default-gateway created
```

That's it. Same spec. Same structure. The `gatewayClassName` tells Kubernetes which controller reconciles the resource. This is exactly why Gateway API is the future — you can swap implementations without rewriting your config.

**Gotchas for both implementations:**
- Gateway API doesn't have a global "default certificate" flag. Each HTTPS listener must explicitly reference a cert via `certificateRefs`. To cover everything, use a wildcard hostname.
- The TLS secret must be in the **same namespace** as the Gateway by default. For cross-namespace references, create a `ReferenceGrant`.
- TLS modes: `Terminate` means the gateway decrypts traffic (most common). `Passthrough` forwards encrypted traffic as-is — use this with `TLSRoute`, not `HTTPRoute`.
- NGINX Gateway Fabric currently supports only a single `certificateRef` per listener. If you need multiple certs, create multiple listeners.

---

## Wrapping up

Here's the cheat sheet:

| Controller | Method | Format |
|---|---|---|
| Ingress NGINX *(retired)* | `--default-ssl-certificate` | `namespace/secret` |
| HAProxy Ingress | `--default-ssl-certificate` | `namespace/secret` |
| Istio | `credentialName` on Gateway server | secret name (same namespace) |
| Gateway API (any impl) | `certificateRefs` on listener | secret reference |

If you're starting fresh — go straight to Gateway API. Pick an implementation (NGINX Gateway Fabric, HAProxy, Envoy Gateway, whatever fits your stack), define your Gateway with a TLS listener, and move on.

If you're migrating from Ingress NGINX, take a breath. Your cluster won't explode tomorrow. But start planning the move. The `ingress2gateway` tool can automate most of the conversion, and the Gateway API spec is stable and well-documented.

Next up in this series: **cert-manager and automated TLS** — because creating secrets by hand is so 2024.
