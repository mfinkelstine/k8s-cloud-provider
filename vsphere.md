# Kubernetes Cloud Provider with vSphere configuration

- [vSphere Configuration](#vSphere-Configuration)
  - Creating and assign roles to the vSphere Cloud Provider user and vSphere entities
  - Enable UUID Attribute on all k8s VM's
    - Enable disk.EnableUUID via vSphere Web UI 
    - Enable disk.EnableUUID via linux/windows cli govc command
  - Moving all vm's k8s nodes to a default folder in vCenter
    - Creating VC directory and moving vm's 
    - Moving vm hosts & master to k8s directory
- kubernetes configuration
  - vSphere configuration file creation
  - Configuring kubelet Service , Control Manager, API Server
- Troubleshooting Configuration 
  - Unable to find VM by UUID
  - Master & Nodes labels 
- Kubernetes Storage Type
  - Persistent Volumes & Persistent Volumes Claims
  - Dynamic Provisioning and StorageClass API
  - StatefulSets


## vSphere Configuration
### Creating and assign roles to the vSphere Cloud Provider user and vSphere entities
- Enable UUID AttributeMinimal set of vSphere roles/privileges required for static only persistent volume provisioning
  
  |Roles|Privileges|Entities|Propagate to Children|
  |-----|:---------|:----------|:-----------:|
  |manage-k8s-node-vms|VirtualMachine.Config.AddExistingDisk,<br>VirtualMachine.Config.AddNewDisk,<br>VirtualMachine.Config.AddRemoveDevice,<br>VirtualMachine.Config.RemoveDisk|VM Folder|	  Yes|
  |manage-k8s-volumes|	Datastore.FileManagement (Low level file operations)|	Datastore|	No|
  |Read-only (pre-existing default role)|	System.Anonymous,<br>System.Read,<br>System.View|vCenter,<br>Datacenter,<br>Datastore Cluster,<br>Datastore Storage Folder|No|


- Minimal set of vSphere roles/privileges required for dynamic persistent volume provisioning with storage policy based volume placement
  
  |Roles|Privileges|Entities|Propagate to Children|
  |-----|:---------|:----------|:-----------:|
  |manage-k8s-node-vms|Resource.AssignVMToPool,<br>VirtualMachine.Config.AddExistingDisk, <br>VirtualMachine.Config.AddNewDisk,<br> VirtualMachine.Config.AddRemoveDevice,<br> VirtualMachine.Config.RemoveDisk,<br> VirtualMachine.Inventory.Create,<br> VirtualMachine.Inventory.Delete|Cluster, Hosts,<br>VM Folder|Yes|
  |manage-k8s-volumes|Datastore.AllocateSpace,<br>Datastore.FileManagement (Low level file operations)|Datastore|No|
  |k8s-system-read-and-spbm-profile-view|StorageProfile.View (Profile-driven storage view)|	vCenter	|No|
  |Read-only (pre-existing default role)|System.Anonymous,<br>System.Read,<br>System.View|Datacenter,<br>Datastore Cluster,<br>Datastore Storage Folder|No|


- Minimal set of vSphere roles/privileges required for dynamic volume provisioning without storage policy based volume placement

  |Roles|Privileges|Entities|Propagate to Children|
  |-----|:---------|:----------|:-----------:|
  |manage-k8s-node-vms|VirtualMachine.Config.AddExistingDisk, <br>VirtualMachine.Config.AddNewDisk, <br>VirtualMachine.Config.AddRemoveDevice, <br>VirtualMachine.Config.RemoveDisk|VM older|Yes|
  |manage-k8s-volumes|Datastore.AllocateSpace,<br>Datastore.FileManagement (Low level file operations)|Datastore|No| 
  |Read-only (pre-existing default role)|	System.Anonymous,<br> System.Read,<br> System.View	vCenter,<br>Datacenter,<br>Datastore Cluster,<br>Datastore |Storage Folder	|No|


## Enable UUID Attribute on all k8s VM's


>The disk UUID on the node VMs must be enabled: the disk.EnableUUID value must be set to True.<br>
>This step is necessary so that the VMDK always presents a consistent UUID to the VM, thus allowing the disk to be mounted properly. For each of the virtual machine nodes that will be participating in the cluster, follow the steps below using Web UI or govc Command to enabled disk.EnableUUID 
>
>Note: If your vm hosts are created from vm-template you can define the *disk.EnableUUID=1*  can be set on the template VMs


### Enable disk.EnableUUID via vSphere Web UI 
    1. Open the Host Client, and log in to the ESXi
    2. Locate the virtual machine for which you are enabling the disk UUID attribute, and power off the virtual machine.
    3. After power-off, right-click the virtual machine, and choose Edit Settings.
    4. Click VM Options tab, and select Advanced -> General.
    5. Click Edit Configuration in Configuration Parameters.
    6. Click Add parameter.
    7. In the Key column, type disk.EnableUUID.
    8. In the Value column, type TRUE.
    9. Projects Information > Kubernets: Deploying vsphere cloud-provider on existing > image2019-2-11 10:27:0.png
    10. Click OK and click Save.
    11. Power on the virtual machine
    12. Enable disk.EnableUUID via linux/windows cli govc command


### govc is a vSphere CLI built on top of govmomi.

> The CLI is designed to be a user friendly CLI alternative to the GUI and well suited for automation tasks. It also acts as a test harness for the govmomi APIs and provides working examples of how to use the APIs.

- Installing **govc** command cli from this [govc github](!https://github.com/vmware/govmomi/releases)
   ```
   curl -L https://github.com/vmware/govmomi/releases/download/v0.19.0/govc_linux_amd64.gz | gunzip > /usr/local/bin/govc
   chmod +x /usr/local/bin/govc
   ```
- Linux **govc** envirnment
  govc variable environment explanation 
  ```text
  GOVC_URL         : URL or IP of  ESXi or vCenter instance to connect to.
  GOVC_USERNAME    : USERNAME  to use if not specified in GOVC_URL.
  GOVC_PASSWORD    : PASSWORD to use if not specified in GOVC_URL.
  GOVC_INSECURE    : Disable certificate verification true.
  GOVC_DATASTORE   : Datastore name which we will work on
  GOVC_DATACENTER  : Datacenter name for our envirnemnet
  ```
  Settings Checking govc connectivity 
  ```bash
  export GOVC_URL="<VCENTER_OR_IP "
  export GOVC_USERNAME='<USER_NAME>'
  export GOVC_PASSWORD='<PASSWORD>'
  export GOVC_INSECURE=1
  export GOVC_DATASTORE="<DATASTORE_NAME"
  export GOVC_DATACENTER="<DATACENTER_NAME>"
  ```
  checking vcenter connectivity
  ```bash
  root@k8s-master:~# govc about
  Name:         VMware vCenter Server
  Vendor:       VMware, Inc.
  Version:      6.0.0
  Build:        3634793
  OS type:      win32-x64
  API type:     VirtualCenter
  API version:  6.0
  Product ID:   vpx
  UUID:         43c49851-81d9-4109-a5fb-7616617adf4a
  ```
- Setting disk.EnableUUID to true for all VMs via govc command
  ```govc vm.change -e="disk.enableUUID=1" -vm='vm_location'```

- list all k8s vms
  ```govc ls /datacenter/vm/<vm-folder-name>```

   running script to enableUUID to all k8s vm's
   ```bash
   #! /bin/sh
   
   GOVC_URL="<VCENTER_URL>"      
   GOVC_USERNAME='<USERNAME>'
   GOVC_PASSWORD='<PASSWORD>'
   GOVC_INSECURE=1 # or 0
   GOVC_DATASTORE="<WORKING_DATASTORE_NAME>"
   GOVC_DATACENTER="<WORKING_DATACENTER_NAME>"
   
   VMFOLDER="/${GOVC_DATACENTER}/vm/" 
   
   for vmNamePath in $(govc ls ${VMFOLDER}/k8s*)
   do
     govc vm.change -e="disk.enableUUID=1" -vm="$vm_name" 
   done
   ```

### Moving all vm's k8s nodes to a default folder in vCenter
> All node VMs must be placed in vSphere VM folder to Create a VM folder following the instructions bellow


- Creating VC directory and moving vm's 
  - **vCenter Web UI** : In vCenter go to the “VMs and Templates” view - right click the top level and then create a “New  Folder”  write k8s, the drag the hosts to the directory that you have created.
  - **GOVC Command** : creating k8s directory
    ```bash
    $ govc folder.create /<DATACENTER_NAME>/vm/k8s
    ```
- Moving vm hosts & master to k8s directory
  - **vCenter Web UI**: drag and drop vm's hosts to k8s
  - **GOVC Command** : Moving k8s vm to k8s directory
    ```bash 
    govc object.mv /<DATACENTER_NAME>/vm/<VM_NAMES>* /<DATACENTER_NAME>/vm/k8s
    ```


## kubernetes configuration
### vSphere configuration file creation
  * each cluster node most have vsphere.conf file
    ```bash
     vi /etc/kubernetes/vsphere.conf
    echo "
    [Global]
    user = "<username>"
    password = "<password>"
    port = "443"
    insecure-flag = "1"
    [VirtualCenter "<VC_IPADRESS OR QFND>"]
    datacenters = "<DATACENTER_NAME>"
    [Workspace]
    server = "<VC_IPADRESS OR QFND>"
    datacenter = "<DATACENTER_NAME>"
    default-datastore = "<DATASTORE_NAME>"
    resourcepool-path = "<DATACENTER_NAME>/Resources"
    folder = "k8s"
    [Disk]
    scsicontrollertype = pvscsi
    [Network]
    public-network = "VM Network"
    " > /etc/kubernetes/vsphere.conf
    ```
    example
    ```
    [Global]
    user = "<username>"
    password = "<password>"
    port = "443"
    insecure-flag = "1"
    [VirtualCenter "194.168.10.1"]
    datacenters = "TESTVC"
    [Workspace]
    server = "194.168.10.1"
    datacenter = "TESTVC"
    default-datastore = "NetFS"
    resourcepool-path = "TESTVC/Resources"
    folder = "k8s"
    [Disk]
    scsicontrollertype = pvscsi
    [Network]
    public-network = "VM Network"
    ```
### Configuring kubelet Service , Control Manager, API Server
#### Configure kubeadm file on cluster and workers nodes 
- Master Nodes
  - kubelet service configuration
    adding **--cloud-provider=vsphere**  and **--cloud-config=** to 10-kubeadm.conf file
    ```bash
    vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
    # adding this line to config file
    Environment="KUBELET_EXTRA_ARGS=--cloud-provider=vsphere --cloud-config=/etc/kubernetes/vsphere.conf"
    ```
  - Adding support vsphere 
    configuration files kube-controller-manager.yaml and kube-apiserver.yaml which located at /etc/kubernetes/manifests/
    > both files kube-controller-manager.yaml and kube-apiserver.yaml are identical structure 
    ```bash
    cd /etc/kubernetes/manifests
    ```
    ```bash
    vi kube-controller-manager.yaml kube-apiserver.yaml
    ...
    spec:
      containers:
      - command:
        - kube-apiserver 
        ...
        - --cloud-config=<CONF_PATH>/vsphere.conf
        - --cloud-provider=vsphere
        - --v=2 # extra verbose the higher number more detailes
    ```
  - add mapping point on both files kube-controller-manager.yaml & kube-apiserver.yaml
    ```bash
        ...
        volumeMounts:
        - mountPath: <CONF_PATH>/vsphere.conf
          name: cloud-config
          readOnly: true
      ...
      volumes:
      - hostPath:
          path: <CONF_PATH>/vsphere.conf
          type: FileOrCreate
        name: cloud-config
        ...
    ```
  - Restart kubelet service
    ```bash
    systemctl daemon-reload ; systemctl restart kubelet.service ; systemctl status kubelet
    ```

  - Restarting kube-apiserver & kube-controller docker files
    ```bash
    $ docker stop $(docker ps | egrep "k8s_(kube-apiserver|kube-controller)" | awk '{print $1}')
    ```
  - checking kube-apiserver and kube-controller are up and running
    ```bash
    docker ps | egrep "k8s_(kube-apiserver|kube-controller)"
    9ee83d3d9c41     214c48e87f58 "kube-apiserver --..."   4 seconds ago   Up 4 seconds k8s_kube-apiserver_kube-apiserver-k8s-master_kube-system_..._12
    be1a1859cfd7     55b70b420785 "kube-controller-m..."   5 seconds ago   Up 4 seconds k8s_kube-controller-manager_kube-controller-manager-k8s-master_kube-system_..._14
    ```
- Worker Node (all worker nodes)
  adding **--cloud-provider=vsphere** to **10-kubeadm.conf** (service file) or kubelet default file which one will catch the flag
  
  - adding vsphere cloud provider to 10-kubeadm.conf config file
    ```bash
    vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
    # adding this line
    Environment="KUBELET_EXTRA_ARGS=--cloud-provider=vsphere"
    ```

  - Adding to kubelet default file
    ```bash
    vi /etc/default/kubelet
    KUBELET_EXTRA_ARGS=--cloud-provider=vsphere
    ```
  - Restarting kubelet service 
    ```bash
    systemctl daemon-reload ; systemctl restart kubelet.service ; systemctl status kubelet
    ```
- kubernetes logs for controller-manager, apiserver and kubelet service
    - Master nodes
      - Get logs for controller-manager
        ```bash
        # view kube-controller
        kubectl logs -n kube-system  kube-controller-manager-k8s-master -f
        ```
      - Get logs for apiserver
        ```bash
        kubectl logs -n kube-system kube-apiserver-k8s-master -f
        ```
    - Worker nodes
      - get kubetlet service logs
        ```bash
        systemctl status kubelet.service
        # or 
        journalctl -xe -u kubelet -f
        ```

# Troubleshooting Configuration 
### Issue : Unable to find VM by UUID
- When setting up new/existance cluster or nodes and adding flag **cloud-provider=vsphere** and we see in kubectl logs Error: **Unable to find VM by UUID. VM UUID:** 
  like in the example 
    ```bash
    I0207 09:24:23.681213       1 disruption.go:288] Starting disruption controller
    I0207 09:24:23.681223       1 controller_utils.go:1025] Waiting for caches to sync for disruption controller
    E0207 09:24:23.682107       1 datacenter.go:78] Unable to find VM by UUID. VM UUID: 42232fc2-e8f5-c9c8-4cb5-19225600e9d2
    E0207 09:24:23.682372       1 datacenter.go:78] Unable to find VM by UUID. VM UUID: 42232fc2-e8f5-c9c8-4cb5-19225600e9d2
    ```
    - check that each node have cloud provider name
        ```bash
        ubuntu@k8s-master:~$ kubectl get nodes -o json | jq '.items[]|[.metadata.name, .spec.providerID, .status.nodeInfo.systemUUID]'
        [
         "k8s-master",
         "vsphere://42029181-1b2d-36cb-5f9f-447c461ff4ef",
         "42029181-1B2D-36CB-5F9F-447C461FF4EF"
        ]
        [
         "k8s-worker1",
         null,
         "4202384A-C7B2-D4C8-6C0E-A111F3827945"
        ]
        [
         "k8s-worker2",
         null,
         "4202ED19-DCDC-61F0-BD25-4537912337A3"
        ]
        [
         "k8s-worker3",
         null,
         "4202AD89-589C-5BA0-2EFE-52A3B63F440D"
        ]
        ```
    - fixing buy running this script 
        ```bash
        #!/bin/sh
        export KUBECONFIG=/etc/kubernetes/admin.conf
        NeedRebootList=""
        for h in $(kubectl get nodes | tail -n +2 | awk '{print $1}'); do
          uuid=$(kubectl describe node/$h | grep -i UUID | tr '[:upper:]' '[:lower:]' | awk '{print $3}')
          eval kubectl patch node $h -p \'{\"spec\":{\"providerID\":\"vsphere://${uuid}\"}}\' | grep 'no change' >/dev/null
          [[ $? -gt 0 ]] && NeedRebootList="$NeedRebootList $h"
        done
        if [[ -n $NeedRebootList ]]; then
          echo "$NeedRebootList" | tr ' ' '\n' | tail -n +2
        fi
        ### NeedRebootList holds the list of machines where there was a change and requrie reboot
        ```

    - Nodes labes check
      another issue that I have encounter is that **instance-type=vsphere-vm** label was not defined in the nodes
        ```bash
        root@k8s-master:~# kubectl get nodes --show-labels
        NAME          STATUS   ROLES    AGE   VERSION   LABELS
        k8s-master    Ready    master   26h   v1.13.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=bob,node-role.kubernetes.io/master=
        k8s-worker1   Ready    <none>   26h   v1.13.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=bob-worker-1
        ```
    * what should be expected 

        ```bashs
        NAME             STATUS    ROLES     AGE       VERSION   LABELS
        k8s-master    Ready     master    210d      v1.11.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=vsphere-vm,beta.kubernetes.io/os=linux,        kubernetes.io/hostname=k8s-master,node-role.kubernetes.io/master=
        k8s-worker1   Ready     <none>    1d        v1.11.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=pa-k8s-worker1
        k8s-worker2   Ready     <none>    206d      v1.11.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=vsphere-vm,beta.kubernetes.io/os=linux,        kubernetes.io/hostname=pa-k8s-worker5
        ```
        ```bash
        kubectl label node pa-k8s-worker1 beta.kubernetes.io/instance-type=vsphere-vm --overwrite
        ```

# Kubernetes Storage Type
- Persistent Volumes & Persistent Volumes Claims
- Dynamic Provisioning and StorageClass API
- vSphere is one of the provisioners and it allows following parameters:

diskformat which can be thin(default), zeroedthick and eagerzeroedthick

datastore is an optional field which can be VMFSDatastore or VSANDatastore. This allows user to select the datastore to provision PV from, if not specified the default datastore from vSphere config file is used.

storagePolicyName is an optional field which is the name of the SPBM policy to be applied. The newly created persistent volume will have the SPBM policy configured with it.

VSAN Storage Capability Parameters (cacheReservation, diskStripes, forceProvisioning, hostFailuresToTolerate, iopsLimit and objectSpaceReservation) are supported by vSphere provisioner for vSAN storage. The persistent volume created with these parameters will have these vSAN storage capabilities configured with it

- Creating Storage Class yaml file
```bash
echo "apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: 
  name: thin-fast
provisioner: kubernetes.io/vsphere-volume
parameters: 
  datastore: VSANDatastore
  diskformat: thin
  fstype: ext3" >> vsphere-vol-sts.yaml
```

- Creating Storage Class to kubelet
```bash
$ kubectl create -f vsphere-vol-sts.yaml
```
* Verify storage class is created
```bash
$ kubectl describe storageclass thin-fast
```

- Create Persistent Volume Claim
```bash
$ echo "
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvcsc001
  annotations:
    volume.beta.kubernetes.io/storage-class: thin-fast
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi" > vsphere-volume-pvcsc.yaml
```
- Creating the persistent volume claim

```
kubectl create -f vsphere-volume-pvcsc.yaml
```


```bash


StatefulSets
echo "
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: thin-disk
provisioner: Kubernetes.io/vsphere-volume
parameters:
    diskformat: thin
" > vsphere-vol-sts.yaml
```
- creating storage class
```
$ kubectl create -f vsphere-vol-sts.yaml
$ kubectl describe storageclass thin-disk
```
