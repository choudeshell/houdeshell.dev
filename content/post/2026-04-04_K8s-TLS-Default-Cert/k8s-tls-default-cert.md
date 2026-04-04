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

<div class="k8s-terminal">
<span class="comment"># Create the TLS secret from your cert and key files</span><br>
<span class="prompt">$</span> <span class="cmd">kubectl create secret tls default-tls \</span><br>
<span class="cmd">&nbsp;&nbsp;--cert=tls.crt \</span><br>
<span class="cmd">&nbsp;&nbsp;--key=tls.key \</span><br>
<span class="cmd">&nbsp;&nbsp;-n default</span><br>
<br>
<span class="output">secret/default-tls created</span>
</div>

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

<div class="k8s-terminal">
<span class="comment"># values.yaml</span><br>
<span class="cmd">controller:</span><br>
<span class="cmd">&nbsp;&nbsp;extraArgs:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;default-ssl-certificate: "default/default-tls"</span>
</div>

<div class="k8s-terminal">
<span class="prompt">$</span> <span class="cmd">helm upgrade ingress-nginx ingress-nginx/ingress-nginx \</span><br>
<span class="cmd">&nbsp;&nbsp;-f values.yaml \</span><br>
<span class="cmd">&nbsp;&nbsp;-n ingress-nginx</span>
</div>

**Without Helm** — patch the controller deployment directly:

<div class="k8s-terminal">
<span class="comment"># Add to the container args in the ingress-nginx-controller Deployment</span><br>
<span class="cmd">spec:</span><br>
<span class="cmd">&nbsp;&nbsp;containers:</span><br>
<span class="cmd">&nbsp;&nbsp;- name: controller</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;args:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;- /nginx-ingress-controller</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;- --default-ssl-certificate=default/default-tls</span>
</div>

The format is `namespace/secret-name`. Without this flag, NGINX generates a self-signed certificate for the catch-all server — that's the browser warning you're seeing.

**Gotcha:** If your Ingress resource has a `tls:` section but no `secretName`, NGINX uses this default cert and forces an HTTPS redirect. If the `tls:` section is missing entirely, it still serves the default cert but does *not* redirect. Use `force-ssl-redirect: "true"` in the ConfigMap if you want to enforce HTTPS everywhere.

---

### HAProxy Ingress

There are two HAProxy ingress projects — the community [haproxy-ingress](https://haproxy-ingress.github.io/) and the one from [HAProxy Technologies](https://github.com/haproxytech/kubernetes-ingress). Both support the same flag pattern.

**With Helm (HAProxy Technologies):**

<div class="k8s-terminal">
<span class="comment"># values.yaml</span><br>
<span class="cmd">controller:</span><br>
<span class="cmd">&nbsp;&nbsp;defaultTLSSecret:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;secretNamespace: default</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;secret: default-tls</span>
</div>

**With controller args (both projects):**

<div class="k8s-terminal">
<span class="comment"># Same flag format as NGINX</span><br>
<span class="cmd">--default-ssl-certificate=default/default-tls</span>
</div>

<div class="k8s-terminal">
<span class="prompt">$</span> <span class="cmd">helm upgrade haproxy-ingress haproxytech/kubernetes-ingress \</span><br>
<span class="cmd">&nbsp;&nbsp;--set controller.defaultTLSSecret.secretNamespace=default \</span><br>
<span class="cmd">&nbsp;&nbsp;--set controller.defaultTLSSecret.secret=default-tls \</span><br>
<span class="cmd">&nbsp;&nbsp;-n haproxy-ingress</span>
</div>

Without a default cert configured, both HAProxy implementations generate a self-signed fake certificate — same story as NGINX.

**Note:** HAProxy Technologies controller v1.11+ automatically enables QUIC (HTTP/3) when you set a default TLS cert. If you don't want that yet, add `--disable-quic`.

---

### Istio Ingress Gateway

Istio doesn't use the Kubernetes Ingress API at all. It has its own `Gateway` CRD (not the same as Gateway API — I know, the naming is painful).

**Important:** The TLS secret **must live in the same namespace as the Istio ingress gateway pod** — typically `istio-system`.

<div class="k8s-terminal">
<span class="prompt">$</span> <span class="cmd">kubectl create secret tls default-tls \</span><br>
<span class="cmd">&nbsp;&nbsp;--cert=tls.crt \</span><br>
<span class="cmd">&nbsp;&nbsp;--key=tls.key \</span><br>
<span class="cmd">&nbsp;&nbsp;-n istio-system</span><br>
<br>
<span class="output">secret/default-tls created</span>
</div>

Then create the Gateway resource:

<div class="k8s-terminal">
<span class="comment"># istio-gateway.yaml</span><br>
<span class="cmd">apiVersion: networking.istio.io/v1</span><br>
<span class="cmd">kind: Gateway</span><br>
<span class="cmd">metadata:</span><br>
<span class="cmd">&nbsp;&nbsp;name: default-gateway</span><br>
<span class="cmd">spec:</span><br>
<span class="cmd">&nbsp;&nbsp;selector:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;istio: ingressgateway</span><br>
<span class="cmd">&nbsp;&nbsp;servers:</span><br>
<span class="cmd">&nbsp;&nbsp;- port:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;number: 443</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;name: https</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;protocol: HTTPS</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;tls:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;mode: SIMPLE</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;credentialName: default-tls</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;hosts:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;- "*.example.com"</span>
</div>

<div class="k8s-terminal">
<span class="prompt">$</span> <span class="cmd">kubectl apply -f istio-gateway.yaml</span><br>
<span class="output">gateway.networking.istio.io/default-gateway created</span>
</div>

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

<div class="k8s-terminal">
<span class="comment"># gateway.yaml</span><br>
<span class="cmd">apiVersion: gateway.networking.k8s.io/v1</span><br>
<span class="cmd">kind: Gateway</span><br>
<span class="cmd">metadata:</span><br>
<span class="cmd">&nbsp;&nbsp;name: default-gateway</span><br>
<span class="cmd">spec:</span><br>
<span class="cmd">&nbsp;&nbsp;gatewayClassName: nginx</span><br>
<span class="cmd">&nbsp;&nbsp;listeners:</span><br>
<span class="cmd">&nbsp;&nbsp;- name: http</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;port: 80</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;protocol: HTTP</span><br>
<span class="cmd">&nbsp;&nbsp;- name: https</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;hostname: "*.example.com"</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;port: 443</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;protocol: HTTPS</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;tls:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;mode: Terminate</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;certificateRefs:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- kind: Secret</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;name: default-tls</span>
</div>

<div class="k8s-terminal">
<span class="prompt">$</span> <span class="cmd">kubectl apply -f gateway.yaml</span><br>
<span class="output">gateway.gateway.networking.k8s.io/default-gateway created</span>
</div>

Then attach your HTTPRoutes to the HTTPS listener:

<div class="k8s-terminal">
<span class="comment"># route.yaml</span><br>
<span class="cmd">apiVersion: gateway.networking.k8s.io/v1</span><br>
<span class="cmd">kind: HTTPRoute</span><br>
<span class="cmd">metadata:</span><br>
<span class="cmd">&nbsp;&nbsp;name: my-app</span><br>
<span class="cmd">spec:</span><br>
<span class="cmd">&nbsp;&nbsp;parentRefs:</span><br>
<span class="cmd">&nbsp;&nbsp;- name: default-gateway</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;sectionName: https</span><br>
<span class="cmd">&nbsp;&nbsp;hostnames:</span><br>
<span class="cmd">&nbsp;&nbsp;- "app.example.com"</span><br>
<span class="cmd">&nbsp;&nbsp;rules:</span><br>
<span class="cmd">&nbsp;&nbsp;- backendRefs:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;- name: my-app</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;port: 80</span>
</div>

**With cert-manager** — add an annotation and cert-manager handles the rest:

<div class="k8s-terminal">
<span class="comment"># gateway.yaml — cert-manager will create and manage the secret</span><br>
<span class="cmd">apiVersion: gateway.networking.k8s.io/v1</span><br>
<span class="cmd">kind: Gateway</span><br>
<span class="cmd">metadata:</span><br>
<span class="cmd">&nbsp;&nbsp;name: default-gateway</span><br>
<span class="cmd">&nbsp;&nbsp;annotations:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;cert-manager.io/cluster-issuer: letsencrypt-prod</span><br>
<span class="cmd">spec:</span><br>
<span class="cmd">&nbsp;&nbsp;gatewayClassName: nginx</span><br>
<span class="cmd">&nbsp;&nbsp;listeners:</span><br>
<span class="cmd">&nbsp;&nbsp;- name: https</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;hostname: "app.example.com"</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;port: 443</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;protocol: HTTPS</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;tls:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;mode: Terminate</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;certificateRefs:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- kind: Secret</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;name: default-tls</span>
</div>

---

### HAProxy (Gateway API)

The beauty of Gateway API is that it's a standard. The Gateway resource looks almost identical regardless of the underlying implementation — you just swap the `gatewayClassName`.

<div class="k8s-terminal">
<span class="comment"># gateway.yaml — same pattern, different class</span><br>
<span class="cmd">apiVersion: gateway.networking.k8s.io/v1</span><br>
<span class="cmd">kind: Gateway</span><br>
<span class="cmd">metadata:</span><br>
<span class="cmd">&nbsp;&nbsp;name: default-gateway</span><br>
<span class="cmd">spec:</span><br>
<span class="cmd">&nbsp;&nbsp;gatewayClassName: haproxy</span><br>
<span class="cmd">&nbsp;&nbsp;listeners:</span><br>
<span class="cmd">&nbsp;&nbsp;- name: https</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;hostname: "*.example.com"</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;port: 443</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;protocol: HTTPS</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;tls:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;mode: Terminate</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;certificateRefs:</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- kind: Secret</span><br>
<span class="cmd">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;name: default-tls</span>
</div>

<div class="k8s-terminal">
<span class="prompt">$</span> <span class="cmd">kubectl apply -f gateway.yaml</span><br>
<span class="output">gateway.gateway.networking.k8s.io/default-gateway created</span>
</div>

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
