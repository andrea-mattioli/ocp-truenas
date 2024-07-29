# HOMELAB TrueNAS Scale - Openshift Virtualization

This is not an official guide but rather a collection of steps I used to migrate the workloads from my home lab from RHV to OpenShift Virtualization.

Here is the hardware I used:

| **Specification**   | **Host 1**                                 | **Host 2**                                 | **Host 3**                                 | **Storage**                                 |
|---------------------|--------------------------------------------|--------------------------------------------|--------------------------------------------|--------------------------------------------|
| **Model**           | ThinkCentre M900                          | ThinkCentre M900                          | ThinkCentre M700                          | ProLiant MicroServer Gen10                 |
| **CPU Model**       | Intel(R) Core(TM) i7-6700T CPU @ 2.80GHz  | Intel(R) Core(TM) i7-6700T CPU @ 2.80GHz  | Intel(R) Core(TM) i7-6700T CPU @ 2.80GHz  | AMD Opteron(tm) X3216 APU                  |
| **CPU Cores**       | 8                                          | 8                                          | 8                                          | 2                                          |
| **RAM**             | 32GB                                        | 32GB                                        | 32GB                                        | 32GB                                  |
| **Disk**            | 256G                | 256G                  | 256G                  | 256G                              |


---

## Install TrueNas Scale

Follow the official guide to install TrueNAS Scale: [TrueNAS Scale Install Guide](https://www.truenas.com/docs/scale/24.04/gettingstarted/install/installingscale/)

By default, TrueNAS uses the entire disk for the operating system and does not allow its use for other purposes. However, during the installation phase, you can specify to use only 64GB + the swap partition as follows:

1. When the installer GUI shows up, choose the `[]shell` option from the 4 available.
2. We're going to adjust the installer script:
    - If you want to take a look at it beforehand, it's in this repo under "/usr/sbin/truenas-install": [TrueNAS Installer Script](https://github.com/truenas/truenas-installer)
3. To get working arrow keys and command recall, type `bash` to start a bash console:
    ```bash
    bash    
    ```
4. Find the installer script, this should yield 3 hits:
    ```bash
    find / -name truenas-install
    ```
    `/usr/sbin/truenas-install` is the one we're after.
5. Edit the script with `vi` (the only available editor):
    ```bash
    vi /usr/sbin/truenas-install
    ```
6. We are interested in the `create_partition` function, specifically in the call to create the boot-pool partition. Around line ~3xx:
    ```bash
    create_partitions()
    ...
    # Create boot pool
    if ! sgdisk -n3:0:0 -t3:BF01 /dev/${_disk}; then
        return 1
    fi
    ```
7. Move the cursor over the second `0` in `-n3:0:0` and press `x` to delete. Then press `i` to enter edit mode. Type in `+64GiB` or whatever size you want the boot pool to be. Press `esc`, then type `:wq` to save the changes:
    ```bash
    # Create boot pool
    if ! sgdisk -n3:0:+64GiB -t3:BF01 /dev/${_disk}; then
        return 1
    fi
    ```
8. You should be out of `vi` now with the install script updated. Let's run it and install TrueNAS Scale:
    ```bash
    /usr/sbin/truenas-install
    ```
9. The 'GUI' installer should be started again. Select '[]install/upgrade' this time. When prompted to select the drive(s) to install TrueNAS Scale to, select your desired SSD(s). Wait for the installation to finish and reboot.
10. Enable the SSH service:
    - Once booted, connect to the web interface. Enable SSH: `System Settings -> Services -> SSH -> Edit (add admin to Password Login Groups and flag Allow Password Authentication)`

    Or connect to the shell in `System -> Settings`.

11. Create the storage pool on the remaining space:
    - Login via SSH and escalate privilege (sudo su -), then prompt for the admin password:
    ```bash
    sudo su -
    ```
    - Figure out which disks are in the boot-pool:
    ```bash
    zpool status boot-pool
    ```

In my case, `sdb` was the system disk, so I created a new partition on the unused portion of the disk using `fdisk`. The result was:

```bash
Disk /dev/sdb: 223.57 GiB, 240057409536 bytes, 468862128 sectors
Disk model: CT240BX500SSD1  
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: 9B8425BF-8A35-4E95-9BFE-B9358334B0FE

Device         Start       End   Sectors   Size Type
/dev/sdb1       4096      6143      2048     1M BIOS boot
/dev/sdb2       6144   1054719   1048576   512M EFI System
/dev/sdb3   34609152 168826879 134217728    64G Solaris /usr & Apple ZFS
/dev/sdb4    1054720  34609151  33554432    16G Linux swap
/dev/sdb5  168826880 468860927 300034048 143.1G Linux filesystem
```

At this point, I created a zpool named "local-virt-storage" on `sdb5`:
```bash
zpool create -f local-virt-storage /dev/sdb5
```

Once created, I exported it:
```bash
zpool export local-virt-storage
```

Through the TrueNAS graphical interface, I then imported the new storage:
`Storage -> Import Pool`

![image](https://github.com/user-attachments/assets/0258e32e-c52d-4328-973a-c7278bc32ded)

I also created another zpool called `ocp-virt` on a 2TB disk to be used with the democratic-csi driver.

![image](https://github.com/user-attachments/assets/1bb8ebb5-23be-4b25-b37c-c08a8be13a13)

**Note:** When creating the zpool for the CSI driver, the label must not exceed 17 characters, including the entire driver path. For example:
ocp-virt/k8s/b/v
In this case, it is 16 characters.

---

## Installing the CSI Driver

Create the CSI user credential – `Local Users -> ADD`:
1. Add password
2. Valid login shell (bash)
3. Allowed sudo command (no passwd ALL)
![image](https://github.com/user-attachments/assets/1e6b29b0-98c7-4966-8ca7-0f16aa4ac156)

Enable the CSI user for the SSH service:
`System Settings -> Services -> SSH -> Edit (add csi to Password Login Groups and flag Allow Password Authentication)`

Enable and configure the ISCSI service:

1. Set up iSCSI
    - Go to `Sharing -> Block Shares (iSCSI)`.
    - Use the default settings in the `Target Global Configuration` tab.
2. In the `Portals` tab, click `ADD`, then create a `Description`. Set the IP Address to `0.0.0.0` and the Port to `3260`, then click `SUBMIT`.
3. In the `Initiators Groups` tab, click `ADD`. For ease of use, check the `Allow ALL Initiators`, then click `SAVE`. You can make restrictions later using the Allowed Initiators (IQN) function.
4. Create a dummy target to prevent [this issue](https://github.com/democratic-csi/democratic-csi/issues/412)
![image](https://github.com/user-attachments/assets/96989e82-87d4-4523-be7a-a425da57024c)

5. Install the driver on OpenShift using helm:   
  ```bash
  sudo curl -L https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/helm-linux-amd64 -o /usr/local/bin/helm
  sudo chmod +x /usr/local/bin/helm
  helm repo add democratic-csi https://democratic-csi.github.io/charts/
  helm repo update
  helm search repo democratic-csi/
  ```

Create the `freenas-iscsi.yaml` file:
```yaml
csiDriver:
  name: "org.democratic-csi.iscsi"

storageClasses:
  - name: freenas-iscsi-csi
    defaultClass: true
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    allowVolumeExpansion: true
    parameters:
      fsType: xfs
    mountOptions: []
    secrets:
      provisioner-secret:
      controller-publish-secret:
      node-stage-secret:
      node-publish-secret:
      controller-expand-secret:

volumeSnapshotClasses: []

driver:
  config:
    driver: freenas-iscsi
    instance_id:
    httpConnection:
      protocol: http
      host: 192.168.2.16
      port: 80
      username: admin
      password: myadminpassword #changeme
      allowInsecure: true
    sshConnection:
      host: 192.168.2.16 #changeme
      port: 22
      username: csi
      password: "mycsipassword" #changeme
    zfs:
      cli:
        sudoEnabled: true
      datasetParentName: ocp-virt/k8s/b/v
      detachedSnapshotsDatasetParentName: tanks/k8s/b/s
      zvolCompression:
      zvolDedup:
      zvolEnableReservation: false
      zvolBlocksize:
    iscsi:
      targetPortal: "192.168.2.16:3260" #changeme
      targetPortals: []
      interface:
      namePrefix: csi-
      nameSuffix: "-clustera"
      targetGroups:
        - targetGroupPortalGroup: 1
          targetGroupInitiatorGroup: 1
          targetGroupAuthType: None
      extentInsecureTpc: true
      extentXenCompat: false
      extentDisablePhysicalBlocksize: true
      extentBlocksize: 512
      extentRpm: "SSD"
      extentAvailThreshold: 0
```

If everything has worked correctly, you will find a project named `democratic-csi` in OpenShift. To test it, you can create a new project and try to create a PVC:

```bash
[amattiol@amattiol-laptop ~]$ cat pvc-test.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test
spec:
  accessModes:
  - ReadWriteMany
  volumeMode: Block
  resources:
    requests:
      storage: 1Gi
  storageClassName: freenas-iscsi-csi
```

```bash
[amattiol@amattiol-laptop ~]$ oc create -f pvc-test.yaml
persistentvolumeclaim/test created
```

```bash
[amattiol@amattiol-laptop ~]$ oc get pvc
NAME   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        VOLUMEATTRIBUTESCLASS   AGE
test   Bound    pvc-38373434-9983-4603-90d2-f0d51296fcd2   1Gi        RWX            freenas-iscsi-csi   <unset>                 88s
```

This setup verifies that the Persistent Volume Claim (PVC) was created successfully and is bound to a volume with the expected specifications.

Here's how to verify the correct removal of the Persistent Volume Claim (PVC) from OpenShift and TrueNAS:

```bash
[amattiol@amattiol-laptop ~]$ oc delete pvc test
persistentvolumeclaim "test" deleted

[amattiol@amattiol-laptop ~]$ oc get pvc
No resources found in test-pvc namespace.
```

**Note:** Make sure to check TrueNAS to confirm that the corresponding volume has been removed as well.

In TrueNAS:

1. **Access the TrueNAS Web Interface:**
   - Log in to the TrueNAS web interface.

2. **Navigate to Storage:**
   - Go to the "Storage" section.

3. **Check for Existing Pools:**
   - Ensure that the pool where the PVC was created no longer shows the volume associated with the PVC.

4. **Verify Deletion:**
   - Confirm that the storage pool or dataset used by the deleted PVC has been properly cleaned up or is no longer listed.

This process ensures that the PVC is completely removed from OpenShift and that any associated storage is correctly handled in TrueNAS.

---

## Preparation for OpenShift Virtualization

1. **Create a New Project:**
   - For example, create a project named `virtualmachines`.

2. **Install Operators:**
   - **NMState Operator:** Install with default settings.
   - **MTV Operator:** Install with default settings.

3. **Modify the Storage Profile `claimPropertySets`:**

   Update the `claimPropertySets` in the storage profile to include the following configuration:

   ```yaml
   spec:
     claimPropertySets:
       - accessModes:
           - ReadWriteMany
         volumeMode: Block
   ```

4. **Verify the Storage Profile Configuration:**

   You can check the current configuration of the storage profile by running the following command:

   ```bash
   [amattiol@amattiol-laptop ~]$ oc -n virtualmachines get storageprofile freenas-iscsi-csi -o yaml
   ```

   The output should look something like this:

   ```yaml
   apiVersion: cdi.kubevirt.io/v1beta1
   kind: StorageProfile
   metadata:
     creationTimestamp: "2024-07-27T11:01:25Z"
     generation: 12
     labels:
       app: containerized-data-importer
       app.kubernetes.io/component: storage
       app.kubernetes.io/managed-by: cdi-controller
       app.kubernetes.io/part-of: hyperconverged-cluster
       app.kubernetes.io/version: 4.16.0
       cdi.kubevirt.io: ""
     name: freenas-iscsi-csi
     ownerReferences:
     - apiVersion: cdi.kubevirt.io/v1beta1
       blockOwnerDeletion: true
       controller: true
       kind: CDI
       name: cdi-kubevirt-hyperconverged
       uid: c29d9559-d701-4e11-873a-504679fc82dd
     resourceVersion: "7852045"
     uid: 2d66c907-74f4-4492-82ed-ecdaca7d7deb
   spec:
     claimPropertySets:
     - accessModes:
       - ReadWriteMany
       volumeMode: Block
   status:
     claimPropertySets:
     - accessModes:
       - ReadWriteMany
       volumeMode: Block
     cloneStrategy: copy
     dataImportCronSourceFormat: pvc
     provisioner: org.democratic-csi.iscsi
     storageClass: freenas-iscsi-csi
   ```

   This ensures that the storage profile is properly configured to use the desired access modes and volume mode.

### Network Setup

On my nodes, I only have one network interface, `eno1`. I tried to create a Linux bridge on that interface but was unsuccessful. I also found this [Bugzilla](https://bugzilla.redhat.com/show_bug.cgi?id=1885605) report but didn't understand if it has been resolved in any way. The only alternative I found was to create a eno1.vlan interface and, consequently, a bridge on top of the VLAN interface. Below is the configuration used for the `NodeNetworkConfigurationPolicy` and the `NetworkAttachmentDefinitions`.

#### Node Network Configuration Policy

To create the `NodeNetworkConfigurationPolicy`, you used the following configuration:

```yaml
interfaces:
  - description: VLAN 13 using eno1
    name: eno1.13
    state: up
    type: vlan
    vlan:
      base-iface: eno1
      id: 13
  - description: VLAN 14 using eno1
    name: eno1.14
    state: up
    type: vlan
    vlan:
      base-iface: eno1
      id: 14
  - bridge:
      options:
        stp:
          enabled: false
      port:
        - name: eno1.13
    description: Linux bridge with eno1.13 as a port
    name: br13
    state: up
    type: linux-bridge
  - bridge:
      options:
        stp:
          enabled: false
      port:
        - name: eno1.14
    description: Linux bridge with eno1.14 as a port
    name: br14
    state: up
    type: linux-bridge
nodeSelector:
  node-role.kubernetes.io/master: ''
```

---

#### Network Attachment Definitions

I created two `NetworkAttachmentDefinitions` to associate with the bridges `br13` and `br14`. Here are the configurations for these definitions:

```yaml
apiVersion: v1
items:
- apiVersion: k8s.cni.cncf.io/v1
  kind: NetworkAttachmentDefinition
  metadata:
    creationTimestamp: "2024-07-28T17:03:00Z"
    generation: 1
    name: vlan13
    namespace: virtualmachines
    resourceVersion: "8219547"
    uid: 6f5edf72-b573-4f7a-a26d-19235ddea90c
  spec:
    config: '{"name":"vlan13","type":"cnv-bridge","cniVersion":"0.3.1","bridge":"br13","macspoofchk":false,"ipam":{}}'
- apiVersion: k8s.cni.cncf.io/v1
  kind: NetworkAttachmentDefinition
  metadata:
    creationTimestamp: "2024-07-28T20:51:42Z"
    generation: 1
    name: vlan14
    namespace: virtualmachines
    resourceVersion: "8494897"
    uid: cdce3e78-6a94-41b0-8ba4-95b77521f0ce
  spec:
    config: '{"name":"vlan14","type":"cnv-bridge","cniVersion":"0.3.1","bridge":"br14","macspoofchk":false,"ipam":{}}'
kind: List
metadata:
  resourceVersion: ""
```

These `NetworkAttachmentDefinitions` will be associated with the NICs of the VMs.

#### END Netowork Steps

1. **Verify Configuration:**
   - Ensure that the VLAN interfaces (`eno1.13`, `eno1.14`) and bridges (`br13`, `br14`) are up and properly configured on your nodes.
   
2. **Apply Network Attachment Definitions:**
   - Verify that the `NetworkAttachmentDefinitions` are correctly applied in the `virtualmachines` namespace.

3. **Check VM Connectivity:**
   - Attach these networks to your VMs and ensure they can communicate through the VLANs as expected if you have already provisioned one.


## Migration of VMs from RHV to OpenShift Virtualization

#### 1. **Create a Service Account and Assign Cluster-Admin Role**

To migrate VMs from RHV to OpenShift Virtualization, follow these steps:

1. **Create a Service Account:**
   ```bash
   oc create serviceaccount ocp-virt -n virtualmachines
   ```

2. **Generate an Access Token:**
   ```bash
   oc -n virtualmachines create token ocp-virt --duration=$((365*24))h
   ```

3. **Assign the Cluster-Admin Role to the Service Account:**
   ```bash
   oc create clusterrolebinding ocp-virt-admin-sa-crb --clusterrole=cluster-admin --serviceaccount=virtualmachines:ocp-virt
   ```

#### 2. **Configure Virtualization Providers**

1. **Access OpenShift GUI:**
   - Go to the OpenShift web console.

2. **Navigate to Virtualization Providers:**
   - Go to **Virtualization** → **Migration** → **Providers for Virtualization**.

3. **Add Providers:**
   - Add the providers for RHV (Red Hat Virtualization) and OpenShift Virtualization (OCP Virt).
   ![image](https://github.com/user-attachments/assets/762e7de3-67b8-4e10-858e-6a74d2322dfc)
   ![image](https://github.com/user-attachments/assets/8278499a-158c-4a11-a0b5-12a5902e94d1)
   
   It is possible to trust a self-signed CA by clicking on the “fetch certificate from URL” button.
   
   ![image](https://github.com/user-attachments/assets/c565545c-dd1e-48c7-a812-4caca3667f11)

   Ensure that the providers are in the "Ready" state if everything is configured correctly.

#### 3. **Create a Migration Plan**

You can create a migration plan for one or more VMs. The migration plan automatically creates the Storage Maps and Network Maps. To proceed, follow these steps:

1. Access the GUI.
2. Navigate to **Migration** → **Plans for Virtualization**.
3. Click on **Create Plan**.
4. Follow the guided procedure to complete the creation of the migration plan.
![image](https://github.com/user-attachments/assets/362eb373-f123-4f13-add2-d3c849522f45)
![image](https://github.com/user-attachments/assets/116aaf71-3dc8-4871-9d29-3c748be8a1a5)

---

Access the GUI.
Navigate to Migration → Plans for Virtualization.
Click on Create Plan.
Follow the guided procedure to complete the creation of the migration plan.

2. **Update Storage Map:**
   - Note the following for the storage map configuration:
     ```yaml
     map:
       - destination:
           accessMode: ReadWriteMany
           storageClass: freenas-iscsi-csi
     ```

---
start migration

![image](https://github.com/user-attachments/assets/12ea1b0e-b33b-4195-988c-cf92f1da1d74)

these are all the activities to migrate workloads from rhv to openshift virtualization!
happy migration!
