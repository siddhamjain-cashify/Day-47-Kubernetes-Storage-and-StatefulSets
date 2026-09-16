
# Day 47 — Kubernetes Storage and StatefulSets

## Goal

Understand how Kubernetes manages persistent storage and stateful applications.

By the end of Day 47, I practiced:

- PersistentVolumes (PV) and PersistentVolumeClaims (PVC)
- StorageClasses and dynamic provisioning
- Storage access modes and reclaim policies
- StatefulSets and stable pod identity
- Headless Services and StatefulSet DNS
- PostgreSQL persistence across pod and StatefulSet recreation
- Scaling StatefulSets and per-replica storage
- Testing connectivity to PostgreSQL from another pod
- Comparing local kind storage with AWS storage options

---

## 1. Environment

| Item | Value |
|---|---|
| Kubernetes context | `kind-day41` |
| Cluster | kind |
| StorageClass | `standard` |
| Provisioner | `rancher.io/local-path` |
| Reclaim policy | `Delete` |
| Volume binding mode | `WaitForFirstConsumer` |
| Volume expansion | Disabled |
| PostgreSQL image | `postgres:16` |
| PostgreSQL port | `5432` |
| Namespace | `default` |

The `standard` StorageClass uses local-path storage in the kind cluster. This is not AWS EBS storage.

---

## 2. Kubernetes Storage Concepts

| Concept | Explanation |
|---|---|
| `emptyDir` | Temporary storage created for a pod. It is removed when the pod is removed. |
| `hostPath` | Mounts a path from the node's filesystem. It is tied to the node and can be risky for portable workloads. |
| PV | A PersistentVolume represents storage available to the cluster. |
| PVC | A PersistentVolumeClaim is a workload's request for storage. |
| StorageClass | Defines how storage is dynamically provisioned. |
| `ReadWriteOnce` (RWO) | Read-write mounting by one node at a time. Multiple pods on that node may still use it, subject to driver and filesystem behavior. |
| `ReadOnlyMany` (ROX) | Read-only mounting by multiple nodes, if supported by the storage driver. |
| `ReadWriteMany` (RWX) | Read-write mounting by multiple nodes, if supported by the storage driver. |
| Reclaim policy | Defines what happens to a PV after its claim is released. |
| `WaitForFirstConsumer` | Delays volume binding/provisioning until a pod's scheduling requirements are known. |
| CSI | Container Storage Interface; allows storage drivers to integrate with Kubernetes. |

### Reclaim policies

- `Retain`: the PV/storage is retained for manual handling after the PVC is released.
- `Delete`: the PV and dynamically provisioned storage are deleted when the PVC is released, if supported by the provisioner.

### StatefulSet concepts

A StatefulSet provides:

- Stable pod names, such as `postgres-0`.
- Ordered pod creation and termination.
- Stable association between each pod ordinal and its PVC.
- Predictable DNS identity through a headless Service.

A StatefulSet does **not** automatically replicate application data between replicas. Three PostgreSQL StatefulSet pods are three separate database instances unless replication is configured separately.

---

## 3. Inspect the StorageClass

Commands:

```bash
kubectl get storageclass
kubectl get storageclass standard -o yaml
```

Observed configuration:

```yaml
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

The StorageClass did not have volume expansion enabled.

---

## 4. Basic PVC Persistence Test

### Create the PVC

The `storage-demo` PVC requested 1Gi of RWO storage using the `standard` StorageClass.

```bash
kubectl apply -f pvc-demo.yaml
```

The PVC initially remained Pending because the StorageClass uses `WaitForFirstConsumer`. A pod consuming the claim was needed before provisioning could complete.

### Create a pod that mounts the PVC

The pod mounted the claim at `/data`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-demo-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 86400"]
      volumeMounts:
        - name: storage
          mountPath: /data
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: storage-demo
```

Apply and inspect:

```bash
kubectl apply -f storage-demo-pod.yaml
kubectl get pod storage-demo-pod
kubectl get pvc storage-demo
kubectl get pv
```

The PVC became Bound and Kubernetes provisioned a 1Gi PV.

### Write a file

```bash
kubectl exec storage-demo-pod -- \
  sh -c 'echo "Day 47 persistent data survives pod deletion" > /data/test.txt'

kubectl exec storage-demo-pod -- cat /data/test.txt
```

### Delete and recreate the pod

```bash
kubectl delete pod storage-demo-pod
kubectl apply -f storage-demo-pod.yaml
```

After the recreated pod was Running, the file was still present:

```bash
kubectl exec storage-demo-pod -- cat /data/test.txt
```

**Result:** The data survived pod deletion because it was stored on the PVC, not only in the container's writable layer.

### Delete the PVC

```bash
kubectl delete pod storage-demo-pod
kubectl delete pvc storage-demo
kubectl get pvc
kubectl get pv
```

Both PVC and PV listings showed no resources.

**Result:** The `Delete` reclaim policy removed the dynamically provisioned PV/storage after the claim was deleted.

---

## 5. PostgreSQL StatefulSet

### Create the Secret

A lab-only password was stored in a Kubernetes Secret:

```bash
kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_PASSWORD='day47-lab-password'

kubectl get secret postgres-secret
```

The Secret was referenced by the StatefulSet through an environment variable.

> This password is for the local lab only, not production.

### Create a headless Service

File: `postgres-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

Apply and verify:

```bash
kubectl apply -f postgres-service.yaml
kubectl get service postgres
```

The Service showed `CLUSTER-IP: None`, confirming that it is headless.

### Create the StatefulSet

File: `postgres-statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - name: postgres
              containerPort: 5432
          env:
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_DB
              value: postgres
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: standard
        resources:
          requests:
            storage: 1Gi
```

Apply and inspect:

```bash
kubectl apply -f postgres-statefulset.yaml

kubectl get statefulset postgres
kubectl get pods -l app=postgres
kubectl get pvc
```

The StatefulSet created:

- Pod: `postgres-0`
- PVC: `data-postgres-0`
- Requested storage: 1Gi
- PostgreSQL port: 5432

Wait for readiness:

```bash
kubectl wait --for=condition=Ready pod/postgres-0 --timeout=180s
```

Inspect logs:

```bash
kubectl logs postgres-0
```

PostgreSQL logs confirmed that the database system was ready to accept connections.

---

## 6. Create and Verify Database Data

Create a table and insert three rows:

```bash
kubectl exec -it postgres-0 -- \
  psql -U postgres -d postgres \
  -c "CREATE TABLE day47_test (id INT PRIMARY KEY, note TEXT); INSERT INTO day47_test VALUES (1, 'first row'), (2, 'second row'), (3, 'third row');"
```

Verify:

```bash
kubectl exec -it postgres-0 -- \
  psql -U postgres -d postgres \
  -c "SELECT * FROM day47_test ORDER BY id;"
```

Observed data:

| id | note |
|---:|---|
| 1 | first row |
| 2 | second row |
| 3 | third row |

---

## 7. Test Data Survives Pod Deletion

Delete the PostgreSQL pod:

```bash
kubectl delete pod postgres-0
```

Wait for the StatefulSet to recreate it:

```bash
kubectl wait --for=condition=Ready pod/postgres-0 --timeout=180s
```

Check the pod and PVC:

```bash
kubectl get pod postgres-0
kubectl get pvc data-postgres-0
```

Query the table again:

```bash
kubectl exec -it postgres-0 -- \
  psql -U postgres -d postgres \
  -c "SELECT * FROM day47_test ORDER BY id;"
```

**Observed result:** The pod was recreated, the PVC remained Bound, and all three rows were still present.

**Conclusion:** Deleting a pod does not delete the PVC or the database data stored on it.

---

## 8. Test StatefulSet Deletion and Recreation

Delete the StatefulSet:

```bash
kubectl delete statefulset postgres
```

Check the pods and PVC:

```bash
kubectl get pods -l app=postgres
kubectl get pvc data-postgres-0
```

The pod disappeared, but the PVC remained Bound.

Recreate the StatefulSet:

```bash
kubectl apply -f postgres-statefulset.yaml
kubectl wait --for=condition=Ready pod/postgres-0 --timeout=180s
```

Verify the data:

```bash
kubectl exec -it postgres-0 -- \
  psql -U postgres -d postgres \
  -c "SELECT * FROM day47_test ORDER BY id;"
```

**Observed result:** All three rows remained after StatefulSet deletion and recreation.

The StatefulSet's PVC retention policy was:

```json
{
  "whenDeleted": "Retain",
  "whenScaled": "Retain"
}
```

**Conclusion:** Deleting the StatefulSet does not automatically delete its PVCs under this retention policy.

---

## 9. Scale StatefulSet to Three Replicas

Scale up:

```bash
kubectl scale statefulset postgres --replicas=3
kubectl get pods -l app=postgres -w
```

Verify:

```bash
kubectl get statefulset postgres
kubectl get pods -l app=postgres -o wide
kubectl get pvc
```

Observed:

| Pod | PVC |
|---|---|
| `postgres-0` | `data-postgres-0` |
| `postgres-1` | `data-postgres-1` |
| `postgres-2` | `data-postgres-2` |

All three pods became Running and Ready. Each had a separate 1Gi Bound PVC.

**Important:** These are independent PostgreSQL instances. StatefulSet scaling alone does not configure PostgreSQL replication or synchronize tables between pods.

---

## 10. StatefulSet DNS Test

The headless Service provides DNS discovery for the pods.

### Resolve the Service

```bash
kubectl exec debug -- nslookup postgres
```

The Service resolved to all three PostgreSQL pod IPs.

### Resolve individual StatefulSet pods

```bash
kubectl exec debug -- nslookup postgres-0.postgres
kubectl exec debug -- nslookup postgres-1.postgres
kubectl exec debug -- nslookup postgres-2.postgres
```

Observed mapping:

| DNS name | Pod IP |
|---|---|
| `postgres-0.postgres` | `10.244.1.67` |
| `postgres-1.postgres` | `10.244.2.92` |
| `postgres-2.postgres` | `10.244.1.69` |

The fully qualified DNS pattern is:

```text
<pod-name>.<headless-service>.<namespace>.svc.cluster.local
```

Example:

```text
postgres-0.postgres.default.svc.cluster.local
```

The `nslookup` output included “recursion not available,” but the DNS answers were returned successfully.

---

## 11. Scale Back to One Replica

Scale down:

```bash
kubectl scale statefulset postgres --replicas=1
```

Verify:

```bash
kubectl get statefulset postgres
kubectl get pods -l app=postgres
kubectl get pvc
```

Observed:

- Only `postgres-0` remained running.
- StatefulSet was `1/1` Ready.
- All three PVCs remained Bound.

**Conclusion:** Scaling down removes higher-ordinal pods, but the PVCs are retained and can be reused when scaling back up.

---

## 12. Delete PVCs and Verify Data Loss

Before deleting the PVCs, the retention policy was checked:

```bash
kubectl get statefulset postgres \
  -o jsonpath='{.spec.persistentVolumeClaimRetentionPolicy}{"\n"}'
```

Output:

```json
{"whenDeleted":"Retain","whenScaled":"Retain"}
```

Delete the StatefulSet:

```bash
kubectl delete statefulset postgres
kubectl get pods -l app=postgres
```

After confirming the pods were gone, delete the PVCs:

```bash
kubectl delete pvc data-postgres-0 data-postgres-1 data-postgres-2
```

Verify:

```bash
kubectl get pvc
kubectl get pv
```

Both showed no resources.

Recreate PostgreSQL:

```bash
kubectl apply -f postgres-statefulset.yaml
kubectl wait --for=condition=Ready pod/postgres-0 --timeout=180s
kubectl get pvc
```

Query the old table:

```bash
kubectl exec postgres-0 -- \
  psql -U postgres -d postgres \
  -c "SELECT * FROM day47_test;"
```

Observed:

```text
ERROR: relation "day47_test" does not exist
```

**Conclusion:** Deleting the PVCs removed the old database storage. The newly provisioned PVC contained a fresh PostgreSQL database.

---

## 13. PostgreSQL Service Connectivity

### Test TCP connectivity from the debug pod

```bash
kubectl exec debug -- nc -vz postgres 5432
```

Observed:

```text
Connection to postgres (...) 5432 port [tcp/postgresql] succeeded!
```

This confirmed that the debug pod could resolve the Service and establish a TCP connection to PostgreSQL.

### Why curl returned an empty reply

```bash
kubectl exec debug -- \
  sh -c 'curl -v --connect-timeout 3 postgres:5432'
```

The TCP connection was established, but curl sent an HTTP request. PostgreSQL does not speak HTTP; it uses its own database protocol.

Therefore, the empty HTTP reply was expected and did not mean the TCP connection failed.

### Connect using PostgreSQL client

The first attempt without a password failed because TCP authentication required a password.

The working command used the password environment variable inside the PostgreSQL container:

```bash
kubectl exec postgres-0 -- sh -c \
  'export PGPASSWORD="$POSTGRES_PASSWORD"; psql -h postgres -U postgres -d postgres -c "SELECT current_database(), current_user, inet_server_addr(), inet_server_port();"'
```

Observed:

| Field | Value |
|---|---|
| Database | `postgres` |
| User | `postgres` |
| Server address | `10.244.1.73` |
| Server port | `5432` |

**Conclusion:** PostgreSQL was reachable through the headless Service using its DNS name.

---

## 14. AWS Storage Comparison

| kind lab | Typical AWS EKS setup |
|---|---|
| `standard` StorageClass uses `rancher.io/local-path` | EBS CSI commonly provisions EBS volumes |
| Storage is local to a kind node | EBS is network block storage in an AWS Availability Zone |
| Expansion is disabled in this StorageClass | Expansion depends on the CSI driver, StorageClass, and filesystem support |
| RWO behavior is provided by the local storage driver | EBS volumes are generally attached read-write to one node at a time |
| PVC requests storage | PVC can trigger dynamic provisioning through a StorageClass |

EBS volumes are Availability-Zone-specific, so pod scheduling must be compatible with the volume's AZ.

For shared read-write storage across nodes, AWS EFS with a suitable CSI setup is a different option.

---

## 15. Stretch Goals Status

| Stretch goal | Status |
|---|---|
| Volume expansion | Not performed. The `standard` StorageClass has expansion disabled. |
| RWO multi-node mounting | Not performed. Behavior depends on the storage driver and scheduling. |

These were stretch goals; the core storage and StatefulSet exercises were completed.

---

## 16. Key Lessons

1. A container's writable layer is not a substitute for persistent storage.
2. A PVC can outlive the pod that mounts it.
3. A StatefulSet provides stable pod identities and per-replica storage claims.
4. StatefulSet replicas do not automatically share or replicate application data.
5. A headless Service provides DNS discovery without a virtual ClusterIP.
6. Scaling down normally retains StatefulSet PVCs.
7. Deleting a PVC can permanently remove data when the provisioner uses a `Delete` reclaim policy.
8. TCP connectivity does not mean an application protocol such as HTTP is supported.
9. Local kind storage and AWS EBS have different behavior and failure boundaries.

---

## 17. Journal Notes

I created a basic PVC and mounted it into a pod. I wrote a file to the mounted storage, deleted the pod, recreated it, and confirmed that the file remained. Deleting the PVC removed the provisioned PV/storage.

I deployed PostgreSQL as a StatefulSet using a headless Service, a Secret, and `volumeClaimTemplates`. I created a table and verified that its rows survived pod deletion and StatefulSet deletion/recreation because the PVC was retained.

I scaled PostgreSQL to three replicas and confirmed that each pod had its own PVC and stable DNS name. The replicas were independent PostgreSQL instances, not a replicated database cluster. After scaling back to one, the three PVCs remained.

I then deleted the StatefulSet and all three PVCs. Recreating PostgreSQL created a fresh database, and the previous table no longer existed. Finally, I verified TCP connectivity from the debug pod to PostgreSQL on port 5432 and connected to PostgreSQL through its Service DNS name.

My kind cluster uses local-path storage, not AWS EBS. The current StorageClass does not support volume expansion, so the expansion stretch goal was not performed.

---

## 18. Review Questions

1. What is the difference between a PV and a PVC?
2. Why did the PVC remain Pending before a pod consumed it?
3. What does `WaitForFirstConsumer` do?
4. Why did the test file survive pod deletion?
5. What is the difference between `Retain` and `Delete` reclaim policies?
6. Why does a StatefulSet use stable pod names?
7. What does `volumeClaimTemplates` create?
8. Does scaling a PostgreSQL StatefulSet to three replicas replicate the database data?
9. What is the purpose of a headless Service?
10. What is the DNS pattern for an individual StatefulSet pod?
11. Why did the PVCs remain after scaling down?
12. Why did the old database table disappear after PVC deletion?
13. Why did `curl postgres:5432` fail even though TCP connectivity succeeded?
14. Why is local-path storage not equivalent to AWS EBS?
15. What additional configuration is required to make PostgreSQL replicas replicate data?

---

## Final Status

**Day 47 core hands-on exercises: completed.**

Storage persistence, StatefulSet identity, PVC retention, scaling, DNS discovery, intentional data loss, and PostgreSQL network connectivity were all tested.
