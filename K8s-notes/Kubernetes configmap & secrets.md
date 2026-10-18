Kubernetes **ConfigMaps** store non-sensitive application configuration, while **Secrets** store sensitive values such as passwords, API keys, and TLS certificates. Both keep configuration separate from container images, so the same image can run across dev, staging, and production with different settings. [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)

## Why they exist

Without ConfigMaps or Secrets, you might bake values like a database URL or password directly into a Docker image. That creates problems: changing one environment means rebuilding the image, and credentials may leak into image layers or Git history.

Kubernetes solves this by separating:

- **Application image** — code plus dependencies; usually identical across environments.
- **Configuration** — environment-specific values such as URLs, feature flags, and log levels.
- **Credentials** — passwords, tokens, private keys, certificates.

```text
                 Kubernetes Pod
        ┌─────────────────────────────────────┐
        │  Container: my-app:1.4.0            │
        │                                     │
        │  Reads:                             │
        │   • Environment variables           │
        │   • Files mounted in the container  │
        └───────────┬─────────────┬───────────┘
                    │             │
        ┌───────────▼───┐   ┌─────▼───────────┐
        │   ConfigMap   │   │     Secret      │
        │---------------│   │-----------------│
        │ DB_HOST       │   │ DB_PASSWORD     │
        │ LOG_LEVEL     │   │ API_TOKEN       │
        │ app.properties│   │ tls.crt/tls.key │
        └───────────────┘   └─────────────────┘
```

A ConfigMap is a key-value object for non-confidential data; a Secret is the equivalent object intended for confidential data. [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)

## ConfigMap

A ConfigMap stores plain configuration data as key-value pairs. Pods can consume it as environment variables, command-line arguments, or files mounted through a volume. [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)

### Example: application settings

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "INFO"
  application.properties: |
    server.port=8080
    feature.payment-enabled=true
```

Here:

- `DATABASE_HOST`, `DATABASE_PORT`, and `LOG_LEVEL` are simple settings.
- `application.properties` is a multi-line file stored inside the ConfigMap.
- ConfigMaps use `data` for UTF-8 text and `binaryData` for base64-encoded binary data. [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)

### Consuming a ConfigMap as environment variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: my-app:1.4.0
      envFrom:
        - configMapRef:
            name: app-config
```

`envFrom` injects **all keys** from the ConfigMap as environment variables. For example, the container receives `DATABASE_HOST`, `DATABASE_PORT`, and `LOG_LEVEL`. [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)

To inject only selected keys:

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: DATABASE_HOST
```

This maps the ConfigMap key `DATABASE_HOST` to the container environment variable `DB_HOST`. [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)

### Consuming a ConfigMap as files

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.4.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

Kubernetes creates `/etc/app/application.properties` inside the container. This is useful for applications that expect configuration files rather than environment variables, such as Nginx, Spring Boot, or Prometheus. [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)

### Important ConfigMap behavior

| Behavior | Details |
|---|---|
| Data limit | A ConfigMap can hold up to 1 MiB.  [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/) |
| Namespace scope | The Pod and ConfigMap must be in the same namespace.  [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/) |
| Mounted files | Updates are eventually reflected in mounted volumes.  [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/) |
| Environment variables | Changes do **not** update a running Pod; restart the Pod to pick up new values.  [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/) |
| Security | ConfigMaps provide no secrecy or encryption; never store passwords there.  [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/) |
| Immutable option | Setting `immutable: true` prevents accidental updates and reduces API-server watch load.  [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/) |

## Secret

A Secret stores small amounts of sensitive data, such as database passwords, OAuth tokens, SSH private keys, and TLS certificates. Like ConfigMaps, Secrets can be exposed to Pods as environment variables or mounted as files. [kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)

### Example: database credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USER: "appuser"
  DB_PASSWORD: "S3cureP@ssw0rd"
```

`stringData` accepts plain-text values; Kubernetes converts them internally to base64-encoded values under `data`. [kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)

You can also create it imperatively:

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=appuser \
  --from-literal=DB_PASSWORD='S3cureP@ssw0rd'
```

### Consuming a Secret as environment variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: my-app:1.4.0
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
```

The application reads `DB_USER` and `DB_PASSWORD` from its environment, without those values being present in the image or Pod YAML. [kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)

### Consuming a Secret as a mounted file

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.4.0
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: db-secret
```

Each Secret key becomes a file under `/etc/secrets`, for example `/etc/secrets/DB_PASSWORD`. Mounted Secret updates are eventually propagated to the Pod, while a `subPath` mount does not receive automatic updates. [kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)

## ConfigMap vs Secret

| Aspect | ConfigMap | Secret |
|---|---|---|
| Intended data | Non-sensitive configuration | Sensitive data |
| Examples | Service URLs, feature flags, log level, config files | Passwords, API tokens, SSH keys, TLS certificates |
| Storage format | Plain UTF-8 text or base64 binary data | Base64-encoded data; `stringData` accepts plain text |
| Encryption | None | Base64 is **not encryption**; enable encryption at rest for real protection |
| Access control | RBAC applies | RBAC applies; use stricter, least-privilege permissions |
| Size limit | 1 MiB | 1 MiB |
| Pod consumption | Env vars, args, mounted files, Kubernetes API | Env vars, mounted files, image-pull credentials |
| Update behavior | Mounted volumes update eventually; env vars require restart | Mounted volumes update eventually; env vars require restart |

ConfigMaps and Secrets both have a 1 MiB size limit, and both are consumed by Pods in similar ways. The major difference is purpose: ConfigMaps are for ordinary settings, while Secrets are for confidential values. [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)

## Secret types

Kubernetes provides built-in Secret types for common use cases. [kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)

| Type | Typical use |
|---|---|
| `Opaque` | General user-defined credentials; the default type |
| `kubernetes.io/tls` | TLS certificate and private key, often for Ingress |
| `kubernetes.io/dockerconfigjson` | Private container-registry credentials |
| `kubernetes.io/basic-auth` | Username and password for basic authentication |
| `kubernetes.io/ssh-auth` | SSH private key |
| `kubernetes.io/service-account-token` | Legacy long-lived ServiceAccount token |

For TLS, Kubernetes expects the keys `tls.crt` and `tls.key`; for Docker registry credentials, it expects `.dockerconfigjson`. [kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)

Example TLS Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-tls
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    ...
    -----END PRIVATE KEY-----
```

## Critical interview point: Secrets are not automatically encrypted

A common misconception is that Kubernetes Secrets are encrypted because their values appear base64-encoded. **Base64 is encoding, not encryption**—anyone who can read the Secret can decode it.

By default, Secrets are stored unencrypted in etcd, the API server’s backing store. Kubernetes recommends enabling encryption at rest, applying least-privilege RBAC, restricting Secret access to specific containers, and considering external secret stores. [kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)

### Practical security controls

- Enable **encryption at rest** for Secret objects in etcd.
- Use **RBAC** so application service accounts can read only the Secrets they need.
- Avoid granting broad `get`, `list`, or `watch` permissions on Secrets; `list` or `watch` can expose all Secrets in a namespace. [kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)
- Do not commit Secret manifests containing real credentials to Git.
- Prefer external secret managers such as HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, or Google Secret Manager, often integrated through the External Secrets Operator or Secrets Store CSI Driver.
- Use short-lived credentials where possible instead of long-lived static passwords.

## Common interview questions

**Q: Can a Pod use a ConfigMap or Secret from another namespace?**  
No, not through the normal Pod `env` or volume references; the referenced object must be in the same namespace. An application can access another namespace’s ConfigMap only by calling the Kubernetes API with sufficient permissions. [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)

**Q: What happens if a referenced Secret does not exist?**  
By default, the Pod cannot start until the required Secret is available. You can mark a Secret reference as `optional: true`, in which case Kubernetes ignores a missing Secret. [kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)

**Q: If I update a ConfigMap, will my application see the change immediately?**  
If it is mounted as a volume, the files are eventually updated. If it is injected as an environment variable, the running container keeps the old value until the Pod is restarted. [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)

**Q: When should you use ConfigMap vs Secret?**  
Use ConfigMap for values that are safe to expose, such as hostnames and feature flags. Use Secret for anything whose disclosure would create a security risk, such as credentials, tokens, private keys, or certificates. [kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/)