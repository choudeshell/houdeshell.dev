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

Two secrets. One for Azure Blob credentials, one for PostgreSQL:

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

#### The ConfigMap

SFTPGo has a mountain of configuration options. The data provider config is handled by the env vars in the secret above, so the ConfigMap just needs the SFTP and HTTP bindings:

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
      }
    }
```

Port `2022` for SFTP, port `8080` for the web admin. PostgreSQL connection details come from the `sftpgo-db` secret, so they stay out of the ConfigMap where they belong.

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
          envFrom:
            - secretRef:
                name: sftpgo-azure
            - secretRef:
                name: sftpgo-db
          volumeMounts:
            - name: config
              mountPath: /etc/sftpgo/sftpgo.json
              subPath: sftpgo.json
            - name: data
              mountPath: /var/lib/sftpgo
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
        - name: data
          persistentVolumeClaim:
            claimName: sftpgo-data
```

Both secrets are mounted via `envFrom`. The PVC for `/var/lib/sftpgo` persists host keys across pod restarts. If the host keys regenerate, every client that's connected before will get an angry `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` and your users will think they're being MITM'd.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sftpgo-data
  namespace: sftpgo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

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

### Things to Watch Out For

**Host keys.** Persist them. I said it already but I'll say it again because I've seen this bite people in production.

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
| Admin UI exposure | N/A | Internal only (port-forward) |
| Setup complexity | Toggle in portal | Deploy to K8s |

The "setup complexity" column is doing some heavy lifting for Azure there. Toggling a switch is simpler. But $219/month simpler? For most workloads, no.

### Wrapping Up

SFTPGo fills a gap that shouldn't exist. SFTP is a 20+ year old protocol. Mapping it to object storage shouldn't cost $2,600/year. Running your own is a few YAML files and a container image. The SFTPGo project is actively maintained, well-documented, and handles edge cases I haven't even thought about yet.

If you're already running Kubernetes, this is maybe 20 minutes of work. The hardest part is explaining to your finance team why you're *not* using the Azure-native option.
