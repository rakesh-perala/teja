# Kubernetes Persistent Storage – PV, PVC & Pod Hands-on Lab

## 1. Lab Overview

In this hands-on lab, we will learn how to configure persistent storage in Kubernetes using:

1. PersistentVolume (PV)
2. PersistentVolumeClaim (PVC)
3. Pod
4. VolumeMounts
5. StorageClass
6. Dynamic Provisioning
7. Data Persistence Testing

**Main objective:**

Create storage, attach it to a Kubernetes Pod, store a file inside the mounted directory, delete the Pod, recreate it and verify that the previously stored data is still available.

---

## 2. Environment Details

| Component | Details |
|---|---|
| Operating System | Ubuntu 26.04 WSL2 |
| Kubernetes | Minikube |
| Kubernetes Version | v1.35.8 |
| Node Name | minikube |
| Container Image | busybox |
| StorageClass | standard |
| Storage Capacity | 1Gi |
| Access Mode | ReadWriteOnce |
| Namespace | default |

### Check Kubernetes Nodes

```bash
kubectl get nodes
```

**Output:**

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   69d   v1.35.8
```

### Check Existing Pods

```bash
kubectl get pods
```

**Output:**

```text
NAME                                   READY   STATUS    RESTARTS       AGE
hotstar-pod                            1/1     Running   6 (52s ago)    47d
nginx-node-selector-55db7bdccd-792s8   1/1     Running   3 (52s ago)    45d
nginx-node-selector-55db7bdccd-h6qpv   1/1     Running   3 (52s ago)    45d
payment-svc                            1/1     Running   12 (52s ago)   46d
```

---

## 3. Understand Kubernetes Persistent Storage

### What is PersistentVolume (PV)?

PersistentVolume is a storage resource available in a Kubernetes cluster.

It represents storage that can be used by applications.

### What is PersistentVolumeClaim (PVC)?

PersistentVolumeClaim is a request for storage made by a user or application.

For example, an application requests 1Gi of storage.

### What is Pod?

A Pod is the smallest deployable unit in Kubernetes.

A Pod can use persistent storage by referencing a PVC.

### Storage Architecture

```text
             Kubernetes Cluster
                     |
                     |
                 Pod
               demo-pod
                     |
                     |
               Container
                 BusyBox
                     |
                     |
              Volume Mount
                 /data
                     |
                     |
             PersistentVolumeClaim
                  demo-pvc
                     |
                     |
              PersistentVolume
                  demo-pv
                     |
                     |
              Actual Storage
```

**Important:**

A Pod references a PVC. It does not directly reference a PV.

---

# PART 1: Create PersistentVolume (PV)

## 4. Create PV YAML File

Create a file:

```bash
nano my-pv.yaml
```

### my-pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume

metadata:
  name: demo-pv

spec:
  capacity:
    storage: 1Gi

  accessModes:
    - ReadWriteOnce

  hostPath:
    path: "/mnt/data"

  persistentVolumeReclaimPolicy: Retain
```

### Explain Each Field

| Field | Explanation |
|---|---|
| apiVersion | Kubernetes API version |
| kind | Resource type |
| metadata.name | PersistentVolume name |
| capacity.storage | Storage capacity |
| accessModes | Access permission |
| ReadWriteOnce | Read-write access from one node at a time |
| hostPath | Uses a directory from the Kubernetes node |
| path | Actual node directory |
| persistentVolumeReclaimPolicy | Defines what happens to the PV after PVC deletion |

### Apply PV

```bash
kubectl apply -f my-pv.yaml
```

**Output:**

```text
persistentvolume/demo-pv created
```

### Verify PV

```bash
kubectl get pv
```

**Output:**

```text
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
demo-pv   1Gi        RWO            Retain           Available
```

**Explanation:**

The PV is created successfully.

Its status is Available because no PVC has claimed it yet.

---

# PART 2: Create PersistentVolumeClaim (PVC)

## 5. Create PVC YAML File

Create a file:

```bash
nano my-pvc.yaml
```

### my-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: demo-pvc

spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 1Gi
```

### Explain Each Field

| Field | Explanation |
|---|---|
| apiVersion | Kubernetes API version |
| kind | PersistentVolumeClaim |
| metadata.name | PVC name |
| accessModes | Required storage access mode |
| resources.requests.storage | Requested storage size |

### Apply PVC

```bash
kubectl apply -f my-pvc.yaml
```

**Output:**

```text
persistentvolumeclaim/demo-pvc created
```

### Verify PVC

```bash
kubectl get pvc
```

**Output:**

```text
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
demo-pvc   Bound    pvc-b72bd49d-5c54-47c1-9467-fa45ac1063c2   1Gi        RWO            standard
```

### Important Observation

We created a manual PV named:

```text
demo-pv
```

But our PVC was automatically bound to:

```text
pvc-b72bd49d-5c54-47c1-9467-fa45ac1063c2
```

Why?

Because our PVC did not specify a StorageClass.

Kubernetes selected the default StorageClass named `standard`.

Minikube automatically provisioned another PV for our PVC.

This process is called **Dynamic Provisioning**.

Our manually created `demo-pv` remained Available and was not used by this PVC.

---

# PART 3: Create Pod Using PVC

## 6. Create Pod YAML File

Create a file:

```bash
nano my-pod.yaml
```

### my-pod.yaml

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: demo-pod

spec:
  containers:
    - name: app
      image: busybox

      command:
        - sh
        - -c
        - sleep 3600

      volumeMounts:
        - mountPath: "/data"
          name: storage

  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: demo-pvc
```

### Explain Each Field

| Field | Explanation |
|---|---|
| kind | Pod resource |
| metadata.name | Pod name |
| containers | Container configuration |
| name | Container name |
| image | BusyBox image |
| command | Keeps the container running for 3600 seconds |
| volumeMounts | Mounts storage inside the container |
| mountPath | Storage mount location |
| volumes | Defines Pod volumes |
| persistentVolumeClaim | Uses an existing PVC |
| claimName | PVC name |

**Important:**

```yaml
claimName: demo-pvc
```

We must provide the PVC name, not the PV name.

### Apply Pod

```bash
kubectl apply -f my-pod.yaml
```

**Output:**

```text
pod/demo-pod created
```

### Verify Pod

```bash
kubectl get pods
```

**Output:**

```text
NAME                                   READY   STATUS    RESTARTS       AGE
demo-pod                               1/1     Running   0              8s
hotstar-pod                            1/1     Running   6 (31m ago)    47d
nginx-node-selector-55db7bdccd-792s8   1/1     Running   3 (31m ago)    45d
nginx-node-selector-55db7bdccd-h6qpv   1/1     Running   3 (31m ago)    45d
payment-svc                            1/1     Running   12 (31m ago)   46d
```

Our Pod is running successfully.

---

# PART 4: Verify PV and PVC Binding

## 7. Check PV

```bash
kubectl get pv
```

**Important output from our lab:**

```text
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM
demo-pv                                    1Gi        RWO            Retain           Available
pvc-b72bd49d-5c54-47c1-9467-fa45ac1063c2   1Gi        RWO            Delete           Bound       default/demo-pvc
```

Notice:

- `demo-pv` is Available.
- The dynamically created PV is Bound.
- Our PVC is using the dynamically created PV.

## 8. Check PVC

```bash
kubectl get pvc
```

**Output:**

```text
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
demo-pvc   Bound    pvc-b72bd49d-5c54-47c1-9467-fa45ac1063c2   1Gi        RWO            standard
```

Our PVC is Bound successfully.

---

# PART 5: Describe PVC and Pod

## 9. Describe PVC

```bash
kubectl describe pvc demo-pvc
```

**Actual output from our lab:**

```text
Name:          demo-pvc
Namespace:     default
StorageClass:  standard
Status:        Bound
Volume:        pvc-b72bd49d-5c54-47c1-9467-fa45ac1063c2
Capacity:      1Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       demo-pod
```

### Important Events

```text
Normal  ExternalProvisioning
Waiting for a volume to be created either by the external provisioner
or manually by the system administrator.

Normal  Provisioning
External provisioner is provisioning volume for claim "default/demo-pvc"

Normal  ProvisioningSucceeded
Successfully provisioned volume pvc-b72bd49d-5c54-47c1-9467-fa45ac1063c2
```

**What did we learn?**

The event `ProvisioningSucceeded` confirms that Minikube successfully created storage dynamically.

## 10. Describe Pod

```bash
kubectl describe pod demo-pod
```

**Important output from our lab:**

```text
Name:             demo-pod
Namespace:        default
Node:             minikube/192.168.49.2
Status:           Running
IP:               10.244.0.9

Containers:
  app:
    Image:         busybox

    State:          Running
    Ready:          True

    Mounts:
      /data from storage (rw)

Volumes:
  storage:
    Type:       PersistentVolumeClaim
    ClaimName:  demo-pvc
    ReadOnly:   false
```

**Important observation:**

```text
/data from storage (rw)
```

This confirms that our PVC is mounted inside the container at `/data`.

---

# PART 6: Test Persistent Data

## 11. Enter the Container

```bash
kubectl exec -it demo-pod -- sh
```

We enter the BusyBox container.

Expected prompt:

```text
/ #
```

## 12. Create a File Inside the Mounted Directory

Execute:

```bash
echo "Hello From pv" > /data/rakesh.txt
```

The file is created inside the persistent storage mount.

## 13. Verify the File

```bash
cat /data/rakesh.txt
```

**Output:**

```text
Hello From pv
```

### Important Troubleshooting Point

Initially, we executed:

```bash
echo "hello from pv" > /data/hello.txt
```

But tried reading it using:

```bash
cat hello.txt
```

We received:

```text
cat: can't open 'hello.txt': No such file or directory
```

**Why?**

Our current directory was `/`, but the file was created inside `/data`.

Correct command:

```bash
cat /data/hello.txt
```

Always use the correct file path.

## 14. Exit the Container

```bash
exit
```

---

# PART 7: Delete and Recreate Pod

## 15. Delete Existing Pod

```bash
kubectl delete pod demo-pod
```

**Output:**

```text
pod "demo-pod" deleted from default namespace
```

Important: We are deleting only the Pod, not the PVC.

## 16. Recreate Pod

```bash
kubectl apply -f my-pod.yaml
```

**Output:**

```text
pod/demo-pod created
```

## 17. Verify Previously Created Data

Now execute:

```bash
kubectl exec -it demo-pod -- cat /data/rakesh.txt
```

**Actual output from our lab:**

```text
Hello From pv
```

Excellent!

The previously created file is still available after deleting and recreating the Pod.

**This proves that our application data is stored in persistent storage rather than only in the container's writable filesystem.**

---

# PART 8: Understand Static vs Dynamic Provisioning

## 18. Static Provisioning

In static provisioning:

1. Administrator manually creates a PV.
2. User creates a PVC.
3. Kubernetes matches the PVC with an available PV.
4. Pod uses the PVC.

Architecture:

```text
Administrator
     |
     v
Creates PV
     |
     v
User creates PVC
     |
     v
PVC binds to existing PV
     |
     v
Pod uses PVC
```

## 19. Dynamic Provisioning

In dynamic provisioning:

1. User creates a PVC.
2. PVC references a StorageClass.
3. StorageClass provisioner automatically creates a PV.
4. PVC binds to the newly created PV.
5. Pod uses the PVC.

Architecture:

```text
User creates PVC
       |
       v
  StorageClass
       |
       v
Dynamic Provisioner
       |
       v
Automatically creates PV
       |
       v
PVC becomes Bound
       |
       v
Pod uses PVC
```

### Difference Between Static and Dynamic Provisioning

| Static Provisioning | Dynamic Provisioning |
|---|---|
| PV created manually | PV created automatically |
| Administrator prepares storage | Provisioner creates storage |
| PVC claims existing PV | PVC triggers provisioning |
| Requires available matching PV | Requires a working StorageClass and provisioner |

---

# PART 9: Important Kubernetes Commands

| Command | Purpose |
|---|---|
| `kubectl get nodes` | Check Kubernetes nodes |
| `kubectl get pods` | List Pods |
| `kubectl get pv` | List PersistentVolumes |
| `kubectl get pvc` | List PersistentVolumeClaims |
| `kubectl describe pv demo-pv` | Detailed PV information |
| `kubectl describe pvc demo-pvc` | Detailed PVC information |
| `kubectl describe pod demo-pod` | Detailed Pod information |
| `kubectl apply -f my-pv.yaml` | Create PV |
| `kubectl apply -f my-pvc.yaml` | Create PVC |
| `kubectl apply -f my-pod.yaml` | Create Pod |
| `kubectl exec -it demo-pod -- sh` | Enter container |
| `kubectl delete pod demo-pod` | Delete Pod |

---

# PART 10: Important Interview Questions

### Q1. What is PersistentVolume in Kubernetes?

PersistentVolume is a storage resource available in a Kubernetes cluster. It provides storage to applications.

### Q2. What is PersistentVolumeClaim?

PVC is a request for storage made by a user or application.

### Q3. What is the difference between PV and PVC?

PV represents available storage, whereas PVC requests storage from Kubernetes.

### Q4. Can a Pod directly use a PV?

No. A Pod normally references a PVC, and the PVC is bound to a PV.

### Q5. What happens when a Pod is deleted?

The Pod is deleted, but data stored in persistent storage remains available as long as the underlying storage and PVC are retained.

### Q6. What is Dynamic Provisioning?

Dynamic provisioning automatically creates storage when a PVC requests storage through a StorageClass.

### Q7. What is Static Provisioning?

Static provisioning means an administrator manually creates PV resources before users request storage through PVCs.

### Q8. What is StorageClass?

StorageClass provides a way to define storage provisioning configurations in Kubernetes.

### Q9. What is ReadWriteOnce?

ReadWriteOnce (RWO) allows a volume to be mounted as read-write by a single node at a time. Multiple Pods on that same node may be able to use it, depending on the storage driver.

### Q10. What is the purpose of volumeMounts?

VolumeMounts defines where a volume is mounted inside a container.

### Q11. What is the difference between emptyDir and PVC?

`emptyDir` is temporary storage associated with a Pod's lifecycle. PVC provides access to separately provisioned persistent storage.

### Q12. What is the purpose of reclaimPolicy?

It defines what happens to the PV's underlying storage when its PVC is released.

Common policies:

- Retain
- Delete

### Q13. What is hostPath?

hostPath mounts a directory or file from the Kubernetes node into a Pod.

It is useful for local learning environments but is generally not recommended as an application storage solution in production clusters.

---

# PART 11: Important Production Notes

1. Do not use hostPath as a general production storage solution.
2. Use appropriate storage backends such as AWS EBS, EFS or supported CSI storage solutions.
3. Understand StorageClass configuration before creating PVCs.
4. Always verify PV and PVC binding.
5. Do not delete PVCs without understanding their reclaim policy.
6. Configure backups for important application data.
7. Persistent storage does not automatically provide backup or disaster recovery.

**Important:** Our lab demonstrated Pod deletion and recreation on the same Minikube node. It does not prove that data will survive node or disk failure.

---

# PART 12: Final Lab Completion Checklist

- [x] Checked Kubernetes nodes.
- [x] Checked existing Pods.
- [x] Created PersistentVolume YAML.
- [x] Created PersistentVolume successfully.
- [x] Created PersistentVolumeClaim YAML.
- [x] Created PVC successfully.
- [x] Understood dynamic provisioning.
- [x] Created Pod using BusyBox.
- [x] Mounted PVC at `/data`.
- [x] Verified PV and PVC.
- [x] Described PVC and Pod.
- [x] Created a test file inside `/data`.
- [x] Deleted the Pod.
- [x] Recreated the Pod.
- [x] Verified previously stored data.
- [x] Understood static and dynamic provisioning.

**Lab Status: COMPLETED SUCCESSFULLY**

---

## Final Learning Summary

In this lab, we learned how Kubernetes persistent storage works.

We created a PersistentVolume, PersistentVolumeClaim and Pod. We mounted storage inside the container using `volumeMounts`.

We also observed how Minikube dynamically provisions a new PV using the default StorageClass.

Finally, we created a file inside the mounted directory, deleted the Pod, recreated it and successfully accessed the same file.

**Final verification:**

```bash
kubectl exec -it demo-pod -- cat /data/rakesh.txt
```

Output:

```text
Hello From pv
```

This confirms successful data persistence across Pod recreation.

**Thank you! Happy Learning – Kubernetes & DevOps.**
