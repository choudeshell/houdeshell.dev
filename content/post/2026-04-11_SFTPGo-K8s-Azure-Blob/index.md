---
author: "CRH"
date: "2026-04-11"
description: "Running SFTPGo on Kubernetes with Azure Blob Storage as a virtual folder backend, because Azure's native SFTP support costs $219/month for the privilege."
tags: ["kubernetes", "azure", "sftp", "sftpgo"]
title: "SFTPGo on Kubernetes: Azure Blob Without the $219/Month SFTP Tax"
type: "post"
draft: false
---

Someone needed SFTP access to Azure Blob Storage. Simple enough, right? Azure even has native SFTP support built into storage accounts now. Problem solved.

Until you look at the price tag.

### The Azure SFTP Tax

Azure's native SFTP support for Blob Storage costs **$0.30 per hour**. That's roughly **$219 per month**. Just for the endpoint. Just for it to *exist*. You haven't transferred a single file yet. That's before storage costs, before transaction costs, before anything useful happens.

Oh, and it requires **hierarchical namespace** (Azure Data Lake Storage Gen2), which means you can't just flip it on for an existing standard Blob Storage account. You need ADLS Gen2 from the start, or you're creating a new storage account.

For a high-throughput enterprise workload moving terabytes a day, $219/month is a rounding error. For the rest of us who just need to let a vendor or partner drop some files into a blob container? That's an absurd premium for what amounts to a protocol adapter.

### SFTPGo: The Alternative

[SFTPGo](https://github.com/drakkan/sftpgo) is an open-source, fully featured SFTP server written in Go. It supports virtual folders backed by local filesystem, S3, Google Cloud Storage, and Azure Blob Storage. It handles SSH keys, password auth, per-user quotas, bandwidth limits, and a web admin UI. It's also a single binary that runs great in a container.

The plan: run SFTPGo on Kubernetes, point virtual folders at Azure Blob Storage, and skip the $219/month endpoint tax entirely.

### The Kubernetes Setup

Here's what the deployment looks like. Nothing exotic. A Deployment, a couple of Services, a ConfigMap, some Secrets, and a PostgreSQL database for SFTPGo's user/config store.

#### Namespace

Keep it tidy:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sftpgo
```

#### The Secrets

Three secrets. Azure Blob credentials, PostgreSQL connection, and SSH host keys.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sftpgo-azure
  namespace: sftpgo
type: Opaque
stringData:
  SFTPGO_AZBLOB_CONTAINER: "your-container-name"
  SFTPGO_AZBLOB_ACCOUNT_NAME: "yourstorageaccount"
  SFTPGO_AZBLOB_ACCOUNT_KEY: "your-storage-account-key"
---
apiVersion: v1
kind: Secret
metadata:
  name: sftpgo-db
  namespace: sftpgo
type: Opaque
stringData:
  SFTPGO_DATA_PROVIDER__DRIVER: "postgresql"
  SFTPGO_DATA_PROVIDER__NAME: "postgresql://sftpgo:your-db-password@sftpgo-db:5432/sftpgo?sslmode=disable"
```

If you're using Managed Identity for the Azure side (and you should be if you can), you can skip the account key and rely on the pod's identity. But that's a topic for another post.

SFTPGo reads its data provider config from environment variables using the `SFTPGO_DATA_PROVIDER__` prefix. The double underscore maps to the nested JSON structure. Neat trick that saves you from templating JSON.

The host keys get their own secret. Generate them once, store them in the cluster, and every pod (current and future replicas) uses the same keys. If you skip this step and let SFTPGo generate keys on startup, every pod restart hands clients a different fingerprint. That `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` message trains users to blindly accept key changes, which is the opposite of what SSH security is for.

```bash
# Generate the keys locally
ssh-keygen -t ed25519 -f id_ed25519 -N ""
ssh-keygen -t rsa -b 4096 -f id_rsa -N ""

# Create the secret
kubectl create secret generic sftpgo-hostkeys -n sftpgo \
  --from-file=id_ed25519 \
  --from-file=id_rsa

# Clean up local copies
rm id_ed25519 id_ed25519.pub id_rsa id_rsa.pub
```

#### The ConfigMap

SFTPGo has a mountain of configuration options. The data provider config is handled by the env vars in the secret above, so the ConfigMap just needs the SFTP, HTTP, and telemetry bindings:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sftpgo-config
  namespace: sftpgo
data:
  sftpgo.json: |
    {
      "sftpd": {
        "bindings": [{ "port": 2022, "address": "" }],
        "host_keys": [
          "/var/lib/sftpgo/id_ed25519",
          "/var/lib/sftpgo/id_rsa"
        ]
      },
      "httpd": {
        "bindings": [{
          "port": 8080,
          "address": "",
          "enable_web_admin": true,
          "enable_web_client": true
        }]
      },
      "telemetry": {
        "bind_port": 10000,
        "bind_address": "",
        "enable_profiler": false,
        "auth_user_file": ""
      }
    }
```

Port `2022` for SFTP, port `8080` for the web admin, port `10000` for the Prometheus metrics endpoint. PostgreSQL connection details come from the `sftpgo-db` secret, so they stay out of the ConfigMap where they belong.

#### PostgreSQL

SFTPGo needs a database for users, folders, and configuration. PostgreSQL is the right choice here. It handles concurrent access without drama, and if you ever want to scale SFTPGo beyond a single replica, you'll need it.

A simple StatefulSet gets the job done:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sftpgo-pg
  namespace: sftpgo
type: Opaque
stringData:
  POSTGRES_USER: "sftpgo"
  POSTGRES_PASSWORD: "your-db-password"
  POSTGRES_DB: "sftpgo"
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sftpgo-db
  namespace: sftpgo
spec:
  serviceName: sftpgo-db
  replicas: 1
  selector:
    matchLabels:
      app: sftpgo-db
  template:
    metadata:
      labels:
        app: sftpgo-db
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: sftpgo-pg
          volumeMounts:
            - name: pgdata
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
  volumeClaimTemplates:
    - metadata:
        name: pgdata
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi
---
apiVersion: v1
kind: Service
metadata:
  name: sftpgo-db
  namespace: sftpgo
spec:
  clusterIP: None
  selector:
    app: sftpgo-db
  ports:
    - port: 5432
      targetPort: 5432
```

A headless Service gives SFTPGo a stable `sftpgo-db:5432` endpoint to connect to. The StatefulSet's `volumeClaimTemplates` handle persistent storage so your data survives pod restarts.

If you're already running a managed PostgreSQL instance (Azure Database for PostgreSQL, RDS, etc.), just point the connection string at that and skip this entirely.

#### The Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sftpgo
  namespace: sftpgo
  labels:
    app: sftpgo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sftpgo
  template:
    metadata:
      labels:
        app: sftpgo
    spec:
      containers:
        - name: sftpgo
          image: drakkan/sftpgo:latest
          ports:
            - containerPort: 2022
              name: sftp
            - containerPort: 8080
              name: http
            - containerPort: 10000
              name: metrics
          envFrom:
            - secretRef:
                name: sftpgo-azure
            - secretRef:
                name: sftpgo-db
          volumeMounts:
            - name: config
              mountPath: /etc/sftpgo/sftpgo.json
              subPath: sftpgo.json
            - name: hostkeys
              mountPath: /var/lib/sftpgo/id_ed25519
              subPath: id_ed25519
              readOnly: true
            - name: hostkeys
              mountPath: /var/lib/sftpgo/id_rsa
              subPath: id_rsa
              readOnly: true
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
      volumes:
        - name: config
          configMap:
            name: sftpgo-config
        - name: hostkeys
          secret:
            secretName: sftpgo-hostkeys
            defaultMode: 0600
```

No PVC needed. Config comes from a ConfigMap, credentials from Secrets, host keys from a Secret, and user data lives in PostgreSQL. The Deployment is completely stateless, which means Kubernetes can reschedule pods freely and scaling replicas is trivial (more on that below).

#### The Service

The Service is internal only. ClusterIP, not LoadBalancer. We don't want the admin UI anywhere near the public internet.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sftpgo
  namespace: sftpgo
spec:
  type: ClusterIP
  selector:
    app: sftpgo
  ports:
    - name: sftp
      port: 2022
      targetPort: 2022
      protocol: TCP
    - name: http
      port: 8080
      targetPort: 8080
      protocol: TCP
    - name: metrics
      port: 10000
      targetPort: 10000
      protocol: TCP
```

To access the admin UI, use `kubectl port-forward`:

```bash
kubectl port-forward -n sftpgo svc/sftpgo 8080:8080
```

Then open `http://localhost:8080/web/admin` in your browser. This keeps the admin UI off the network entirely. No Ingress, no TLS cert to manage, no authentication proxy to configure. Just a direct tunnel from your machine to the pod when you need it.

#### The Ingress

SFTP needs to be reachable from the outside, so we expose *only* the SFTP port. Most ingress controllers support TCP/UDP proxying alongside HTTP. With nginx-ingress, you configure it through a TCP services ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  "2222": "sftpgo/sftpgo:2022"
```

This tells the ingress controller to listen on port `2222` externally and forward TCP traffic to `sftpgo:2022` in the `sftpgo` namespace. Make sure your ingress controller's Service and Deployment are configured to expose this port. You'll need to add it to the controller's `--tcp-services-configmap` arg and open the port on the controller's Service:

```yaml
# Patch the ingress-nginx controller Service to expose port 2222
# (or add it to your existing Service definition)
spec:
  ports:
    - name: sftp-proxy
      port: 2222
      targetPort: 2222
      protocol: TCP
```

Your clients connect with `sftp -P 2222 user@your-ingress-ip`. The admin UI stays completely internal. The only thing exposed to the internet is the SFTP port, which is exactly the attack surface you want: SSH protocol with key-based auth. Not a web UI with a login form.

### Configuring the Azure Blob Virtual Folder

Once everything is running, port-forward into the admin UI and create a user:

```bash
kubectl port-forward -n sftpgo svc/sftpgo 8080:8080
```

Open `http://localhost:8080/web/admin`. The default admin credentials are in the SFTPGo docs (change them immediately, obviously).

The fun part: virtual folders. When you create or edit a user, you can map a virtual path to an Azure Blob container. In the SFTPGo admin:

1. Go to your user's settings
2. Under **Virtual Folders**, add a new folder
3. Set the **mapped path** (e.g., `/uploads`)
4. Choose **Azure Blob Storage** as the filesystem
5. Fill in the container name and credentials (or let the env vars handle it)

Now when a client connects via SFTP and `cd /uploads`, they're reading and writing directly to Azure Blob Storage. The client has no idea. They just see files and directories.

You can also do this via the REST API if you're automating user provisioning:

```bash
curl -X POST http://localhost:8080/api/v2/users \
  -H "Content-Type: application/json" \
  -u admin:password \
  -d '{
    "username": "vendor-uploads",
    "password": "a-strong-password-please",
    "permissions": { "/": ["*"] },
    "virtual_folders": [
      {
        "name": "azure-uploads",
        "mapped_path": "/uploads",
        "virtual_path": "/uploads",
        "filesystem": {
          "provider": 3,
          "azblobconfig": {
            "container": "vendor-data",
            "account_name": "yourstorageaccount",
            "account_key": { "payload": "base64-key-here" }
          }
        }
      }
    ]
  }'
```

Provider `3` is Azure Blob Storage in SFTPGo's config. Provider `1` is S3, `2` is Google Cloud Storage, `0` is local. The kind of magic number situation that makes you love (and occasionally curse) open source software.

### What You Get

For the cost of a small pod running on your existing cluster, you now have:

- **SFTP access to Azure Blob Storage** without the $219/month Azure tax
- **Per-user access control** with SSH keys or passwords
- **Virtual folder mappings** so different users see different containers or prefixes
- **Bandwidth throttling and quotas** per user
- **Audit logging** of every connection and file operation
- **A web UI** for managing users via `kubectl port-forward`

### Scaling Up: Multiple Replicas

A single SFTPGo pod works fine until it doesn't. Maybe you need high availability. Maybe you have enough concurrent connections that one pod is sweating. Maybe you just don't want a single point of failure sitting between your vendors and their file drops.

The good news: because we set this up with PostgreSQL for state and Secrets for host keys, the Deployment is already stateless. Scaling is just:

```yaml
spec:
  replicas: 3
```

That's it. No migration, no rearchitecture. Every replica mounts the same host keys from the `sftpgo-hostkeys` Secret, so clients get a consistent SSH fingerprint regardless of which pod they land on. User data, virtual folder configs, and quota tracking all live in PostgreSQL, so every replica sees the same state.

SFTPGo also supports a shared event system through the data provider. When one instance updates a user, other instances pick up the change. No manual cache invalidation, no restart required.

#### Load Distribution

SFTP is a TCP protocol, so Kubernetes' default round-robin Service routing handles connection distribution automatically. Each new SFTP connection lands on a different pod. Connections are sticky for their duration (TCP, after all), so a long-running transfer won't bounce between pods mid-stream.

If you're using the nginx-ingress TCP proxy from earlier, the same applies there. Each new inbound connection on port 2222 gets routed to the ClusterIP Service, which distributes across replicas.

One thing to note: SFTP connections are long-lived. CPU and memory might look low even under heavy transfer load because the bottleneck is usually network I/O, not compute. If you want autoscaling, you'll need custom metrics rather than CPU. The monitoring section below covers exactly how to set that up.

### Monitoring with Prometheus and Grafana

Running an SFTP server without monitoring is like deploying to production without logs. You technically *can*, but the first time a vendor calls asking why their upload failed three hours ago, you'll wish you hadn't.

SFTPGo has a built-in Prometheus metrics endpoint. We already enabled it in the config on port `10000`. Hit `/metrics` on that port and you get the standard Prometheus exposition format with everything you'd want to know: active connections, bytes transferred, upload/download counts, error rates, and data provider health.

#### ServiceMonitor

If you're running the [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator) (and if you're on Kubernetes with Prometheus, you probably are), a ServiceMonitor makes scrape configuration automatic:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sftpgo
  namespace: sftpgo
  labels:
    app: sftpgo
spec:
  selector:
    matchLabels:
      app: sftpgo
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

That's the whole thing. The Prometheus Operator picks this up, finds the `sftpgo` Service's `metrics` port, and starts scraping. No need to edit Prometheus config files or restart anything.

If you're running vanilla Prometheus without the Operator, add a scrape job to your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: "sftpgo"
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
            - sftpgo
    relabel_configs:
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        action: keep
        regex: metrics
```

#### What Gets Exposed

SFTPGo's `/metrics` endpoint exports a solid set of counters and gauges. Some of the useful ones:

- `sftpgo_active_connections` - current active SFTP connections per pod
- `sftpgo_active_transfers` - in-progress file transfers
- `sftpgo_bytes_sent_total` / `sftpgo_bytes_received_total` - cumulative transfer volume
- `sftpgo_uploads_total` / `sftpgo_downloads_total` - file operation counts
- `sftpgo_uploads_errors_total` / `sftpgo_downloads_errors_total` - failed transfers
- `sftpgo_data_provider_availability` - whether the PostgreSQL connection is healthy

That last one is important. If the data provider goes down, SFTPGo can't authenticate users. You want to know about that before your vendors do.

#### Grafana Dashboard

SFTPGo ships a pre-built Grafana dashboard that you can import directly. Grab it from the [SFTPGo repo](https://github.com/drakkan/sftpgo) or import dashboard ID `16498` from Grafana's dashboard marketplace.

If you'd rather build your own, here are a few PromQL queries worth pinning:

**Active connections across all replicas:**

```promql
sum(sftpgo_active_connections)
```

**Transfer throughput (bytes/sec, 5-minute rate):**

```promql
sum(rate(sftpgo_bytes_sent_total[5m]) + rate(sftpgo_bytes_received_total[5m]))
```

**Upload error rate as a percentage:**

```promql
sum(rate(sftpgo_uploads_errors_total[5m]))
  / sum(rate(sftpgo_uploads_total[5m])) * 100
```

**Data provider health (alert if any pod loses DB connectivity):**

```promql
min(sftpgo_data_provider_availability) == 0
```

That last query is a good candidate for a Prometheus alert rule. If any replica can't reach PostgreSQL, fire an alert. The `min` catches the case where one pod is healthy but another has lost its connection.

#### Tying It Back to Autoscaling

Remember the HPA note from the replicas section? `sftpgo_active_connections` is your custom metric for scaling. With the [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter), you can scale SFTPGo pods based on actual connection count instead of CPU:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sftpgo
  namespace: sftpgo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sftpgo
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metric:
          name: sftpgo_active_connections
        target:
          type: AverageValue
          averageValue: "50"
```

When the average connection count per pod crosses 50, Kubernetes spins up another replica. When it drops, it scales back down. Way better than guessing based on CPU, which barely moves during SFTP transfers.

### Central Logging

Metrics tell you what's happening right now. Logs tell you what happened at 2am when that vendor's automated upload script decided to authenticate 400 times in a row with the wrong password.

SFTPGo writes structured JSON logs to stdout by default, which is exactly what you want on Kubernetes. No sidecar containers, no log file rotation, no volume mounts for log directories. Your cluster's log collector (Fluentd, Fluent Bit, Promtail, Vector, whatever you're running) picks them up automatically from the container runtime.

A typical SFTPGo log line looks like this:

```json
{
  "level": "info",
  "time": "2026-04-11T14:32:01.123Z",
  "sender": "sftpd",
  "message": "User logged in",
  "username": "vendor-uploads",
  "remote_address": "10.244.3.15:48212",
  "protocol": "SFTP",
  "connection_id": "abc123"
}
```

Every log entry includes the username, remote address, protocol, and connection ID. File operations include the file path and transfer size. Auth failures include the attempted username and the reason. All structured, all parseable, all ready to ship to wherever your logs land.

#### Configuring Log Format

SFTPGo defaults to JSON on stdout, but you can control the verbosity. Add a `logger` block to the ConfigMap if you want to tune it:

```json
{
  "logger": {
    "enabled": true,
    "level": "info",
    "utc_time": true
  }
}
```

Keep it at `info` for production. `debug` is useful when you're troubleshooting auth issues or virtual folder mappings, but it's chatty. It logs every single SSH handshake step, every directory listing, every stat call. On a busy server, that's a lot of log volume.

#### What to Query For

Once your logs are in Loki, Elasticsearch, or whatever your stack uses, these are the queries worth saving:

**Failed authentication attempts** are the first thing you want to see. Brute force attempts, expired credentials, misconfigured clients. In Loki with LogQL:

```logql
{namespace="sftpgo"} | json | level="error" | message=~".*authentication.*"
```

**File transfer activity by user** gives you an audit trail. Who uploaded what, when, and how big:

```logql
{namespace="sftpgo"} | json | message="Upload" | line_format "{{.username}} {{.virtual_path}} {{.elapsed_ms}}ms"
```

**Connection patterns** help you spot anomalies. A user that normally connects once a day suddenly hammering the server every 30 seconds is worth investigating:

```logql
sum by (username) (count_over_time({namespace="sftpgo"} | json | message="User logged in" [1h]))
```

#### Audit Trail

For compliance-heavy environments (and if you're handling file transfers for enterprise partners, you're probably in one), SFTPGo's structured logs give you a complete audit trail out of the box. Every login, every file operation, every disconnect. Combined with the username and remote IP in each log entry, you can reconstruct exactly who did what and when.

Pipe these into a long-retention store (separate from your operational logs) and you've got audit coverage without bolting on a separate audit system. Set a retention policy that matches your compliance requirements and forget about it.

### Things to Watch Out For

**Keep the admin UI internal.** The admin UI has full control over user accounts. Don't Ingress it. Don't LoadBalancer it. `kubectl port-forward` when you need it, close it when you don't. If you absolutely need remote access, put it behind an auth proxy and a VPN. Not just one of those. Both.

**PostgreSQL backups.** You're running a real database now, which means you need real backups. `pg_dump` on a cron, or use your managed PostgreSQL provider's automated backups. The SFTPGo user database isn't big, but losing it means recreating every user and virtual folder mapping from scratch.

**Azure Blob isn't a filesystem.** Listing large directories can be slow because blobs use a flat namespace with prefix-based "directory" simulation. If your users are running `ls` on a container with 500,000 objects, they're going to have a bad time.

### The Math

| | Azure Native SFTP | SFTPGo on K8s |
|---|---|---|
| Monthly endpoint cost | ~$219 | $0 (runs on existing cluster) |
| Requires ADLS Gen2 | Yes | No |
| Multi-user support | Local users only | Full user management |
| Quotas/throttling | No | Yes |
| Audit logging | Azure Monitor | Built-in + syslog |
| Prometheus metrics | No | Native /metrics endpoint |
| Admin UI exposure | N/A | Internal only (port-forward) |
| Setup complexity | Toggle in portal | Deploy to K8s |

The "setup complexity" column is doing some heavy lifting for Azure there. Toggling a switch is simpler. But $219/month simpler? For most workloads, no.

### Wrapping Up

SFTPGo fills a gap that shouldn't exist. SFTP is a 20+ year old protocol. Mapping it to object storage shouldn't cost $2,600/year. Running your own is a few YAML files and a container image. The SFTPGo project is actively maintained, well-documented, and handles edge cases I haven't even thought about yet.

If you're already running Kubernetes, this is maybe 20 minutes of work. The hardest part is explaining to your finance team why you're *not* using the Azure-native option.
