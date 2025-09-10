# 🔧 Control Node Replacement - IPI Hub RDS

<div align="center">

![OpenShift](https://img.shields.io/badge/OpenShift-4.19+-red?style=for-the-badge&logo=redhat)
![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.32.7-blue?style=for-the-badge&logo=kubernetes)
![Status](https://img.shields.io/badge/Status-Production_Ready-green?style=for-the-badge)

</div>

---

## 📋 Procedure Overview Table

| Step | Section | Procedure | Time Estimate | Risk Level | Prerequisites |
|------|---------|-----------|---------------|------------|---------------|
| **1** | **🔍 Pre-check** | Validate current cluster state | 5 min | 🟢 Low | Admin access |
| **2** | **🔴 ETCD Removal** | Check ETCD member status | 5 min | 🟡 Medium | ETCD access |
| **3** | **🗑️ Member Cleanup** | Remove unhealthy ETCD member | 10 min | 🔴 High | Member ID noted |
| **4** | **🔒 Quorum Guard** | Disable quorum protection | 2 min | 🔴 High | Cluster stable |
| **5** | **🔐 Secret Cleanup** | Remove old node secrets | 5 min | 🟡 Medium | Secret list |
| **6** | **🏗️ Infrastructure** | Delete machine and BMH | 5 min | 🟡 Medium | Resource names |
| **7** | **🧹 Node Cleanup** | Remove failed node | 10 min | 🟡 Medium | Pod migration |
| **8** | **🔧 BMC Prep** | Prepare new bare metal node | 15 min | 🟢 Low | BMC credentials |
| **9** | **🛠️ Node Config** | Create node configuration | 10 min | 🟡 Medium | Network details |
| **10** | **📊 BMH Monitor** | Monitor BMH provisioning | 30-60 min | 🟢 Low | Virtual media |
| **11** | **📜 CSR Approval** | Approve certificate requests | 5 min | 🟢 Low | CSR monitoring |
| **12** | **🤖 Machine Link** | Create machine resource | 5 min | 🟡 Medium | BMH ready |
| **13** | **🔗 Verification** | Verify BMH-machine linking | 5 min | 🟢 Low | Resources created |
| **14** | **✅ Node Status** | Verify node ready status | 10 min | 🟢 Low | Node joined |
| **15** | **🔒 Quorum Guard** | Re-enable quorum protection | 2 min | 🟡 Medium | Node stable |
| **16** | **🧪 Final Validation** | Complete system health check | 15 min | 🟢 Low | All components |

### 📊 Procedure Summary

| **Category** | **Details** |
|--------------|-------------|
| **🎯 Total Time** | ~2-3 hours (including provisioning) |
| **🔧 Active Time** | ~1 hour (excluding BMH provisioning) |
| **⚠️ Downtime** | Minimal (cluster remains operational) |
| **🛡️ Risk Level** | Medium (with proper backup) |
| **👥 Required Personnel** | 1 OpenShift Administrator |
| **📚 Skill Level** | Intermediate to Advanced |

### 🚨 Critical Checkpoints

| **Checkpoint** | **Validation** | **Rollback Point** |
|----------------|----------------|-------------------|
| **ETCD Health** | All endpoints responding before removal | Stop if < 2 healthy members |
| **Quorum Status** | Verify quorum guard disabled | Re-enable if issues |
| **BMH Provisioning** | Node boots and installs RHCOS | Check BMC/network config |
| **Node Join** | Node appears in cluster | Verify CSRs approved |
| **ETCD Recovery** | 3-member cluster restored | Check member list |
| **Cluster Operators** | All operators healthy | Monitor for degradation |

---

## 🎯 Purpose 

> **📋 Objective:** This procedure demonstrates how to replace a failed Control Plane node in a Baremetal OpenShift cluster (3+0 or 3+N) using a streamlined IPI-based methodology.

This approach leverages **Infrastructure Provider Installation (IPI)** to enable rapid replacement of failed Control nodes with minimal downtime.

---

## ⚡ Prerequisites 

### 🔧 System Requirements

| Requirement | Description | Status |
|-------------|-------------|---------|
| **🏗️ Existing Cluster** | Baremetal cluster installed with [IPI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/installing_on_bare_metal/installer-provisioned-infrastructure), [ABI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/installing_an_on-premise_cluster_with_the_agent-based_installer/index), [Assisted Installer](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/installing_on-premise_with_assisted_installer/index), or [UPI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/installing_on_bare_metal/user-provisioned-infrastructure) | ✅ Required |
| **🌐 DNS Records** | All required DNS records configured | ✅ Required |
| **👤 Admin Access** | `cluster-admin` role access | ✅ Required |
| **💾 ETCD Backup** | Recent [etcd backup](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/etcd/backing-up-and-restoring-etcd-data) taken | ✅ Required |
| **🤖 Baremetal Operator** | Available and running | ✅ Required |
| **⚙️ Server Configuration** | UEFI boot mode + Redfish multimedia support | ✅ Required |
| **📦 OCP Version** | OpenShift >= 4.19.1 | ✅ Required |

---

## 🚀 Replacing a Master Node

> **🎯 Scenario:** Replacing `master-2` with `master-3` (simulating node failure by shutting down `master-2`)

### 🔍 Pre-check Validation 

```bash
# 📊 Check current node status
$ oc get nodes
```

**Expected Output:**
```
NAME                                                   STATUS   ROLES                         AGE     VERSION
master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   4h7m    v1.32.7
master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   4h29m   v1.32.7
master-2.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   27m     v1.32.7
```

---

## 🛠️ Control Node Replacement Process

> **📚 Reference:** For detailed etcd member removal procedures, see [Remove Unhealthy ETCD](https://docs.openshift.com/container-platform/4.12/backup_and_restore/control_plane_backup_and_restore/replacing-unhealthy-etcd-member.html)

### 🔴 Step 1: Remove Unhealthy ETCD Member (master-2)

#### 📈 Check ETCD Member Status

```bash
# 🔍 List all ETCD members
$ oc -n openshift-etcd rsh etcd-master-0 etcdctl member list -w table
```

**Output:**
```
+------------------+---------+----------+----------------------------+----------------------------+------------+
|        ID        | STATUS  |   NAME   |         PEER ADDRS         |        CLIENT ADDRS        | IS LEARNER |
+------------------+---------+----------+----------------------------+----------------------------+------------+
|  c300d358075445b | started | master-0 | https://192.168.24.87:2380 | https://192.168.24.87:2379 |      false |
| 1a7b6f4c3aac9be1 | started | master-1 | https://192.168.24.88:2380 | https://192.168.24.88:2379 |      false |
| 6fd2f8909c811461 | started | master-2 | https://192.168.24.86:2380 | https://192.168.24.86:2379 |      false |
+------------------+---------+----------+----------------------------+----------------------------+------------+
```

#### 🏥 Check ETCD Health Status

```bash
# 💓 Verify ETCD endpoint health
$ oc -n openshift-etcd rsh etcd-master-0 etcdctl endpoint health
```

**Expected Output (with failed node):**
```
{"level":"warn","ts":"2023-04-24T17:11:59.984Z","logger":"client","caller":"v3@v3.5.6/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc00028c000/192.168.24.86:2379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = context deadline exceeded"}
https://192.168.24.88:2379 is healthy: successfully committed proposal: took = 7.12757ms
https://192.168.24.87:2379 is healthy: successfully committed proposal: took = 7.216856ms
https://192.168.24.86:2379 is unhealthy: failed to commit proposal: context deadline exceeded
Error: unhealthy cluster
```

> **⚠️ Important:** Note the `master-2` member-ID (`6fd2f8909c811461`) for the next step

#### 🗑️ Remove ETCD master-2 Member

```bash
# 🔧 Connect to ETCD pod and remove failed member
$ oc -n openshift-etcd rsh etcd-master-0 
sh-4.4# etcdctl member list
sh-4.4# etcdctl member remove 6fd2f8909c811461
```

**Output:**
```
Member 6fd2f8909c811461 removed from cluster c413f45f7dfe9590
```

#### ✅ Verify Member Removal

```bash
# 📋 Confirm master-2 is no longer listed
sh-4.4# etcdctl member list -w table
```

**Expected Output:**
```
+------------------+---------+----------+----------------------------+----------------------------+------------+
|        ID        | STATUS  |   NAME   |         PEER ADDRS         |        CLIENT ADDRS        | IS LEARNER |
+------------------+---------+----------+----------------------------+----------------------------+------------+
|  c300d358075445b | started | master-0 | https://192.168.24.87:2380 | https://192.168.24.87:2379 |      false |
| 1a7b6f4c3aac9be1 | started | master-1 | https://192.168.24.88:2380 | https://192.168.24.88:2379 |      false |
+------------------+---------+----------+----------------------------+----------------------------+------------+
```

---

### 🔒 Step 2: Disable Quorum Guard

> **⚠️ Critical:** This temporarily disables etcd quorum protection during replacement

```bash
# 🛡️ Disable quorum guard for maintenance
oc patch etcd/cluster --type=merge -p '{"spec": {"unsupportedConfigOverrides": {"useUnsupportedUnsafeNonHANonProductionUnstableEtcd": true}}}'
```

---

### 🔐 Step 3: Remove Old Secrets

#### 📋 List Existing Secrets for master-2

```bash
# 🔍 Find all secrets related to master-2
$ oc get secret -n openshift-etcd | grep master-2
```

**Output:**
```
etcd-peer-master-2              kubernetes.io/tls                     2      56m
etcd-serving-master-2           kubernetes.io/tls                     2      56m
etcd-serving-metrics-master-2   kubernetes.io/tls                     2      56m
```

#### 🗑️ Delete Old Secrets

```bash
# 🧹 Clean up old certificates and secrets
$ oc get secrets -n openshift-etcd|grep master-2 |awk '{print $1}'|xargs oc -n openshift-etcd delete secrets
```

**Output:**
```
secret "etcd-peer-master-2" deleted
secret "etcd-serving-master-2" deleted
secret "etcd-serving-metrics-master-2" deleted
```

---

### 🏗️ Step 4: Remove Failed Infrastructure

#### 📊 Check ETCD Pod Status

```bash
# 👀 Monitor ETCD pod status
$ oc get pods -n openshift-etcd | grep -v etcd-quorum-guard | grep etcd
```

**Output:**
```
etcd-master-0              5/5     Running     0          54m
etcd-master-2               2/5     NotReady    0          50m
etcd-master-1               5/5     Running     0          52m
```

#### 🗑️ Delete Machine and BMH Resources

```bash
# 🔧 Remove machine and bare metal host objects
$ oc delete machine master-2 -n openshift-machine-api
$ oc delete bmh master-2 -n openshift-machine-api
```

#### 🧹 Clean Up Node Resources

```bash
# 👀 Check for remaining pods on master-2
$ oc get po -A -o wide|grep master-2

# 🗑️ Delete the failed node
$ oc delete node master-2

# ✅ Verify no pods remain on master-2
$ oc get po -o wide -A|grep master-2|wc -l
# Expected output: 0
```

#### 🎯 Verify Current Cluster State

```bash
# 📊 Check remaining nodes
$ oc get nodes
```

**Expected Output:**
```
NAME       STATUS   ROLES                         AGE   VERSION
master-0   Ready    control-plane,master,worker   47m   v1.25.8+27e744f
master-1   Ready    control-plane,master,worker   75m   v1.25.8+27e744f
```

---

## 🆕 Adding the New Control Node

### 🔧 Step 5: Prepare the Bare Metal Node

#### 📋 Prerequisites for New Node

> **💡 Note:** The node can use either static or DHCP IP configuration. For DHCP, ensure a reservation exists.

1. **🔌 Power State:** Ensure the new master node is powered off
2. **🔧 OC Version:** Verify `oc` client version matches cluster version  
3. **🔐 BMC Credentials:** Retrieve baseboard management controller credentials

#### 🔑 Encode BMC Credentials

```bash
# 🔐 Create base64 encoded credentials
echo -ne "root" | base64      # Username
echo -ne "password" | base64  # Password
```

---

### 🛠️ Step 6: Create Node Configuration

```yaml
# 📝 Apply the complete node configuration
cat <<EOF | oc apply -f -
---
# 🌐 Network Configuration Secret
apiVersion: v1 
kind: Secret
metadata:
 name: master-3
 namespace: openshift-machine-api
type: Opaque
stringData:
 nmstate: | 
  interfaces: 
  - name: eno1np0
    type: ethernet
    state: up
    mac-address: b8:ce:f6:56:3d:b2
    ipv4:
      address:
      - ip: 192.168.24.89
        prefix-length: 25
      enabled: true
  dns-resolver:
    config:
      server:
      - 192.168.24.80
  routes:
    config:
    - destination: 0.0.0.0/0
      next-hop-address: 192.168.24.1
      next-hop-interface: eno1np0 
---
# 🔐 BMC Credentials Secret
apiVersion: v1
kind: Secret
metadata:
  name: master-3-bmc
  namespace: openshift-machine-api
type: Opaque
data:
  username: cm9vdAo=      # base64 encoded username
  password: Y2FsdmluCg==  # base64 encoded password
---
# 🖥️ Bare Metal Host Configuration
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com 
  namespace: openshift-machine-api
spec:
  externallyProvisioned: false
  hardwareProfile: unknown
  online: True
  bootMACAddress: b8:ce:f6:56:3d:b2 
  bmc:
    address: idrac-virtualmedia://192.168.24.157/redfish/v1/Systems/System.Embedded.1 
    credentialsName: master-3-bmc
    disableCertificateVerification: True 
  rootDeviceHints:
    deviceName: /dev/disk/by-path/pci-0000:18:00.0-scsi-0:2:1:0 
  preprovisioningNetworkDataName: master-3
  customDeploy:
    method: install_coreos
  userData:
    name: master-user-data-managed
    namespace: openshift-machine-api
EOF
```

---

### 📊 Step 7: Monitor BMH Provisioning

#### 🔄 BMH State Transitions

The BMH object will progress through these states:

```mermaid
graph LR
    A[Registering] --> B[Inspecting]
    B --> C[Provisioning]
    C --> D[Provisioned]
    style A fill:#ffd700
    style B fill:#87ceeb
    style C fill:#ffa500
    style D fill:#90ee90
```

#### 👀 Monitor Progress

```bash
# 📊 Watch BMH status progression
$ oc get bmh -w
```

**Progress Monitoring:**
```
NAME                                                   STATE         CONSUMER                ONLINE   ERROR   AGE
master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   unmanaged     mno-cu-8wdjc-master-0   true             4h45m
master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   unmanaged     mno-cu-8wdjc-master-1   true             4h45m
master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Inspecting      true             74m

# ⏳ Wait for provisioned status...

master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   provisioned     true             74m
```

> **📝 Note:** The server will boot using Virtual Media to install RHCOS during the provisioning phase.

---

### 📜 Step 8: Approve Certificate Signing Requests

```bash
# 🔍 Check for pending CSRs
$ oc get csr

# 📝 Expected CSRs for new node
NAME        AGE   SIGNERNAME                                    REQUESTOR                                                                   REQUESTEDDURATION   CONDITION
csr-c9gcr   39m   kubernetes.io/kube-apiserver-client           system:node:master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com            24h                 Approved,Issued
csr-gxdkl   41m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   <none>              Approved,Issued
csr-pdd7g   39m   kubernetes.io/kube-apiserver-client           system:node:master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com            24h                 Approved,Issued
csr-q2fv5   40m   kubernetes.io/kubelet-serving                 system:node:master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com            <none>              Approved,Issued
```

---

### 🤖 Step 9: Create Machine Resource

```yaml
# 🏗️ Create the machine CR to link BMH with cluster
cat <<EOF | oc apply -f -
apiVersion: machine.openshift.io/v1beta1
kind: Machine
metadata:
  annotations:
    metal3.io/BareMetalHost: openshift-machine-api/master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com
  labels:
    machine.openshift.io/cluster-api-cluster: mno-cu-8wdjc 
    machine.openshift.io/cluster-api-machine-role: master
    machine.openshift.io/cluster-api-machine-type: master
  name: mno-cu-8wdjc-master-3
  namespace: openshift-machine-api
spec:
  metadata: {}
  providerSpec:
    value:
      apiVersion: baremetal.cluster.k8s.io/v1alpha1
      customDeploy:
        method: install_coreos
      hostSelector: {}
      image:
        checksum: ""
        url: ""
      kind: BareMetalMachineProviderSpec
      metadata:
        creationTimestamp: null
EOF
```

---

### 🔗 Step 10: Verify BMH-Machine Linking

```bash
# 🔍 Check BMH consumer linkage
$ oc get bmh 
```

**Expected Output:**
```
NAME                                                   STATE         CONSUMER                ONLINE   ERROR   AGE
master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   unmanaged     mno-cu-8wdjc-master-0   true             4h50m
master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   unmanaged     mno-cu-8wdjc-master-1   true             4h50m
master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   provisioned   mno-cu-8wdjc-master-3   true             79m
```

```bash
# 🤖 Verify machine status
$ oc get machine
```

**Expected Output:**
```
NAME                    PHASE     TYPE   REGION   ZONE   AGE
mno-cu-8wdjc-master-0   Running                          4h51m
mno-cu-8wdjc-master-1   Running                          4h51m
mno-cu-8wdjc-master-3   Running                          39m
```

---

### ✅ Step 11: Verify Node Status

```bash
# 🎯 Check new node is ready
$ oc get nodes
```

**Expected Output:**
```
NAME                                                   STATUS   ROLES                         AGE     VERSION
master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   4h24m   v1.32.7
master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   4h46m   v1.32.7
master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   44m     v1.32.7
```

---

### 🔒 Step 12: Re-enable Quorum Guard

```bash
# 🛡️ Restore etcd quorum protection
oc patch etcd/cluster --type=merge -p '{"spec": {"unsupportedConfigOverrides": null}}'
```

---

## 🧪 Final Validation

### 📊 Comprehensive System Health Check

#### 🖥️ Node Status Verification

```bash
# ✅ Verify all nodes are ready
$ oc get nodes
```

```
NAME                                                   STATUS   ROLES                         AGE     VERSION
master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   4h24m   v1.32.7
master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   4h46m   v1.32.7
master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   44m     v1.32.7
```

#### 🤖 Machine API Status

```bash
# 🔧 Check machine API objects
$ oc get machine
$ oc get bmh 
```

#### 🌐 Cluster Operator Status

```bash
# 📊 Verify all cluster operators are healthy
$ oc get co
```

**Expected Output:** All operators should show `AVAILABLE=True`, `PROGRESSING=False`, `DEGRADED=False`

```
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.19.10   True        False         False      4h24m   
baremetal                                  4.19.10   True        False         False      4h42m   
cloud-controller-manager                   4.19.10   True        False         False      4h47m   
...
```

#### 🏥 Cluster Version Status

```bash
# 📈 Verify cluster version
$ oc get clusterversion
```

```
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.19.10   True        False         4h26m   Cluster version is 4.19.10
```

#### 💓 ETCD Health Verification

```bash
# 🔍 Check ETCD pods
$ oc get pods -n openshift-etcd
```

**Key Pods to Verify:**
```
etcd-guard-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com           1/1     Running     0          4h27m
etcd-guard-master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com           1/1     Running     0          4h44m
etcd-guard-master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com           1/1     Running     0          46m
etcd-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com                 5/5     Running     0          41m
etcd-master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com                 5/5     Running     0          39m
etcd-master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com                 5/5     Running     0          37m
```

#### 🔗 ETCD Member Status

```bash
# 📋 Verify ETCD cluster membership
$ oc -n openshift-etcd rsh etcd-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com etcdctl member list -w table
```

**Expected Output:**
```
+------------------+---------+------------------------------------------------------+----------------------------+----------------------------+------------+
|        ID        | STATUS  |                         NAME                         |         PEER ADDRS         |        CLIENT ADDRS        | IS LEARNER |
+------------------+---------+------------------------------------------------------+----------------------------+----------------------------+------------+
| 3609a04589b18117 | started | master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.89:2380 | https://192.168.24.89:2379 |      false |
| 7ba70a5da2a758f8 | started | master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.87:2380 | https://192.168.24.87:2379 |      false |
| e9dca6063087aaf5 | started | master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.88:2380 | https://192.168.24.88:2379 |      false |
+------------------+---------+------------------------------------------------------+----------------------------+----------------------------+------------+
```

#### 💚 ETCD Endpoint Health

```bash
# 💓 Final health check
$ oc -n openshift-etcd rsh etcd-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com etcdctl endpoint health
```

**Expected Output:**
```
https://192.168.24.88:2379 is healthy: successfully committed proposal: took = 7.967754ms
https://192.168.24.87:2379 is healthy: successfully committed proposal: took = 8.308346ms
https://192.168.24.89:2379 is healthy: successfully committed proposal: took = 11.115965ms
```

---

## 🎉 Success Criteria

### ✅ Validation Checklist

- [ ] **🖥️ All 3 master nodes** showing `Ready` status
- [ ] **🤖 All machine objects** in `Running` phase  
- [ ] **🔧 All BMH objects** properly linked to machines
- [ ] **🌐 All cluster operators** healthy (`AVAILABLE=True`, `DEGRADED=False`)
- [ ] **💓 ETCD cluster** healthy with 3 members
- [ ] **🔗 ETCD endpoints** all responding properly
- [ ] **📈 Cluster version** stable and not progressing

### 🏆 Completion Status

> **🎯 Result:** Control plane node replacement completed successfully!
> 
> **⏱️ Downtime:** Minimal - cluster remained operational throughout the process
> 
> **🔄 Recovery:** Full 3-node control plane restored with `master-3` replacing `master-2`

---

<div align="center">

### 🔧 **Node Replacement Complete** 🔧

![Success](https://img.shields.io/badge/Replacement-Successful-brightgreen?style=for-the-badge)
![ETCD](https://img.shields.io/badge/ETCD-Healthy-green?style=for-the-badge)
![Cluster](https://img.shields.io/badge/Cluster-Stable-blue?style=for-the-badge)

**📚 Documentation maintained by the OpenShift Platform Team**

</div>
