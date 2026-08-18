# DSO202 Practical 02 Report

## Implementing Persistent Storage for a Stateful Application in Kubernetes

## 1. Objective

Practical 1 only used objects that could be thrown away. This practical is about
workloads whose value is their data. The aim was to learn where Kubernetes
actually writes bytes (PersistentVolumes, PersistentVolumeClaims and
StorageClasses), what happens to those bytes when a Pod, a claim or a whole
cluster is deleted, and why databases are run as StatefulSets rather than
Deployments. It finishes with PostgreSQL 18, the database Assignment 1 uses,
keeping its rows after its Pod was destroyed.

Descriptor sections covered: Unit I 1.2.5 (Secrets), 1.4.1–1.4.3 (Volumes,
PV/PVC, StorageClasses), 1.5.3 (ResourceQuota/LimitRange); Unit II 2.1.1–2.1.4
(StatefulSets, stable network identity, headless Services, persistent storage
for stateful workloads). Learning outcome LO4 primarily, with LO1–LO3
reinforced.

## 2. Environment

| Item                       | Value                                                      |
| -------------------------- | ---------------------------------------------------------- |
| Operating system           | macOS 26.7.1, Apple Silicon (arm64)                         |
| Docker                     | Docker Desktop, Engine 29.9.2                               |
| kind                       | v0.32.0 (go1.26.3 darwin/arm64)                             |
| kubectl                    | v1.37.2 (one minor version ahead of the cluster - supported)|
| Cluster Kubernetes version | v1.36.1 (`kindest/node:v1.36.1`, pinned by digest)          |
| Storage provisioner        | `rancher.io/local-path`, installed by kind                  |
| PostgreSQL image           | `postgres:18-alpine` (server reported PostgreSQL 18.6)      |
| Other images               | `busybox:1.37`, `nginx:1.30-alpine`, `nginx:1.31-alpine`    |

Listing 1 (with the node-name patches) worked first time, so the fallback
Listing 1B was not needed. Full output: [stage0](../evidence/stage0-prerequisites.txt).

## 3. Procedure and Observations

> **Screenshots.** The whole practical, including the extensions, was run a second time on
> 10 November 2026 and every command was captured in the terminal. Those screenshots are
> under each stage, in [`screenshots/`](../screenshots/), and the matching
> plain-text output is in [`evidence/logs/`](../evidence/logs/). The inline output blocks and
> the `.txt` evidence linked in the text come from the first run on 2 November, so Pod name
> suffixes, IP addresses, volume UIDs and timestamps differ between the two runs; the
> behaviour is the same.

### 3.0 Stage 0 – Prerequisites

The three tools reported their versions, the stopped Practical 1 cluster
`dso202` was deleted, and `/tmp/dso202-p2-storage` was created *before* the
cluster so that kind could mount it.

```
$ kind delete cluster --name dso202
Deleted nodes: ["dso202-control-plane"]
$ kind get clusters
No kind clusters found.
```

The output shows a clean starting point with no second cluster competing for
memory. ([evidence](../evidence/stage0-prerequisites.txt))

#### Screenshots

**Step 1 – Docker, kind and kubectl versions**

![Step 1 – Docker, kind and kubectl versions](<../screenshots/s0-step1-versions.png>)

**Step 2 – the Practical 1 cluster `dso202` deleted**

![Step 2 – the Practical 1 cluster `dso202` deleted](<../screenshots/s0-step2-delete-p1-cluster.png>)

**Step 3 – host directory created before the cluster**

![Step 3 – host directory created before the cluster](<../screenshots/s0-step3-host-dir.png>)

**Step 4 – free disk on the host and in Docker**

![Step 4 – free disk on the host and in Docker](<../screenshots/s0-step4-disk.png>)

### 3.1 Stage 1 – Cluster, namespace, and the storage landscape

```
$ kubectl get storageclass
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
dso202-retain        rancher.io/local-path   Retain          WaitForFirstConsumer   false
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false

$ kubectl -n local-path-storage get pods
local-path-provisioner-855c7b7774-8fn8g   1/1     Running   0          14s

$ kubectl -n local-path-storage get configmap local-path-config -o jsonpath='{.data.config\.json}'
  "paths":["/var/local-path-provisioner"]
```

Both classes use the same provisioner and differ only in policy. Dynamic
provisioning is done by an ordinary Pod in `local-path-storage`, and it writes
into `/var/local-path-provisioner` on whichever node the consuming Pod lands on.
The quota showed the three new storage caps (`persistentvolumeclaims 12`,
`requests.storage 20Gi`, and the two per-class caps). Node mapping recorded:
`worker-node-1` = `dso202-p2-worker`, `worker-node-2` = `dso202-p2-worker2`.
([evidence](../evidence/stage1-cluster-and-storage-landscape.txt))

#### Screenshots

**Step 1 – cluster created; note the *Installing StorageClass* line**

![Step 1 – cluster created; note the *Installing StorageClass* line](<../screenshots/s1-step1-create-cluster.png>)

**Step 2 – Node names vs Docker container names**

![Step 2 – Node names vs Docker container names](<../screenshots/s1-step2-nodes.png>)

**Step 3 – host directory visible inside `dso202-p2-worker`**

![Step 3 – host directory visible inside `dso202-p2-worker`](<../screenshots/s1-step3-host-mount.png>)

**Step 4 – namespace, quota/limits and the `dso202-retain` class applied**

![Step 4 – namespace, quota/limits and the `dso202-retain` class applied](<../screenshots/s1-step4-namespace-quota-sc.png>)

**Step 5 – the quota, including the three storage caps**

![Step 5 – the quota, including the three storage caps](<../screenshots/s1-step5-quota.png>)

**Step 6 – both classes use `rancher.io/local-path`, only the policies differ**

![Step 6 – both classes use `rancher.io/local-path`, only the policies differ](<../screenshots/s1-step6-storageclass.png>)

**Step 7 – the provisioner Pod in `local-path-storage`**

![Step 7 – the provisioner Pod in `local-path-storage`](<../screenshots/s1-step7-provisioner.png>)

**Step 8 – where the provisioner writes on each node**

![Step 8 – where the provisioner writes on each node](<../screenshots/s1-step8-provisioner-path.png>)

### 3.2 Stage 2 – Static provisioning and the meaning of Retain

The claim bound immediately to `pv-web-static`, although `kubectl get
storageclass manual` returned `NotFound`: the class name was only a matching
label. The Pod was placed on `worker-node-1` with no `nodeSelector` of its own,
because the PV's `nodeAffinity` pins it there.

```
$ kubectl get pv            # after deleting the Pod and the claim
NAME            ... RECLAIM POLICY   STATUS     CLAIM
pv-web-static   ... Retain           Released   dso202-practical-02/pvc-web-static

$ kubectl delete pv pv-web-static
$ ls -l /tmp/dso202-p2-storage/pv-web-static/
.rw-r--r-- 128 ... ledger.txt

# new PV + new PVC + new Pod (Step 10)
$ kubectl exec static-writer -- cat /data/ledger.txt
2026-11-02T14:37:12Z start pod=static-writer node=worker-node-1
2026-11-02T14:37:20Z start pod=static-writer node=worker-node-1
2026-11-02T14:37:27Z start pod=static-writer node=worker-node-1
```

The PV became `Released`, not `Available`, because Retain will not hand one
claim's data to the next claim. Deleting the PV object removed an API entry and
not a single byte, so three brand-new objects adopted data written before any of
them existed. ([evidence](../evidence/stage2-static-provisioning.txt))

#### Screenshots

**Step 1 – static PV `Available`**

![Step 1 – static PV `Available`](<../screenshots/s2-step1-pv.png>)

**Step 2 – hostPath and nodeAffinity**

![Step 2 – hostPath and nodeAffinity](<../screenshots/s2-step2-pv-fields.png>)

**Step 3 – claim bound immediately**

![Step 3 – claim bound immediately](<../screenshots/s2-step3-pvc.png>)

**Step 4 – no StorageClass named `manual` exists**

![Step 4 – no StorageClass named `manual` exists](<../screenshots/s2-step4-no-manual-class.png>)

**Step 5 – Pod placed on worker-node-1 by the volume**

![Step 5 – Pod placed on worker-node-1 by the volume](<../screenshots/s2-step5-pod.png>)

**Step 6 – same line inside the container and on the host**

![Step 6 – same line inside the container and on the host](<../screenshots/s2-step6-read-file.png>)

**Step 7 – data outlives the Pod**

![Step 7 – data outlives the Pod](<../screenshots/s2-step7-recreate-pod.png>)

**Step 8 – PV `Released`, not `Available`**

![Step 8 – PV `Released`, not `Available`](<../screenshots/s2-step8-released.png>)

**Step 9 – deleting the PV deleted no data**

![Step 9 – deleting the PV deleted no data](<../screenshots/s2-step9-delete-pv.png>)

**Step 10 – new PV, PVC and Pod adopt the old data (three lines)**

![Step 10 – new PV, PVC and Pod adopt the old data (three lines)](<../screenshots/s2-step10-readopt.png>)

### 3.3 Stage 3 – Dynamic provisioning and two uncomfortable truths

```
$ kubectl describe pvc dynamic-data | tail -n 6
  Normal  WaitForFirstConsumer  3s  persistentvolume-controller  waiting for first consumer to be created before binding

$ kubectl get pv     # after the Pod was applied
pvc-4a2fbcf3-b9cf-4a28-b698-3d250a61d026   1Gi   RWO   Delete   Bound   dso202-practical-02/dynamic-data   standard

$ kubectl exec dynamic-writer -- df -h /data
/dev/vda1                58.4G     54.8G    554.5M  99% /data

$ kubectl patch pvc dynamic-data --type merge -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
Error from server (Forbidden): ... only dynamically provisioned pvc can be resized and the storageclass that provisions the pvc must support resize
```

The claim stayed `Pending` until a Pod used it, which is correct behaviour and
not a fault. The PV was created automatically (`pvc-<uid>`) with `Delete`
inherited from the class. The container saw the whole 58.4 GB node disk, not
1 Gi, so this provisioner does not enforce capacity. The resize was rejected by
the API server because the class has `allowVolumeExpansion: false`. After the
Pod and claim were deleted, the PV and the node directory
(`docker exec dso202-p2-worker2 ls /var/local-path-provisioner`) were both
gone. ([evidence](../evidence/stage3-dynamic-provisioning.txt))

#### Screenshots

**Step 1 – claim `Pending`**

![Step 1 – claim `Pending`](<../screenshots/s3-step1-pending.png>)

**Step 2 – `WaitForFirstConsumer` event**

![Step 2 – `WaitForFirstConsumer` event](<../screenshots/s3-step2-describe.png>)

**Step 3 – consumer Pod triggers provisioning; PV `pvc-<uid>` with `Delete`**

![Step 3 – consumer Pod triggers provisioning; PV `pvc-<uid>` with `Delete`](<../screenshots/s3-step3-consumer.png>)

**Step 4 – the volume is a directory on worker-node-2**

![Step 4 – the volume is a directory on worker-node-2](<../screenshots/s3-step4-node-dir.png>)

**Step 5 – the 1Gi volume sees the whole node disk**

![Step 5 – the 1Gi volume sees the whole node disk](<../screenshots/s3-step5-df.png>)

**Step 6 – resize rejected by the API server**

![Step 6 – resize rejected by the API server](<../screenshots/s3-step6-resize.png>)

**Step 7 – claim deleted: dynamic PV and node directory gone (only the static PV remains)**

![Step 7 – claim deleted: dynamic PV and node directory gone (only the static PV remains)](<../screenshots/s3-step7-delete.png>)

### 3.4 Stage 4 – Why a Deployment cannot own state

This stage is a deliberate failure, not a successful deployment.

```
$ kubectl get pods -l app=shared-writer -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
shared-writer-6f4b987c7b-4r5vt   worker-node-2
shared-writer-6f4b987c7b-n6228   worker-node-2
shared-writer-6f4b987c7b-p95kv   worker-node-2

$ kubectl exec deploy/shared-writer -- cat /data/visitors.log
2026-11-02T14:39:34Z pod=shared-writer-6f4b987c7b-p95kv node=worker-node-2
2026-11-02T14:39:34Z pod=shared-writer-6f4b987c7b-n6228 node=worker-node-2
2026-11-02T14:39:34Z pod=shared-writer-6f4b987c7b-4r5vt node=worker-node-2

# after deleting the running replicas
shared-writer-6f4b987c7b-cqgw2   worker-node-2
shared-writer-6f4b987c7b-g7mvb   worker-node-2
shared-writer-6f4b987c7b-mbz77   worker-node-2
```

- **Observation 1 – the volume decided placement.** All three replicas ran on
  `worker-node-2` although the Deployment states no node preference. The one
  claim was bound to a directory on that node, so no other node could take the
  Pods.
- **Observation 2 – one set of data, not three.** Three processes appended to
  the same `visitors.log`. Harmless for this log, but two PostgreSQL processes
  on one data directory would corrupt it, and a Deployment has no syntax to give
  replica 2 a different claim from replica 1.
- **Observation 3 – no identity survives.** Every replacement got a new random
  suffix, so nothing can refer to "the first replica" across restarts.

It "worked" here only because `ReadWriteOnce` means one *node*, and all replicas
were forced onto one node. On a cloud cluster with a zonal disk, replicas
scheduled elsewhere would fail with a multi-attach error.
([evidence](../evidence/stage4-deployment-shared-pvc.txt))

#### Screenshots

**Observation 1 – all three replicas on one node**

![Observation 1 – all three replicas on one node](<../screenshots/s4-step1-shared-deploy.png>)

**Observation 2 – one shared file written by three replicas**

![Observation 2 – one shared file written by three replicas](<../screenshots/s4-step2-shared-file.png>)

**Observation 3 – replacements get brand-new names**

![Observation 3 – replacements get brand-new names](<../screenshots/s4-step3-identity.png>)

**The shared log now mixes six different Pod names**

![The shared log now mixes six different Pod names](<../screenshots/s4-step3b-shared-file-after.png>)

**Step 4 – experiment removed**

![Step 4 – experiment removed](<../screenshots/s4-step4-remove.png>)

### 3.5 Stage 5 – StatefulSets and stable identity

```
$ kubectl get service webnote
webnote   ClusterIP   None   <none>   80/TCP

$ kubectl get pods -l app=webnote -w        (abridged)
webnote-0   0/1   Pending ... webnote-0   1/1   Running
webnote-1   0/1   Pending ... webnote-1   1/1   Running
webnote-2   0/1   Pending ... webnote-2   1/1   Running

$ kubectl get pvc -l app=webnote
content-webnote-0   Bound   pvc-06ae3dbb-...   1Gi   RWO   standard
content-webnote-1   Bound   pvc-a6d09c68-...   1Gi   RWO   standard
content-webnote-2   Bound   pvc-a4111d16-...   1Gi   RWO   standard

$ kubectl exec client -- nslookup webnote.dso202-practical-02.svc.cluster.local
Address: 10.244.2.16
Address: 10.244.2.14
Address: 10.244.1.6
$ kubectl exec client -- nslookup webnote-1.webnote.dso202-practical-02.svc.cluster.local
Address: 10.244.1.6
```

`webnote-1` was not created until `webnote-0` was `1/1` (OrderedReady). One
template produced three claims named `content-webnote-<ordinal>`, and the Pods
spread over both workers, unlike Stage 4. The headless Service returned every
Pod address for the set name and exactly one address for each Pod name. (This
busybox `nslookup` prints only addresses, not the reverse names shown in the
guide.) A note written into `webnote-0` did not appear in `webnote-1`, so the
volumes are private.

```
# webnote-1 before deletion: worker-node-1  10.244.1.6
$ kubectl delete pod webnote-1
# after:                     worker-node-1  10.244.1.7
$ kubectl get pvc content-webnote-1
content-webnote-1   Bound   pvc-a6d09c68-34e4-4b57-b21e-f8c4d447da19   ...   AGE 72s
$ kubectl exec client -- wget -qO- http://webnote-1.webnote.dso202-practical-02.svc.cluster.local
<h2>webnote-1</h2>
created: 2026-11-02T14:43:54Z on worker-node-1
started: 2026-11-02T14:43:54Z on worker-node-1
started: 2026-11-02T14:44:56Z on worker-node-1
```

All four guarantees appear in one output. The **name** is unchanged. The
**claim** is the same volume (its age is older than the new Pod). The
`created:` line kept its original timestamp, so the **data** was reattached
rather than regenerated. The **IP** changed from `.6` to `.7`, which is why
clients must use the DNS name. ([evidence](../evidence/stage5-statefulset-identity.txt))

#### Screenshots

**Step 1 – headless Service, `CLUSTER-IP None`**

![Step 1 – headless Service, `CLUSTER-IP None`](<../screenshots/s5-step1-headless.png>)

**Step 2 – ordered creation: webnote-1 starts only after webnote-0 is Ready (watch stopped with Ctrl-C)**

![Step 2 – ordered creation: webnote-1 starts only after webnote-0 is Ready (watch stopped with Ctrl-C)](<../screenshots/s5-step2-ordered-create.png>)

**Step 3 – one claim per ordinal**

![Step 3 – one claim per ordinal](<../screenshots/s5-step3-claims.png>)

**Step 4 – Pods spread across both workers**

![Step 4 – Pods spread across both workers](<../screenshots/s5-step4-placement.png>)

**Step 5 – the set name resolves to three Pods; the per-Pod name to one**

![Step 5 – the set name resolves to three Pods; the per-Pod name to one](<../screenshots/s5-step5-dns.png>)

**Step 6 – page served by one specific Pod**

![Step 6 – page served by one specific Pod](<../screenshots/s5-step6-one-pod.png>)

**Step 7 – EndpointSlice lists the Pod IPs**

![Step 7 – EndpointSlice lists the Pod IPs](<../screenshots/s5-step7-endpointslice.png>)

**Step 8 – a note written to webnote-0 only**

![Step 8 – a note written to webnote-0 only](<../screenshots/s5-step8-private.png>)

**Step 9 – webnote-1 before deletion**

![Step 9 – webnote-1 before deletion](<../screenshots/s5-step9a-before-delete.png>)

**Step 9 – same name, same claim, original `created:` line, new IP**

![Step 9 – same name, same claim, original `created:` line, new IP](<../screenshots/s5-step9-survive.png>)

### 3.6 Stage 6 – Scaling, retention and ordered updates

```
$ kubectl scale statefulset webnote --replicas=2      (watch, abridged)
webnote-3   1/1   Terminating ...
webnote-2   1/1   Terminating ...

$ kubectl get pvc -l app=webnote        # two Pods, four claims
content-webnote-0 / -1 / -2 / -3   Bound

$ kubectl scale statefulset webnote --replicas=3
$ kubectl exec client -- wget -qO- http://webnote-2.webnote...
created: 2026-11-02T14:44:45Z on worker-node-2
started: 2026-11-02T14:44:45Z on worker-node-2
started: 2026-11-02T14:46:40Z on worker-node-2
```

Pods were removed highest ordinal first, one at a time. Their claims stayed
because `whenScaled: Retain`, and the returning `webnote-2` reattached its own
claim by name, with the original `created:` timestamp. The ordinal is the
identity that links a replica to its data.

The partitioned update was done by editing the committed manifest
(`partition: 2`, image `nginx:1.31-alpine`):

```
NAME        IMAGE
webnote-0   nginx:1.30-alpine
webnote-1   nginx:1.30-alpine
webnote-2   nginx:1.31-alpine
```

Only ordinals ≥ 2 were updated. Setting `partition: 0` finished the rollout in
descending order, and every `created:` line was unchanged. `kubectl explain
statefulset.spec.updateStrategy.rollingUpdate` on this v1.36 cluster lists
`maxUnavailable` as "beta-level and enabled by default", with the caveat that
it "might not be effective for the OrderedReady podManagementPolicy".
Finally, `kubectl delete statefulset webnote` removed every Pod but left all four
claims (`whenDeleted: Retain`). Re-applying the manifest brought `webnote-1`
back with its `created: 14:43:54Z` line and two further `started:` lines.
([evidence](../evidence/stage6-scaling-and-retention.txt))

#### Screenshots

**Step 1 – scaling up creates a fourth claim**

![Step 1 – scaling up creates a fourth claim](<../screenshots/s6-step1-scale-up.png>)

**Step 2 – termination in descending order: webnote-3, then webnote-2**

![Step 2 – termination in descending order: webnote-3, then webnote-2](<../screenshots/s6-step2-scale-down.png>)

**Step 3 – two Pods, four claims (`whenScaled: Retain`)**

![Step 3 – two Pods, four claims (`whenScaled: Retain`)](<../screenshots/s6-step3-claims-retained.png>)

**Step 4 – returning webnote-2 has its original `created:` line**

![Step 4 – returning webnote-2 has its original `created:` line](<../screenshots/s6-step4-scale-back.png>)

**Step 5 – `partition: 2`: only webnote-2 runs nginx 1.31**

![Step 5 – `partition: 2`: only webnote-2 runs nginx 1.31](<../screenshots/s6-step5-partition-2.png>)

**Step 6 – `partition: 0`: rollout completes; data unchanged**

![Step 6 – `partition: 0`: rollout completes; data unchanged](<../screenshots/s6-step6-partition-0.png>)

**`maxUnavailable` is beta on this cluster version**

![`maxUnavailable` is beta on this cluster version](<../screenshots/s6-step6b-explain-rollingupdate.png>)

**Step 7 – StatefulSet deleted, all four claims kept**

![Step 7 – StatefulSet deleted, all four claims kept](<../screenshots/s6-step7-delete-sts.png>)

**Step 8 – recreated StatefulSet returns with its data**

![Step 8 – recreated StatefulSet returns with its data](<../screenshots/s6-step8-recreate-sts.png>)

### 3.7 Stage 7 – PostgreSQL

```
$ kubectl get secret postgres-credentials -o jsonpath='{.data.POSTGRES_USER}' | base64 -d
taskuser
$ kubectl get services
postgres            ClusterIP   10.96.188.63   5432/TCP
postgres-headless   ClusterIP   None           5432/TCP

$ kubectl get pv     (database volume)
pvc-98678389-...   2Gi   RWO   Retain   Bound   dso202-practical-02/data-postgres-0   dso202-retain
$ kubectl exec postgres-0 -- sh -c 'echo "$PGDATA"; ls /var/lib/postgresql'
/var/lib/postgresql/18/docker
18

$ kubectl exec postgres-0 -- psql ... -c "SELECT id, title, done FROM tasks ORDER BY id;"
  1 | Complete Practical 2 | f
  2 | Read Unit II notes   | f
  3 | Draft the report     | f
$ kubectl delete pod postgres-0
$ kubectl exec postgres-0 -- psql ... -c "SELECT count(*) FROM tasks;"
     3
$ kubectl logs postgres-0 | head
PostgreSQL Database directory appears to contain a database; Skipping initialization
... LOG:  database system was shut down at 2026-11-02 14:54:56 UTC
```

The Secret was readable with one command, so it is encoding, not encryption.
The volume is mounted at `/var/lib/postgresql`, and PostgreSQL 18 keeps its data
one level down in `18/docker`. The three rows written before the Pod was deleted
were read back through a Pod that did not exist when they were written. The log
shows that initialisation was skipped and that the previous shutdown was clean.
`postgres.<ns>.svc.cluster.local` resolved to the ClusterIP `10.96.188.63` (what
an application uses). `postgres-0.postgres-headless.<ns>.svc.cluster.local`
resolved to the Pod IP `10.244.2.25` (what a backup job or replica uses).
([evidence](../evidence/stage7-postgresql.txt))

#### Screenshots

**Step 1 – a Secret is encoded, not encrypted**

![Step 1 – a Secret is encoded, not encrypted](<../screenshots/s7-step1-secret.png>)

**Step 2 – headless and ClusterIP Services**

![Step 2 – headless and ClusterIP Services](<../screenshots/s7-step2-services.png>)

**Step 3 – postgres-0 created (watch stopped during a 4-minute image pull)**

![Step 3 – postgres-0 created (watch stopped during a 4-minute image pull)](<../screenshots/s7-step3-deploy.png>)

**Step 3 – events and the first-start initialisation log**

![Step 3 – events and the first-start initialisation log](<../screenshots/s7-step3b-events-readiness.png>)

**Step 4 – claim on `dso202-retain`, PV with `Retain`**

![Step 4 – claim on `dso202-retain`, PV with `Retain`](<../screenshots/s7-step4-log-storage.png>)

**Step 5 – PGDATA is a subdirectory of the mount point**

![Step 5 – PGDATA is a subdirectory of the mount point](<../screenshots/s7-step5-pgdata.png>)

**Step 6 – table created, three rows inserted**

![Step 6 – table created, three rows inserted](<../screenshots/s7-step6-write-rows.png>)

**Step 7 – rows still present after the Pod was deleted and replaced**

![Step 7 – rows still present after the Pod was deleted and replaced](<../screenshots/s7-step7-delete-pod.png>)

**Step 8 – *Skipping initialization*: the existing data directory was reused**

![Step 8 – *Skipping initialization*: the existing data directory was reused](<../screenshots/s7-step8-recovery-log.png>)

**Step 9 – claim reused; both Service names resolve**

![Step 9 – claim reused; both Service names resolve](<../screenshots/s7-step9-claim-dns.png>)

### 3.8 Stage 8 – Cleanup and the cost of Retain

Final state and a logical backup were captured first
([final-state-all](../evidence/final-state-all.txt),
[final-state-storage](../evidence/final-state-storage.txt),
[tasktracker-dump.sql](../evidence/tasktracker-dump.sql)).

```
$ kubectl get pvc        # after every workload was deleted
content-webnote-0..3, data-postgres-0, pvc-web-static   all Bound   (6 claims)

$ kubectl delete pvc --all
$ kubectl get pv
pv-web-static      1Gi   RWO   Retain   Released   dso202-practical-02/pvc-web-static
pvc-98678389-...   2Gi   RWO   Retain   Released   dso202-practical-02/data-postgres-0

$ kubectl delete pv pvc-98678389-4172-4777-a6f3-e8f5812a94ea
$ docker exec dso202-p2-worker2 ls /var/local-path-provisioner
pvc-98678389-4172-4777-a6f3-e8f5812a94ea_dso202-practical-02_data-postgres-0

$ kind delete cluster --name dso202-p2
$ cat /tmp/dso202-p2-storage/pv-web-static/ledger.txt
2026-11-02T14:37:12Z ... 2026-11-02T14:37:20Z ... 2026-11-02T14:37:27Z
```

Deleting the workloads removed no claims, because StatefulSet claims are in no
manifest. Deleting the claims destroyed the four `standard` volumes. The two
`Retain` volumes went to `Released` with their data intact, and even deleting the
PV object left the PostgreSQL directory on the node. Only deleting the cluster
removed it, because it lived inside a node container. The static ledger is still
on the host, because it was never inside the cluster at all.
A `kubectl diff` before cleanup showed every manifest matching the cluster
except 06/07, which Stage 3 deleted on purpose
([stage8-kubectl-diff](../evidence/stage8-kubectl-diff.txt),
[stage8-cleanup](../evidence/stage8-cleanup.txt)).

#### Screenshots

**Step 1 – evidence and SQL dump captured before deleting anything**

![Step 1 – evidence and SQL dump captured before deleting anything](<../screenshots/s8-step1-capture.png>)

**Step 2 – workloads deleted**

![Step 2 – workloads deleted](<../screenshots/s8-step2-delete-workloads.png>)

**Step 3 – six claims survive**

![Step 3 – six claims survive](<../screenshots/s8-step3-claims-survive.png>)

**Step 4 – `Delete` volumes gone, two `Retain` volumes `Released`**

![Step 4 – `Delete` volumes gone, two `Retain` volumes `Released`](<../screenshots/s8-step4-delete-claims.png>)

**Step 5 – PV object deleted, PostgreSQL data still on the node**

![Step 5 – PV object deleted, PostgreSQL data still on the node](<../screenshots/s8-step5-reclaim.png>)

**Step 5 – even after its PV is deleted, the database directory remains**

![Step 5 – even after its PV is deleted, the database directory remains](<../screenshots/s8-step5b-delete-pg-pv.png>)

**Step 6 – cluster deleted**

![Step 6 – cluster deleted](<../screenshots/s8-step6-delete-cluster.png>)

**Step 7 – statically provisioned data survives the cluster**

![Step 7 – statically provisioned data survives the cluster](<../screenshots/s8-step7-host-survives.png>)

### 3.9 Extension tasks

**Extension 1 – `whenScaled: Delete`** (`extensions/ext1-statefulset-whenscaled-delete.yaml`).
Three claims were created, then scaling to one replica left only
`content-webnote-ext1-0`; the claims for ordinals 1 and 2 were deleted together
with their Pods. `Retain` is the default because scale-down is often an incident
response, and destroying data during an incident cannot be undone. `Delete` is
right for replicas whose data is disposable cache or can be rebuilt, for example
a Redis cache tier or a Kafka/Elasticsearch-style node that re-replicates
its data from peers when it rejoins.

**Extension 2 – changing an immutable field** (`extensions/ext2-statefulset-parallel.yaml`).
Applying `podManagementPolicy: Parallel` over the running StatefulSet returned:

```
The StatefulSet "webnote" is invalid: spec: Forbidden: updates to statefulset spec for fields other than
'replicas', 'ordinals', 'template', 'updateStrategy', 'revisionHistoryLimit',
'persistentVolumeClaimRetentionPolicy' and 'minReadySeconds' are forbidden
```

So the mutable fields are exactly those seven. `serviceName`, `selector`,
`podManagementPolicy` and `volumeClaimTemplates` are fixed at creation.
After deleting the StatefulSet (claims retained) and re-applying, all three Pods
went `Pending` → `Init` → `Running` within the same second, unlike the
one-at-a-time order in Stage 5. The graded manifest (OrderedReady) was restored
afterwards.

**Extension 3 – `ordinals.start`** (`extensions/ext3-statefulset-ordinals.yaml`).
`kubectl explain statefulset.spec.ordinals.start` confirmed the field exists on
v1.36. With `start: 2` and two replicas, the Pods were `webnote-ext3-2` and
`webnote-ext3-3`, and the claims were `content-webnote-ext3-2` and
`content-webnote-ext3-3`. Migration use: when moving a database between clusters,
the new StatefulSet can own ordinals 2+ while the old one keeps 0–1. Replicas
then move one ordinal at a time without two Pods ever holding the same identity.

**Extension 4 – restore from the dump** ([evidence](../evidence/extension-4-restore-from-dump.txt)).
The cluster was rebuilt from the repository and PostgreSQL redeployed. `SELECT
count(*)` first failed with `relation "tasks" does not exist`. Then
`kubectl exec -i postgres-0 -- psql -U taskuser -d tasktracker < evidence/tasktracker-dump.sql`
restored all three rows. I then ran `DROP TABLE tasks;` and deleted the Pod:
the table was still missing afterwards, because volume persistence faithfully
keeps a mistaken change. Restoring the dump brought it back. Volume-level
persistence protects against losing a *Pod*; only a backup held outside the
cluster protects against a dropped table.

#### Extension screenshots

**Extension 1 – `whenScaled: Delete`, three claims**

![Extension 1 – `whenScaled: Delete`, three claims](<../screenshots/ext1-a-apply.png>)

**Extension 1 – scale to 1: two claims deleted with their Pods**

![Extension 1 – scale to 1: two claims deleted with their Pods](<../screenshots/ext1-b-scale-down.png>)

**Extension 1 – cleanup**

![Extension 1 – cleanup](<../screenshots/ext1-c-cleanup.png>)

**Extension 2 – `podManagementPolicy` is immutable**

![Extension 2 – `podManagementPolicy` is immutable](<../screenshots/ext2-a-immutable.png>)

**Extension 2 – fields of `statefulset.spec`**

![Extension 2 – fields of `statefulset.spec`](<../screenshots/ext2-b-explain.png>)

**Extension 2 – Parallel: all three Pods start at once**

![Extension 2 – Parallel: all three Pods start at once](<../screenshots/ext2-c-parallel-create.png>)

**Extension 2 – original OrderedReady StatefulSet restored**

![Extension 2 – original OrderedReady StatefulSet restored](<../screenshots/ext2-d-restore.png>)

**Extension 3 – `ordinals.start: 2` gives webnote-ext3-2 and -3**

![Extension 3 – `ordinals.start: 2` gives webnote-ext3-2 and -3](<../screenshots/ext3-a-ordinals.png>)

**Extension 3 – cleanup**

![Extension 3 – cleanup](<../screenshots/ext3-b-cleanup.png>)

**Extension 4 – cluster rebuilt, database redeployed**

![Extension 4 – cluster rebuilt, database redeployed](<../screenshots/ext4-a-rebuild.png>)

**Extension 4 – empty database; restored from the dump**

![Extension 4 – empty database; restored from the dump](<../screenshots/ext4-b-restore.png>)

**Extension 4 – a dropped table survives a Pod restart: the volume keeps the mistake**

![Extension 4 – a dropped table survives a Pod restart: the volume keeps the mistake](<../screenshots/ext4-c-drop-table.png>)

**Extension 4 – only the dump brings it back**

![Extension 4 – only the dump brings it back](<../screenshots/ext4-d-restore-again.png>)

**Extension 4 – final teardown, no kind node containers remain**

![Extension 4 – final teardown, no kind node containers remain](<../screenshots/ext4-e-teardown.png>)

## 4. Analysis

**Q1. Why did the Stage 3 claim stay Pending while the Stage 2 claim bound at once?**
The field is `volumeBindingMode` on the StorageClass. `standard` uses
`WaitForFirstConsumer`, and the event said "waiting for first consumer to be
created before binding". The Stage 2 claim named class `manual`, for which no
StorageClass object exists (`NotFound`), so no binding mode applied and the
control plane bound it at once to the matching `Available` PV. The design reason
is topology: with node-local or zonal storage, creating the volume before the
scheduler has picked a node could put the data where the Pod cannot run. Waiting
lets the scheduler choose the node first and the provisioner create the volume
there (the `dynamic-writer` volume appeared on `worker-node-2`, the node the Pod
was scheduled to).

**Q2. Which object decided whether data survived claim deletion, and who chooses it?**
The reclaim policy. For the dynamic volume it came from the **StorageClass**
(`standard`, `RECLAIMPOLICY Delete`), and the PV inherited `Delete`. For the static
volume it was set on the **PersistentVolume** itself
(`persistentVolumeReclaimPolicy: Retain`). Nothing in the claim can override it.
In an organisation, StorageClasses and static PVs are created by the platform or
cluster administrators, not by the developer who writes the claim. They pick
the value, usually `Retain` for database classes.

**Q3. Why were all three Stage 4 replicas on one node?**
The single claim `shared-data` was bound by the first Pod to a local-path
volume on `worker-node-2`. That PV carries node affinity for that node, so the
scheduler's volume-binding check filtered out every other node for the remaining
replicas. Because `ReadWriteOnce` allows many Pods *on one node*, all three
mounted it. On a managed cluster with a zonal disk (for example AWS EBS), the
disk can only be attached to one node at a time. Replicas the scheduler placed
on other nodes would stay in `ContainerCreating` with a `Multi-Attach error for
volume ... Volume is already used by pod(s)` event, and replicas in another zone
could not be placed at all.

**Q4. FQDN of the second webnote replica, and what must exist for it to resolve.**
`webnote-1.webnote.dso202-practical-02.svc.cluster.local` (it resolved to
`10.244.1.6`, later `10.244.1.7`). Required: the Namespace `dso202-practical-02`;
the headless Service `webnote` (`clusterIP: None`) whose selector matches the
Pod; the StatefulSet `webnote` with `serviceName: webnote`, which sets the Pod's
hostname/subdomain; the Pod `webnote-1` itself, **Ready** (because
`publishNotReadyAddresses: false`); the EndpointSlice the controller generated
for the Service; and CoreDNS running in `kube-system`.

**Q5. What happened to the claims when scaling 4 → 2 → 3?**
At 4 replicas a fourth claim `content-webnote-3` was created (count 4). Scaling
to 2 terminated `webnote-3` then `webnote-2`, but all four claims stayed
`Bound`. Scaling back to 3 recreated `webnote-2`, which reattached the existing
`content-webnote-2` (original `created: 14:44:45Z`). `content-webnote-3` remained
unused. The fields are `persistentVolumeClaimRetentionPolicy.whenScaled` and
`.whenDeleted`. Both default to `Retain`, and Extension 1 showed the opposite
behaviour with `whenScaled: Delete`.

**Q6. Why mount at `/var/lib/postgresql` and not the data directory?**
PostgreSQL 18's image puts `PGDATA` at `/var/lib/postgresql/18/docker` (shown by
`echo $PGDATA`). Mounting the volume at the parent means `initdb` creates and
initialises a fresh, empty subdirectory inside the volume. If the volume were
mounted directly at the data directory and the storage backend had put anything
there (for example `lost+found` on an ext4 block device), `initdb` would refuse
to initialise a non-empty directory. The container would exit and go into
`CrashLoopBackOff` on first start only, and `kubectl logs postgres-0 --previous`
would show the "directory exists but is not empty" error. For PostgreSQL ≤ 17
the equivalent fix is `subPath`.

**Q7. Two things a StatefulSet does not provide.**
(1) *Replication and failover*: three replicas are three independent databases.
In production this is provided by a database **Operator** (e.g. CloudNativePG,
Zalando Postgres Operator; Unit II 2.4), which configures streaming
replication and leader election. (2) *Backup*: a retained volume is not a backup.
Extension 4 showed `DROP TABLE` surviving a Pod restart. Backups come from
scheduled logical dumps (a **CronJob** running `pg_dump`), Operator-managed WAL
archiving to object storage, or **VolumeSnapshots** through a CSI driver. A third
gap is that access to Secrets must be restricted with **RBAC**.

**Q8. Why were the two PVs `Released` rather than `Available` in Stage 8?**
With `Retain`, deleting the claim does not wipe the volume, and the PV still
records its old claim (`CLAIM dso202-practical-02/data-postgres-0` even after the
claim was gone). Kubernetes will not bind a volume that may hold one workload's
data to a different claim. To return it to service, an administrator must decide
what to do with the data (back it up, wipe it or keep it). Then they either
delete and recreate the PV, or remove `spec.claimRef` from the PV (e.g.
`kubectl patch pv <name> --type json -p '[{"op":"remove","path":"/spec/claimRef"}]'`)
so that it becomes `Available` again. For dynamically provisioned volumes they
must also delete the backing storage by hand, as the leftover
`/var/local-path-provisioner/pvc-98678389-...` directory showed.

## 5. Reflection

*(Draft based on what happened in this run. Review it and put it in your own
words before submitting.)*

The hardest idea was keeping three things apart: the PersistentVolume *object*,
the claim, and the bytes on disk. Stage 2 made it concrete. I deleted the PV and
`ls` on the host still showed `ledger.txt`. In Stage 8, deleting the database
PV still left its directory on `worker-node-2`.

The most important error I met was not in the guide. In Stage 3,
`kubectl exec dynamic-writer -- df -h /data` reported **`58.4G size, 554.5M
available, 99% used`**. That was meant to show that the 1 Gi request is not
enforced, but it also meant the Docker Desktop VM was almost out of disk, and
PostgreSQL and nginx 1.31 images still had to be pulled. I checked
`kubectl describe node worker-node-1 | grep -i pressure` (no `DiskPressure`
yet, because kind relaxes the kubelet's disk eviction thresholds) and then
`docker system df`, which showed **21.35 GB of build cache** from unrelated
projects. I cleared only that with `docker builder prune -f`, after which the
node reported 26 GB free. Without the `df` step, the likely symptom later would
have been an `ImagePullBackOff` or `no space left on device` that looked like a
Kubernetes problem. Lesson: check `docker system df` in Stage 0, not just the
host's `df -h /`, because on macOS the two report different filesystems.

A second, smaller error was in my evidence capture. `kubectl get pods -o
custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image` failed in
zsh with `no matches found`: zsh treats `[0]` as a filename glob. Quoting the
argument fixed it. I also noticed image pulls dominated the timings:
`webnote-0` sat between `PodInitializing` and `Running` for about 3 min 30 s,
and `postgres-0` in `ContainerCreating` for about 4 min 45 s, both on first pull
of the image. With the readiness probes in place this caused no errors, only
delay. Extension 2's error message was also useful: it lists the mutable
StatefulSet fields directly, which is more reliable than memory.

If I repeated the practical, I would check Docker's internal disk at the start.
I would pre-pull `postgres:18-alpine` and `nginx:1.31-alpine` into the nodes
(`docker exec <node> crictl pull ...`) before stages that time out. And I would
write `pg_dump` into the procedure right after the data is inserted, not only at
cleanup.

One thing that remains unclear to me: the guide's Stage 4 Step 3 says "delete
one replica", but the command (`--field-selector status.phase=Running`) deletes
all running replicas, which is what I observed. I am also not sure how the
scheduler decided to put `webnote-0` and `webnote-2` both on `worker-node-2`
rather than spreading them evenly. No spread constraint was set, so I assume it
followed the default scoring, but I have not verified which plugin decided it.

## 6. References

1. DSO202 Practical 2 Guide and companion file `DSO202_Practical2_Manifests.md` (HackMD). Accessed 2 November 2026.
2. Kubernetes documentation, "Persistent Volumes". https://kubernetes.io/docs/concepts/storage/persistent-volumes/ Accessed 2 November 2026.
3. Kubernetes documentation, "Storage Classes". https://kubernetes.io/docs/concepts/storage/storage-classes/ Accessed 2 November 2026.
4. Kubernetes documentation, "StatefulSets". https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/ Accessed 2 November 2026.
5. Kubernetes documentation, "DNS for Services and Pods". https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/ Accessed 2 November 2026.
6. Kubernetes documentation, "Secrets". https://kubernetes.io/docs/concepts/configuration/secret/ Accessed 2 November 2026.
7. Docker Hub, "postgres" official image documentation (PGDATA and PostgreSQL 18 layout). https://hub.docker.com/_/postgres Accessed 2 November 2026.
8. kind documentation, "Configuration" (extraMounts, kubeadmConfigPatches). https://kind.sigs.k8s.io/docs/user/configuration/ Accessed 2 November 2026.
9. Rancher local-path-provisioner README. https://github.com/rancher/local-path-provisioner Accessed 2 November 2026.