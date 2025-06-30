
VM creation:
============
 Prerequisites
* OpenShift Virtualization (KubeVirt) installed.
* ODF (Ceph) running with a StorageClass like ocs-storagecluster-ceph-rbd.
* virtctl installed on your machine.

Step-by-Step Instructions

1️ Create Namespace
```oc new-project windows-vms```

2️ Upload Windows ISO as PVC
```virtctl image-upload pvc windows-server-iso \
  --image-path /path/to/WindowsServer.iso \
  --namespace windows-vms \
  --access-mode ReadWriteOnce \
  --storage-class ocs-storagecluster-ceph-rbd \
  --size 10Gi \
  --insecure```

3️ Upload VirtIO Driver ISO as PVC
Download virtio-win.iso from https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/
```virtctl image-upload pvc virtio-drivers \
  --image-path /path/to/virtio-win.iso \
  --namespace windows-vms \
  --access-mode ReadWriteOnce \
  --storage-class ocs-storagecluster-ceph-rbd \
  --size 1Gi \
  --insecure```

4️ Create Blank Root Disk as DataVolume (refer windows-rootdisk-dv.yaml)

5 create VM using windows-vm.yaml. Ensure iso,virtIO and root datadisk volume PVC names are correct. Apply it (oc apply -f windows-vm.yaml)

6 Start the VM 
```virtctl start windows-vm -n windows-vms```
7 Install Windows
  * Use OpenShift Console → Virtualization → windows-vm → Console.
  * At disk selection screen:
    * Click "Load driver".
    * Navigate to: E:\viostor\2k22\amd64
    *  (or 2k19/win10 depending on ISO version)
    * Select the .inf file and load the driver.
    * Disk should now be detected.
  * Install NW driver from “E:\NetKVM\2k22\amd64” 
  * Proceed with installation.
8 Optional Cleanup After Install
  Once Windows is installed:
  * Shut down VM: '''virtctl stop windows-vm -n windows-vms'''
  * Edit VM YAML to remove:
      * cdrom (Windows ISO)
      * virtio (driver ISO)
  This prevents re-attaching on every boot.
9  Steps to Reinstall(in case of any error):
     Stop the VM: ```virtctl stop windows-vm -n windows-vms```
     Delete the current root disk (if okay to lose current install): oc delete dv win-rootdisk -n windows-vms
     Recreate the root disk using the same windows-rootdisk-dv.yaml: oc apply -f windows-rootdisk-dv.yaml 
     Start the VM again: virtctl start windows-vm -n windows-vms



VM clone from Template:
=======================
1. Create a template:
oc apply -f mssql-benchmark-template.yaml

2. Check template:
oc get templates -n windows-vms
3. Create a clone of a PVC (root volume) from a OC console or through YAML

```cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mssql-vm-1-rootdisk
  namespace: windows-vms
spec:
  storageClassName: ocs-storagecluster-ceph-rbd
  dataSource:
    name: win2-rootdisk
    kind: PersistentVolumeClaim
    apiGroup: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
EOF```


4 .Process the template:
```oc process -n windows-vms mssql-benchmark-template-pvc \
  -p NAME=mssql-vm-1 \
  -p NAMESPACE=windows-vms \
  -p PVC_NAME=mssql-vm-1-rootdisk | oc apply -f -```
4. Start the VM
```oc start vm mssql-vm-1 -n windows-vms```
