# Deploying MySQL

This page walks through deploying a single-instance MySQL 8.4 server on Lux Kubernetes, backed by persistent storage, and connecting to it from a client. It is suitable for a workflow that needs a database to coordinate tasks or record results. It is not a highly available configuration: there is one replica, no replication, and no automatic failover.

Replace `<namespace>` in the commands and DNS names below with your project's namespace.

## Prerequisites

- A project namespace on Lux Kubernetes.
- `kubectl` configured with a kubeconfig copied from the Rancher dashboard. See "Using kubectl" in the [Lux Kubernetes User Guide](https://docs.olcf.ornl.gov/systems/lux_user_guide.html#lux-kubernetes-user-guide). `kubectl` must be run from `home.ccs.ornl.gov`.
- A `kubectl` client within one minor version of the Lux API server (v1.35 as of November 2026). If `kubectl version` warns about version skew, install a matching client before continuing.

Set your default namespace so the commands below do not each need `-n`:

```bash
kubectl config set-context --current --namespace=<namespace>
```

> Note: Kubeconfig credentials from Rancher expire every 24 hours, and downloading a fresh kubeconfig resets your default namespace. Re-run the `set-context` command above each time you refresh your credentials.

## Deploying MySQL

MySQL is deployed as a StatefulSet rather than a Deployment. A StatefulSet gives the pod a stable name (`mysql-0`) and a PersistentVolumeClaim that is bound to that pod and survives restarts.

### 1. Create the Secret

Set the user password in your shell without echoing it to the screen or saving it in your shell history, then create the Secret. If the variable is empty, the Secret is created with an empty password and no error.

```bash
read -s -p "MySQL user password: " MYSQL_USER_PW; echo
echo ${#MYSQL_USER_PW}   # Should print a nonzero length.

kubectl create secret generic mysql-secret \
  --from-literal=MYSQL_DATABASE=pocdb \
  --from-literal=MYSQL_USER=pocuser \
  --from-literal=MYSQL_PASSWORD="$MYSQL_USER_PW"
```

`kubectl create secret` does not overwrite an existing Secret. To change it, delete it first with `kubectl delete secret mysql-secret`.

No root password is set, and this is deliberate. Without `MYSQL_ROOT_PASSWORD`, the image creates no root account that can be reached over the network. Root exists only as `root@localhost`, without a password, for administration through `kubectl exec`. Anyone who can `exec` into the pod can already read the credentials from its environment, so this does not widen access.

<!-- TODO(verify): confirm with the Lux team whether Secrets are encrypted at rest, then state it here. -->

> Note: Kubernetes Secrets are encoded, not encrypted, when read through the API. Any user with read access to the namespace, typically every member of the Rancher project, can read Secret values (see [Connecting to MySQL](#connecting-to-mysql)).

### 2. Create the ConfigMap

Non-secret server settings go in a ConfigMap, so they can be changed without touching credentials:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  MYSQL_MAX_CONNECTIONS: "100"
```

```bash
kubectl apply -f mysql-configmap.yaml
```

The image silently ignores environment variables it does not recognize, so check that each setting you add takes effect (see [Verify](#5-verify)). The character set and collation are left at the MySQL 8.4 defaults, `utf8mb4` and `utf8mb4_0900_ai_ci`. For settings the image does not expose as environment variables, see the [sclorg mysql-container documentation](https://github.com/sclorg/mysql-container).

### 3. Create the StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      securityContext:
        fsGroup: 2000
      enableServiceLinks: false
      containers:
        - name: mysql
          # quay.io/sclorg/mysql-84-c9s:latest as of 2026-11-20; see "Updating the image"
          image: quay.io/sclorg/mysql-84-c9s@sha256:38ab8a824330bc0f50dad8a9d961f70b99157684ee53dd2aafa7a3b873c5ffa4
          ports:
            - containerPort: 3306
              name: mysql
          envFrom:
            - secretRef:
                name: mysql-secret
            - configMapRef:
                name: mysql-config
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql/data
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
          livenessProbe:
            exec:
              command: ["sh", "-c", "MYSQL_PWD=\"$MYSQL_PASSWORD\" mysqladmin ping -h 127.0.0.1 -u \"$MYSQL_USER\""]
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
          readinessProbe:
            exec:
              command: ["sh", "-c", "MYSQL_PWD=\"$MYSQL_PASSWORD\" mysql -h 127.0.0.1 -u \"$MYSQL_USER\" -e \"SELECT 1\""]
            initialDelaySeconds: 15
            periodSeconds: 5
            timeoutSeconds: 5
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: netapp-block
        resources:
          requests:
            storage: 2Gi
```

```bash
kubectl apply -f mysql-statefulset.yaml
kubectl get pods -w
```

Several details are worth calling out:

- **Image.** The image is pinned to the tested digest rather than the movable `:latest` tag, so pod restarts never silently change the MySQL version (see [Updating the image](#updating-the-image)). As of November 2026, Lux nodes can pull from quay.io directly. If that changes, mirror the image to Docker Hub and attach an `imagePullSecrets` entry; see "Pull Secret Creation for Kubernetes" in the [Lux Kubernetes User Guide](https://docs.olcf.ornl.gov/systems/lux_user_guide.html#lux-kubernetes-user-guide).
- **`storageClassName: netapp-block`.** Lux offers `netapp-file` (the default) and `netapp-block`. Use `netapp-block` for a database: it provides a block device rather than an NFS share, which is what InnoDB's locking and fsync behavior expects. See the "Storage" section of the [Lux Kubernetes User Guide](https://docs.olcf.ornl.gov/systems/lux_user_guide.html#lux-kubernetes-user-guide).
- **`securityContext.fsGroup: 2000`.** The image runs as a non-root user and needs group write access to its data directory. Lux only allows `fsGroup` values from 800 to 9000, so the value of 0 used in the image's upstream examples is rejected.
- **`enableServiceLinks: false`.** Stops Kubernetes from injecting environment variables for every Service in the namespace. Clients find MySQL through DNS instead.
- **Probes.** Both probes log in as the application user over TCP, with the password passed through `MYSQL_PWD` rather than on the command line. The readiness probe gates the Service, so traffic only reaches the pod once the application account can run a query.

> Note: Deleting a StatefulSet does not delete its PersistentVolumeClaims. A recreated StatefulSet adopts the existing `mysql-data-mysql-0` claim exactly as it is, ignoring any changes to `volumeClaimTemplates` (such as the storage class or size) and keeping the existing data. To start clean, delete the claim as well (see [Teardown](#teardown)).

### 4. Create the Service

A headless Service (`clusterIP: None`) gives the pod a stable DNS name within the cluster: `mysql:3306` from the same namespace, or `mysql.<namespace>.svc.cluster.local:3306` from another namespace.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
      name: mysql
```

```bash
kubectl apply -f mysql-service.yaml
```

### 5. Verify

Confirm that the application account works, that the ConfigMap settings were applied (`max_connections` should be 100, not the image default of 151), and that root is local only (only `root localhost` should be listed):

```bash
kubectl exec mysql-0 -- sh -c 'MYSQL_PWD="$MYSQL_PASSWORD" mysql -u "$MYSQL_USER" pocdb -e "SELECT VERSION();"'
kubectl exec mysql-0 -- mysql -u root -e "SHOW VARIABLES WHERE Variable_name IN ('character_set_server','collation_server','max_connections');"
kubectl exec mysql-0 -- mysql -u root -e "SELECT user, host FROM mysql.user WHERE user='root';"
```

Confirm that storage is persistent by writing a row, deleting the pod, and reading the row back after the StatefulSet recreates the pod:

```bash
kubectl exec -i mysql-0 -- sh -c 'MYSQL_PWD="$MYSQL_PASSWORD" mysql -u "$MYSQL_USER" pocdb' <<'SQL'
CREATE TABLE persistence_test (id INT AUTO_INCREMENT PRIMARY KEY, note VARCHAR(64));
INSERT INTO persistence_test (note) VALUES ('written before pod deletion');
SQL

kubectl delete pod mysql-0
kubectl wait --for=condition=ready pod/mysql-0 --timeout=300s
kubectl exec mysql-0 -- sh -c 'MYSQL_PWD="$MYSQL_PASSWORD" mysql -u "$MYSQL_USER" pocdb -e "SELECT * FROM persistence_test;"'
```

For SQL containing quotes, prefer the heredoc form above: SQL string literals use single quotes, which otherwise end the `sh -c '...'` string early.

## Connecting to MySQL

The `pocuser` password can be read back from the Secret by anyone with read access to the namespace:

```bash
kubectl get secret mysql-secret -o jsonpath='{.data.MYSQL_PASSWORD}' | base64 -d; echo
```

For administrative tasks, connect as root from inside the pod:

```bash
kubectl exec -it mysql-0 -- mysql -u root
```

### From inside the cluster

Clients running as pods in the same namespace connect to `mysql:3306`. Load the credentials from `mysql-secret` (for example, with `envFrom`) rather than copying them into the client's manifest. This is currently the recommended way to reach the database.

If a client pod also needs outbound internet access, use the OLCF proxy environment variables described in the Lux [Kubernetes User Guide](https://docs.olcf.ornl.gov/systems/lux_user_guide.html#lux-kubernetes-user-guide), and add `.svc,.cluster.local,mysql` to `no_proxy` so requests to in-cluster services are not sent to the proxy.

### Port forwarding

For development and testing, forward a local port to the Service, then connect from a second terminal on the same login node:

```bash
kubectl port-forward svc/mysql 3307:3306
mysql -h 127.0.0.1 -P 3307 -u pocuser -p pocdb   # in a second terminal
```

The forward lasts only as long as the command runs, and is limited by the 24-hour lifetime of your kubeconfig credentials. Login nodes are shared, so while the forward is running, other users on the same node can connect to the forwarded port (they still need the password). If the local port is already in use, choose another, and stop the forward when you are finished.

### Long-running external access

<!-- TODO(verify): replace this section once the Lux team confirms the supported mechanism for exposing non-HTTP TCP services. -->

HTTPRoute, the exposure mechanism documented for Lux, routes HTTP traffic only and cannot carry the MySQL protocol. A supported way to expose non-HTTP services outside the cluster is not currently documented, so run clients as pods in the same namespace. If your workflow requires external access to the database, contact OLCF support at help@olcf.ornl.gov.

## Updating the image

The image is pinned to a digest, so it changes only when someone updates it deliberately. Refresh it periodically to pick up security fixes:

1. Back up the database. A newer MySQL version may upgrade the data directory on startup, and that upgrade cannot be undone by returning to the older image:

   ```bash
   kubectl exec mysql-0 -- sh -c 'MYSQL_PWD="$MYSQL_PASSWORD" mysqldump --no-tablespaces -u "$MYSQL_USER" pocdb' > pocdb-backup.sql
   ```

2. Deploy the new tag (for example, `quay.io/sclorg/mysql-84-c9s:latest`) to a test deployment, run the checks in [Verify](#5-verify), and read back the digest it resolved to:

   ```bash
   kubectl get pod mysql-0 -o jsonpath='{.status.containerStatuses[0].imageID}{"\n"}'
   ```

3. Update the `image:` line and its date comment, and apply the StatefulSet. The image can be changed in place; the pod is replaced and the PersistentVolumeClaim is kept.

## Teardown

```bash
kubectl delete statefulset mysql
kubectl delete service mysql
kubectl delete configmap mysql-config
kubectl delete secret mysql-secret
```

Deleting a StatefulSet does not delete the PersistentVolumeClaims it created. Remove the data explicitly once you are sure you no longer need it, or whenever you redeploy with a fresh data directory or changes to `volumeClaimTemplates`:

```bash
kubectl get pvc
kubectl delete pvc mysql-data-mysql-0
```

## Troubleshooting

**Kyverno denies the StatefulSet, mentioning `fsGroup`.** The `fsGroup` value is outside Lux's allowed range of 800 to 9000.

**Pod stuck in `Pending` with `pod has unbound immediate PersistentVolumeClaims`.** Run `kubectl describe pvc mysql-data-mysql-0`. If the claim names a storage class other than `netapp-block`, it is left over from an earlier StatefulSet. Run `kubectl delete pvc mysql-data-mysql-0 --wait=false`, then `kubectl delete pod mysql-0`, and the StatefulSet recreates both from the current template.

**Pod restarting during its first start (`CrashLoopBackOff`).** If initializing the data directory takes longer than the liveness probe allows, the probe kills MySQL before it finishes. Increase the liveness probe's `initialDelaySeconds`, or add a `startupProbe`.

**`Access denied` when connecting as root over the network.** This is expected; root is local only. Use `kubectl exec -it mysql-0 -- mysql -u root` instead.

## Limitations

- Single instance, no replication, and no automatic failover. A node failure or eviction means downtime for as long as the pod takes to reschedule and recover.
- No supported long-running endpoint outside the cluster; see [Long-running external access](#long-running-external-access).
- TLS uses the image's self-signed default certificate; no certificates are managed or enforced.
- The pinned image does not receive security updates automatically; see [Updating the image](#updating-the-image).
- Automated backups are not configured. The `mysqldump` command under [Updating the image](#updating-the-image) can be used for manual backups.
