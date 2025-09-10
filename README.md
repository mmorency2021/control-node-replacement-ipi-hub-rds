# control-node-replacement-ipi-hub-rds

### Purpose 

The purpose of this procedure is to show how to replace a failed Control Plane node in a Baremetal Openshift cluster (3+0 or 3+N ) in a simple way.
This methodology is based on IPI and will allow you to replace a failed Control node quickly.

### Prerequisites 

* Existing Baremetal cluster installed either with [IPI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/installing_on_bare_metal/installer-provisioned-infrastructure) or [ABI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/installing_an_on-premise_cluster_with_the_agent-based_installer/index) or [Asisted Installer](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/installing_on-premise_with_assisted_installer/index) or [UPI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/installing_on_bare_metal/user-provisioned-infrastructure) using OCP >= 4.19.1
* Ensure that all the required DNS records exist 
* You have access to the cluster as a user with the `cluster-admin` role.
* You have taken an [etcd backup](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/etcd/backing-up-and-restoring-etcd-data)
* Baremetal Operator is available ($ oc get clusteroperator baremetal)
* Server boot mode set to UEFI and Redfish multimedia is supported



### Replacing a Master node
Here we will be replacing master-2 with master-3. To simulate the node faillure we will shutdown master-2.

#### Pre-check validation 

```shell
$ oc get nodes
NAME                                                   STATUS   ROLES                         AGE     VERSION
master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   4h7m    v1.32.7
master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   4h29m   v1.32.7
master-2.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   27m     v1.32.7



```

## Control Node replacement 

Please check the Official document for details of how to remove an unhealthy etcd member can be found here [Remove-Unhealth-ETCD](https://docs.openshift.com/container-platform/4.12/backup_and_restore/control_plane_backup_and_restore/replacing-unhealthy-etcd-member.html)  

### Remove Unhealthy ETCD Master-1 Member
- **Checking ETCD Member Status**
```shell
$ oc -n openshift-etcd rsh etcd-master-0 etcdctl member list -w table
+------------------+---------+----------+----------------------------+----------------------------+------------+
|        ID        | STATUS  |   NAME   |         PEER ADDRS         |        CLIENT ADDRS        | IS LEARNER |
+------------------+---------+----------+----------------------------+----------------------------+------------+
|  c300d358075445b | started | master-0 | https://192.168.24.87:2380 | https://192.168.24.87:2379 |      false |
| 1a7b6f4c3aac9be1 | started | master-1 | https://192.168.24.88:2380 | https://192.168.24.88:2379 |      false |
| 6fd2f8909c811461 | started | master-2 | https://192.168.24.86:2380 | https://192.168.24.86:2379 |      false |
+------------------+---------+----------+----------------------------+----------------------------+------------+
$ oc -n openshift-etcd rsh etcd-master-0 etcdctl endpoint health
{"level":"warn","ts":"2023-04-24T17:11:59.984Z","logger":"client","caller":"v3@v3.5.6/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc00028c000/192.168.24.86:2379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = context deadline exceeded"}
https://192.168.24.88:2379 is healthy: successfully committed proposal: took = 7.12757ms
https://192.168.24.87:2379 is healthy: successfully committed proposal: took = 7.216856ms
https://192.168.24.86:2379 is unhealthy: failed to commit proposal: context deadline exceeded
Error: unhealthy cluster
command terminated with exit code 1

```
**Note:** Take note master-2 member-ID for next steps  
- **Remove ETCD master-2 Member-ID**
```shell
$ oc -n openshift-etcd rsh etcd-master-0 
sh-4.4# etcdctl member list
c300d358075445b, started, master-0, https://192.168.24.87:2380, https://192.168.24.87:2379, false
1a7b6f4c3aac9be1, started, master-1, https://192.168.24.88:2380, https://192.168.24.88:2379, false
6fd2f8909c811461, started, master-2, https://192.168.24.86:2380, https://192.168.24.86:2379, false
sh-4.4# etcdctl member remove 6fd2f8909c811461
Member 6fd2f8909c811461 removed from cluster c413f45f7dfe9590

```
- **Check ETCD Member Status Again**  
  **Note:** Make sure master-x ETCD member is no longer shown on the status  
```shell
sh-4.4# etcdctl member list -w table
+------------------+---------+----------+----------------------------+----------------------------+------------+
|        ID        | STATUS  |   NAME   |         PEER ADDRS         |        CLIENT ADDRS        | IS LEARNER |
+------------------+---------+----------+----------------------------+----------------------------+------------+
|  c300d358075445b | started | master-0 | https://192.168.24.87:2380 | https://192.168.24.87:2379 |      false |
| 1a7b6f4c3aac9be1 | started | master-1 | https://192.168.24.88:2380 | https://192.168.24.88:2379 |      false |
+------------------+---------+----------+----------------------------+----------------------------+------------+

```
### Turn off quorum guard by entering the following command 

```shell
oc patch etcd/cluster --type=merge -p '{"spec": {"unsupportedConfigOverrides": {"useUnsupportedUnsafeNonHANonProductionUnstableEtcd": true}}}'
```

### List The Old Secrets for Unhealthy Master-2
```shell
$ oc get secret -n openshift-etcd | grep master-2
etcd-peer-master-2              kubernetes.io/tls                     2      56m
etcd-serving-master-2           kubernetes.io/tls                     2      56m
etcd-serving-metrics-master-2   kubernetes.io/tls                     2      56m

```
- **Remove the old secrets for the unhealthy etcd member that was removed**  
```shell
$ oc get secrets -n openshift-etcd|grep master-2 |awk '{print $1}'|xargs oc -n openshift-etcd delete secrets
secret "etcd-peer-master-2" deleted
secret "etcd-serving-master-2" deleted
secret "etcd-serving-metrics-master-2" deleted
```
### Check ETCD Status
```shell
$ oc get pods -n openshift-etcd | grep -v etcd-quorum-guard | grep etcd
etcd-master-0              5/5     Running     0          54m
etcd-master-2               2/5     NotReady    0          50m
etcd-master-1               5/5     Running     0          52m
```

### Delete Machine and BMH of the failed Master
```shell
$ oc delete machine master-2 -n n openshift-machine-api
$ oc delete bmh master-2 -n openshift-machine-api
```
### Prepare to Delete master-2 Node
- **Check PODs status on Master-2 Node**
```shell
$ oc get po -A -o wide|grep master-2'
```  
It should be PODs still allocated to master-2, then follow next steps to clean them up.
- **Delete Master-2 Node**
```shell
$ oc delete node master-2
```
**Note:** Please check this status again to make sure no more PODs allocated / running on master-2 anymore.    
And also make sure that no more pods on master-2
```shell
$ oc get po -o wide -A|grep master-2|wc -l
0
```

```shell
$ oc get nodes
NAME       STATUS   ROLES                         AGE   VERSION
master-0   Ready    control-plane,master,worker   47m   v1.25.8+27e744f
master-1   Ready    control-plane,master,worker   75m   v1.25.8+27e744f



```
Now we are ready to add the new control node 


#### Preparing the bare metal node 

To replace the failed master node, you can used either static or Dynamic IP configuration. When replacing a master node using a DHCP server, the node must have a DHCP reservation.

1- Make sure the node is poweroff ( new master)

2- Validate that you have the OC version that match the cluster version

3- Retrieve the user name and password of the bare metal node’s baseboard management controller. Then, create base64 strings from the user name and password:

* echo -ne "root" | base64
* echo -ne "password" | base64


####  Create a configuration file for the bare metal node**
```shell
$  cat <<EOF | oc apply -f -
---
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
apiVersion: v1
kind: Secret
metadata:
  name: master-3-bmc
  namespace: openshift-machine-api
type: Opaque
data:
  username: cm9vdAo=  
  password: Y2FsdmluCg==
---
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
    username: ddddd
    password: xdddd 
  rootDeviceHints:
    deviceName: /dev/disk/by-path/pci-0000:18:00.0-scsi-0:2:1:0 
  preprovisioningNetworkDataName: master-3
  customDeploy:
    method: install_coreos
  userData:
    name: master-user-data-managed
    namespace: openshift-machine-api
```


Bmh object get created and will transition to different status 
* Registering 
* Inspecting  
*  Provisioning
* Provisionned

```shell
$ oc get bmh 
NAME                                                   STATE         CONSUMER                ONLINE   ERROR   AGE
master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   unmanaged     mno-cu-8wdjc-master-0   true             4h45m
master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   unmanaged     mno-cu-8wdjc-master-1   true             4h45m
master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Inspecting      true             74m

```
Keep monitoring the bmh until status changed to “Provisionned”. In the meantime , Server will get booted using Virtual Media to install RHCOS.

```shell
$ oc get bmh 
NAME                                                   STATE         CONSUMER                ONLINE   ERROR   AGE
master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   unmanaged     mno-cu-8wdjc-master-0   true             4h45m
master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   unmanaged     mno-cu-8wdjc-master-1   true             4h45m
master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   provisioned     true             74m
```

once BMH is in "provisionned" status , check for csr and approve them

```shell
oc get csr
oc get csr
NAME        AGE   SIGNERNAME                                    REQUESTOR                                                                   REQUESTEDDURATION   CONDITION
csr-c9gcr   39m   kubernetes.io/kube-apiserver-client           system:node:master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com            24h                 Approved,Issued
csr-gxdkl   41m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   <none>              Approved,Issued
csr-pdd7g   39m   kubernetes.io/kube-apiserver-client           system:node:master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com            24h                 Approved,Issued
csr-q2fv5   40m   kubernetes.io/kubelet-serving                 system:node:master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com            <none>              Approved,Issued
```


Create the machine CR 
```shell
$  cat <<EOF | oc apply -f -
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

```

BMH should be linked with new machine 

```shell
$ oc get bmh 
```
```shell
NAME                                                   STATE         CONSUMER                ONLINE   ERROR   AGE
master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   unmanaged     mno-cu-8wdjc-master-0   true             4h50m
master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   unmanaged     mno-cu-8wdjc-master-1   true             4h50m
master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   provisioned   mno-cu-8wdjc-master-3   true             79m
```
```shell
$ oc get machine
NAME                    PHASE     TYPE   REGION   ZONE   AGE
mno-cu-8wdjc-master-0   Running                          4h51m
mno-cu-8wdjc-master-1   Running                          4h51m
mno-cu-8wdjc-master-3   Running                          39m
```


new node should be ready by now 

```shell

$ oc get nodes
NAME                                                   STATUS   ROLES                         AGE     VERSION
master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   4h24m   v1.32.7
master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   4h46m   v1.32.7
master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   44m     v1.32.7
```

### turn the quorum guard back on 

```shell
oc patch etcd/cluster --type=merge -p '{"spec": {"unsupportedConfigOverrides": null}}'
```


#### Validation

 

Validate that all nodes are ready and that cluster is stable  
```shell
$ oc get nodes
NAME                                                   STATUS   ROLES                         AGE     VERSION
master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   4h24m   v1.32.7
master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   4h46m   v1.32.7
master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   44m     v1.32.7

```
```shell
[root@rack1-jumphost 20250910-13:48:45]$ oc get machine
NAME                    PHASE     TYPE   REGION   ZONE   AGE
mno-cu-8wdjc-master-0   Running                          4h53m
mno-cu-8wdjc-master-1   Running                          4h53m
mno-cu-8wdjc-master-3   Running                          41m
$ oc get bmh 
NAME                                                   STATE         CONSUMER                ONLINE   ERROR   AGE
master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   unmanaged     mno-cu-8wdjc-master-0   true             4h53m
master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   unmanaged     mno-cu-8wdjc-master-1   true             4h53m
master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   provisioned   mno-cu-8wdjc-master-3   true             82m
```


```shell

$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.19.10   True        False         False      4h24m   
baremetal                                  4.19.10   True        False         False      4h42m   
cloud-controller-manager                   4.19.10   True        False         False      4h47m   
cloud-credential                           4.19.10   True        False         False      4h53m   
cluster-autoscaler                         4.19.10   True        False         False      4h42m   
config-operator                            4.19.10   True        False         False      4h43m   
console                                    4.19.10   True        False         False      4h30m   
control-plane-machine-set                  4.19.10   True        False         False      4h42m   
csi-snapshot-controller                    4.19.10   True        False         False      4h43m   
dns                                        4.19.10   True        False         False      4h42m   
etcd                                       4.19.10   True        False         False      4h40m   
image-registry                             4.19.10   True        False         False      4h30m   
ingress                                    4.19.10   True        False         False      4h33m   
insights                                   4.19.10   True        False         False      4h43m   
kube-apiserver                             4.19.10   True        False         False      4h38m   
kube-controller-manager                    4.19.10   True        False         False      4h38m   
kube-scheduler                             4.19.10   True        False         False      4h40m   
kube-storage-version-migrator              4.19.10   True        False         False      4h43m   
machine-api                                4.19.10   True        False         False      4h38m   
machine-approver                           4.19.10   True        False         False      4h43m   
machine-config                             4.19.10   True        False         False      4h42m   
marketplace                                4.19.10   True        False         False      4h42m   
monitoring                                 4.19.10   True        False         False      4h27m   
network                                    4.19.10   True        False         False      4h43m   
node-tuning                                4.19.10   True        False         False      45m     
olm                                        4.19.10   True        False         False      4h42m   
openshift-apiserver                        4.19.10   True        False         False      4h33m   
openshift-controller-manager               4.19.10   True        False         False      4h39m   
openshift-samples                          4.19.10   True        False         False      4h33m   
operator-lifecycle-manager                 4.19.10   True        False         False      4h42m   
operator-lifecycle-manager-catalog         4.19.10   True        False         False      4h42m   
operator-lifecycle-manager-packageserver   4.19.10   True        False         False      169m    
service-ca                                 4.19.10   True        False         False      4h43m   
storage                                    4.19.10   True        False         False      4h43m     

$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.19.10   True        False         4h26m   Cluster version is 4.19.10

```
- **Checking ETCD Member Status**
```shell
$ oc get pods -n openshift-etcd
NAME                                                                      READY   STATUS      RESTARTS   AGE
etcd-guard-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com           1/1     Running     0          4h27m
etcd-guard-master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com           1/1     Running     0          4h44m
etcd-guard-master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com           1/1     Running     0          46m
etcd-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com                 5/5     Running     0          41m
etcd-master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com                 5/5     Running     0          39m
etcd-master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com                 5/5     Running     0          37m
installer-11-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com         0/1     Completed   0          126m
installer-11-master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com         0/1     Completed   0          119m
installer-13-master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com         0/1     Completed   0          47m
installer-15-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com         0/1     Completed   0          43m
installer-15-master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com         0/1     Completed   0          40m
installer-15-master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com         0/1     Completed   0          38m
revision-pruner-11-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          126m
revision-pruner-11-master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          126m
revision-pruner-11-master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          47m
revision-pruner-12-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          47m
revision-pruner-12-master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          47m
revision-pruner-12-master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          47m
revision-pruner-13-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          47m
revision-pruner-13-master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          47m
revision-pruner-13-master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          47m
revision-pruner-14-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          43m
revision-pruner-14-master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          43m
revision-pruner-14-master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          43m
revision-pruner-15-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          43m
revision-pruner-15-master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          43m
revision-pruner-15-master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com   0/1     Completed   0          43m

```

```shell

$ oc -n openshift-etcd rsh etcd-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com  etcdctl member list -w table
+------------------+---------+------------------------------------------------------+----------------------------+----------------------------+------------+
|        ID        | STATUS  |                         NAME                         |         PEER ADDRS         |        CLIENT ADDRS        | IS LEARNER |
+------------------+---------+------------------------------------------------------+----------------------------+----------------------------+------------+
| 3609a04589b18117 | started | master-3.mno-cu.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.89:2380 | https://192.168.24.89:2379 |      false |
| 7ba70a5da2a758f8 | started | master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.87:2380 | https://192.168.24.87:2379 |      false |
| e9dca6063087aaf5 | started | master-1.mno-cu.hubcluster-1.lab.eng.cert.redhat.com | https://192.168.24.88:2380 | https://192.168.24.88:2379 |      false |
+------------------+---------+------------------------------------------------------+----------------------------+----------------------------+------------+

```
```shell
$ oc -n openshift-etcd rsh etcd-master-0.mno-cu.hubcluster-1.lab.eng.cert.redhat.com  etcdctl endpoint health
https://192.168.24.88:2379 is healthy: successfully committed proposal: took = 7.967754ms
https://192.168.24.87:2379 is healthy: successfully committed proposal: took = 8.308346ms
https://192.168.24.89:2379 is healthy: successfully committed proposal: took = 11.115965ms

```







