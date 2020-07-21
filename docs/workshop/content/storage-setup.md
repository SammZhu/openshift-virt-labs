Now that we've got the OpenShift Virtualization operator deployed, let's configure some storage for our VMs to use. 

First, switch back to the "**Terminal**" view in your lab environment.

We're going to setup two different types of storage in this section, firstly standard NFS shared storage, and secondly `hostpath` storage which uses the hypervisor's local disks, this is somewhat akin to ephemeral storage provided by OpenStack.

First, make sure you're in the default project:

~~~bash
$ oc project default
Now using project "default" on server "https://172.30.0.1:443".
~~~

>**NOTE**: If you don't use the default project for the next few lab steps, it's likely that you'll run into some errors - some resources are scoped, i.e. aligned to a namespace, and others are not. Ensuring you're in the default namespace ensures that all of the coming lab steps should flow together.

Now let's setup a storage class for NFS backed volumes, utilising the `kubernetes.io/no-provisioner` as the provisioner. 

The `kubernetes.io/no-provisioner` provisioner is generally used for local storage solutions, such as NFS. However it means that **no** dynamic provisioning of volumes is done on demand of a PVC. Instead we have to manually create our PVs for any PVC's to utilise. 

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: nfs
provisioner: kubernetes.io/no-provisioner
reclaimPolicy: Delete
EOF

storageclass.storage.k8s.io/nfs created
~~~

Let's view the newly created storage class:

~~~bash
$ oc get sc
NAME                 PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
nfs                  kubernetes.io/no-provisioner   Delete          Immediate              false                  13s
standard (default)   kubernetes.io/cinder           Delete          WaitForFirstConsumer   true                   19h
~~~

>**NOTE**: The second class there, called **standard** is for OpenStack's Cinder service and is an artefact of the RHPDS' deployment. We won't use it for these labs, as it provides only an RWO access method which is not ideal for VMs. 

Next we will create some NFS-backed physical volumes (PVs) for our VM PVC claims to utilise. However, before we do that, let's review the NFS setup we are going to use.

We have deployed a simple NFS server to the RHPDS bastion host. This is the same machine you connected to with your rhpds username to port forward for the squid. The NFS server is sharing four directories on that host, as follows:

~~~bash
[root@bastion ~]# ls -l /mnt/nfs/
total 0
drwxrwxrwx. 2 root root 6 Jul 19 23:30 four
drwxrwxrwx. 2 root root 6 Jul 19 23:30 one
drwxrwxrwx. 2 root root 6 Jul 19 23:30 three
drwxrwxrwx. 2 root root 6 Jul 19 23:30 two
~~~

We will use each one for a different PV which allows us to run four VMs at a time with NFS-backed storage.

When creating the PV's we need to tell OpenShift what IP the NFS server is on. To determine the IP for your server. On the bastion simply find the interface with the 192 address and use that:

~~~bash
[asimonel-redhat.com@bastion cnv]$ ip a | grep 192
    inet 192.168.0.26/24 brd 192.168.0.255 scope global noprefixroute dynamic eth0
~~~

This network is plumbed into bastion and all the workers. In the example above, you'd use 192.168.0.26 in the PV claims below. Your subnet may differ slightly.

Ok, with all that clear, let's create our PVs (remembering to substitute the correct IP and paths for each claim):

Create nfs-pv1:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv1
spec:
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  capacity:
    storage: 10Gi
  nfs:
    path: /mnt/nfs/one
    server: 192.168.0.26
  persistentVolumeReclaimPolicy: Delete
  storageClassName: nfs
  volumeMode: Filesystem
EOF

persistentvolume/nfs-pv1 created
~~~

Create nfs-pv2:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv2
spec:
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  capacity:
    storage: 10Gi
  nfs:
    path: /mnt/nfs/two
    server: 192.168.0.26
  persistentVolumeReclaimPolicy: Delete
  storageClassName: nfs
  volumeMode: Filesystem
EOF

persistentvolume/nfs-pv2 created
~~~



Check the PV's:

~~~bash
$ oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                             STORAGECLASS   REASON   AGE
nfs-pv1                                    10Gi       RWO,RWX        Delete           Available                                                     nfs                     11s
nfs-pv2                                    10Gi       RWO,RWX        Delete           Available                                                     nfs                     7s
pvc-2c361fda-f643-45bb-8c53-d714ff2cfa0f   100Gi      RWO            Delete           Bound       openshift-image-registry/image-registry-storage   standard                19h
~~~


Now let's create a new NFS-based Peristent Volume Claim (PVC). 

For this volume claim we will use a special annotation `cdi.kubevirt.io/storage.import.endpoint` which utilises the Kubernetes Containerized Data Importer (CDI). 

> **NOTE**: CDI is a utility to import, upload, and clone virtual machine images for OpenShift virtualisation. The CDI controller watches for this annotation on the PVC and if found it starts a process to import, upload, or clone. When the annotation is detected the `CDI` controller starts a pod which imports the image from that URL. Cloning and uploading follow a similar process. Read more about the Containerised Data Importer [here](https://kubevirt.io/2018/containerized-data-importer.html).

Basically we are askng OpenShift to create this PVC and use the image in the endpoint to fill it. In this case we are going to use a publically available Centos 8 image at `"https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.2.2004-20200611.2.x86_64.qcow2"`in the annotation.

In addition to triggering the CDI utility we also specify the storage class we created earlier (`nfs`) which is setting the `kubernetes.io/no-provisioner` type as described. Finally note the `requests` section. We are asking for a 10gb volume size which we ensured were available previously via nfs-pv1 and nfs-pv2.

OK, let's create the PVC with all this included!

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "centos8-nfs"
  labels:
    app: containerized-data-importer
  annotations:
    cdi.kubevirt.io/storage.import.endpoint: "https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.2.2004-20200611.2.x86_64.qcow2"
spec:
  volumeMode: Filesystem
  storageClassName: nfs
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
EOF

persistentvolumeclaim/rhel8-nfs created
~~~

Once created, CDI triggers the importer pod automatically to take care of the conversion for you:

~~~bash
$ oc get pods
NAME                   READY   STATUS              RESTARTS   AGE
importer-centos8-nfs     0/1     ContainerCreating   0          1s
~~~

Watch the logs and you can see the process. The image is imported into the PV which is sitting on our NFS share. Watch the logs and wait a few minutes for the import to register Complete:

~~~bash
$ oc logs importer-centos8-nfs -f
I0721 01:58:35.578683       1 importer.go:51] Starting importer
I0721 01:58:35.580412       1 importer.go:107] begin import process
I0721 01:58:35.824561       1 data-processor.go:275] Calculating available size
I0721 01:58:35.827952       1 data-processor.go:283] Checking out file system volume size.
I0721 01:58:35.828387       1 data-processor.go:287] Request image size not empty.
I0721 01:58:35.828415       1 data-processor.go:292] Target size 10Gi.
I0721 01:58:35.842513       1 data-processor.go:205] New phase: Convert
I0721 01:58:35.842538       1 data-processor.go:211] Validating image
I0721 01:58:36.142987       1 qemu.go:212] 0.00
I0721 01:58:40.640927       1 qemu.go:212] 1.00
I0721 01:58:43.963355       1 qemu.go:212] 2.00
I0721 01:58:47.147158       1 qemu.go:212] 3.01
I0721 01:58:50.607386       1 qemu.go:212] 4.01
I0721 01:58:54.773651       1 qemu.go:212] 5.01
I0721 01:58:58.695689       1 qemu.go:212] 6.01
I0721 01:59:02.230260       1 qemu.go:212] 7.03
(...)
I0721 02:04:40.747989       1 qemu.go:212] 94.64
I0721 02:04:44.432514       1 qemu.go:212] 95.64
I0721 02:04:48.652009       1 qemu.go:212] 96.64
I0721 02:04:53.190578       1 qemu.go:212] 97.64
I0721 02:04:57.453249       1 qemu.go:212] 98.64
I0721 02:05:01.055319       1 qemu.go:212] 99.65
I0721 02:05:02.264574       1 data-processor.go:205] New phase: Resize
I0721 02:05:02.280021       1 data-processor.go:265] No need to resize image. Requested size: 10Gi, Image size: 10737418240.
I0721 02:05:02.280048       1 data-processor.go:205] New phase: Complete
I0721 02:05:02.280128       1 importer.go:160] Import complete
~~~

If you're quick, you can view the structure of the importer pod in another window to get some of the configuration it's using:

~~~bash
$ oc describe pod $(oc get pods | awk '/importer/ {print $1;}')
Name:         importer-centos8-nfs
Namespace:    default
Priority:     0
Node:         ocp-9pv98-worker-pj2dn/10.0.0.38
Start Time:   Mon, 20 Jul 2020 21:58:18 -0400
Labels:       app=containerized-data-importer
              cdi.kubevirt.io=importer
              cdi.kubevirt.io/storage.import.importPvcName=centos8-nfs
              prometheus.cdi.kubevirt.io=
(...)
    Environment:
      IMPORTER_SOURCE:       http
      IMPORTER_ENDPOINT:     https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.2.2004-20200611.2.x86_64.qcow2
      IMPORTER_CONTENTTYPE:  kubevirt
      IMPORTER_IMAGE_SIZE:   10Gi
      OWNER_UID:             31824304-0176-499c-8a28-ac663ae7a3ee
      INSECURE_TLS:          false
    Mounts:
      /data from cdi-data-vol (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-bxr5x (ro)
(...)
Volumes:
  cdi-data-vol:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  centos8-nfs
    ReadOnly:   false
(...)
~~~

Here we can see the importer settings we requested through our claims, such as `IMPORTER_SOURCE`, `IMPORTER_ENDPOINT`, and`IMPORTER_IMAGE_SIZE`. 

Once this process has completed you'll notice that your PVC is ready to use:

~~~bash
$ oc get pvc
NAME          STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
centos8-nfs   Bound    nfs-pv1   10Gi       RWO,RWX        nfs            7m4s
~~~

> **NOTE**: In your environment it may be bound to `nfs-pv1` or `nfs-pv2` - we simply created two earlier for convenience, so don't worry if your output is not exactly the same here. The important part is that it's showing as `Bound`.

This same configuration should be reflected when asking OpenShift for a list of persistent volumes (`PV`), noting that one of the PV's we created earlier is still showing as `Available` and one is `Bound` to the claim we just requested:

~~~bash
$ oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                             STORAGECLASS   REASON   AGE
nfs-pv1                                    10Gi       RWO,RWX        Delete           Bound       default/centos8-nfs                               nfs                     7m58s
nfs-pv2                                    10Gi       RWO,RWX        Delete           Available                                                     nfs                     7m47s
pvc-2c361fda-f643-45bb-8c53-d714ff2cfa0f   100Gi      RWO            Delete           Bound       openshift-image-registry/image-registry-storage   standard                19h
~~~

Recall that when we setup the `PV` resources we specified the location and path of the NFS server that we wanted to utilise:

~~~basic
  nfs:
    path: /mnt/nfs/one
    server: 192.168.0.26
~~~

If we have a look on our NFS server (ie the RHPDS bastion host) we can make sure that it's using this correctly

> **NOTE**: In your environment, if your image was 'pv1' rather than 'pv2', adjust the commands to match your setup (check both /mnt/nfs/one and /mnt/nfs/two). 

~~~bash
[asimonel-redhat.com@bastion cnv]$ ls -l /mnt/nfs/one/
total 2256116
-rw-r--r--. 1 root root 10737418240 Jul 20 22:05 disk.img

[asimonel-redhat.com@bastion cnv]$ sudo qemu-img info /mnt/nfs/one/disk.img
image: /mnt/nfs/one/disk.img
file format: raw
virtual size: 10G (10737418240 bytes)
disk size: 2.2G

[asimonel-redhat.com@bastion cnv]$ sudo file /mnt/nfs/one/disk.img
/mnt/nfs/one/disk.img: x86 boot sector; partition 1: ID=0x83, active, starthead 4, startsector 2048, 16384000 sectors, code offset 0x63
~~~

We'll use this NFS-based Centos8 image when we provision a virtual machine in a future step.


## Hostpath Storage

Now let's create a second storage type based on `hostpath` storage. We'll follow a similar approach to setting this up, but we won't be using shared storage, so all of the data that we create on hostpath type volumes is essentially ephemeral, and resides on the local disk of the hypervisors (ie the OpenShift workers) themselves. As we're not using a pre-configured shared storage pool for this we need to ask OpenShift's `MachineConfigOperator` to do some work for us directly on our worker nodes.

Run the following in the terminal window - it will generate a new `MachineConfig` that the cluster will enact, recognising that we only match on the worker nodes (`machineconfiguration.openshift.io/role: worker`):

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 50-set-selinux-for-hostpath-provisioner-worker
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 2.2.0
    systemd:
      units:
        - contents: |
            [Unit]
            Description=Set SELinux chcon for hostpath provisioner
            Before=kubelet.service

            [Service]
            Type=oneshot
            RemainAfterExit=yes
            ExecStartPre=-mkdir -p /var/hpvolumes
            ExecStart=/usr/bin/chcon -Rt container_file_t /var/hpvolumes

            [Install]
            WantedBy=multi-user.target
          enabled: true
          name: hostpath-provisioner.service
EOF

machineconfig.machineconfiguration.openshift.io/50-set-selinux-for-hostpath-provisioner-worker created
~~~

This deploys a new `systemd` unit file on the worker nodes to create a new directory at `/var/hpvolumes` and relabels it with the correct SELinux contexts at boot-time, ensuring that OpenShift can leverage that directory for local storage. We do this via a `MachineConfig` as the CoreOS machine is immutable. You should first start to witness OpenShift starting to drain the worker nodes and disable scheduling on them so the nodes can be rebooted safely:

~~~bash
$ oc get nodes
NAME                     STATUS                     ROLES    AGE   VERSION
ocp-9pv98-master-0       Ready                      master   20h   v1.18.3+b74c5ed
ocp-9pv98-master-1       Ready                      master   20h   v1.18.3+b74c5ed
ocp-9pv98-master-2       Ready                      master   20h   v1.18.3+b74c5ed
ocp-9pv98-worker-g78bj   Ready,SchedulingDisabled   worker   20h   v1.18.3+b74c5ed
ocp-9pv98-worker-pj2dn   Ready                      worker   20h   v1.18.3+b74c5ed
~~~

> **NOTE**: This will take a few minutes (allow about 10 mins in an RHPDS-based lab) to reflect on the cluster, and causes the worker nodes to reboot. You'll witness a disruption on the lab guide functionality where you will see the consoles hang and/or display a "Closed" image. In some cases we have needed to refresh the entire browser.
>
> <img src="img/disconnected-terminal.png"/>

> It should automatically reconnect but if it doesn't, you can try reloading the terminal by clicking the three bars in the top right hand corner:

<img src="img/reload-terminal.png"/>

When you're able to issue commands again, make sure you're in the correct namespace again:

~~~bash
$ oc project default
Now using project "default" on server "https://172.30.0.1:443".
~~~

Now wait for the following command to return `True` as it indicates when the `MachineConfigPool`'s worker has been updated with the latest `MachineConfig` requested (again, this will take about 10 minutes in RHPDS): 

~~~bash
$ oc get machineconfigpool worker -o=jsonpath="{.status.conditions[?(@.type=='Updated')].status}{\"\n\"}"
True
~~~

Now we can set the HostPathProvisioner configuration itself, i.e. telling the operator what path to actually use - the systemd file we just applied merely ensures that the directory is present and has the correct SELinux labels applied to it:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: hostpathprovisioner.kubevirt.io/v1alpha1
kind: HostPathProvisioner
metadata:
  name: hostpath-provisioner
spec:
  imagePullPolicy: IfNotPresent
  pathConfig:
    path: "/var/hpvolumes"
    useNamingPrefix: "false"
EOF

hostpathprovisioner.hostpathprovisioner.kubevirt.io/hostpath-provisioner created
~~~

When you've applied this config, an additional pod will be spawned on each of the worker nodes; this pod is responsible for managing the hostpath access on the respective host; note the shorter age (42s in the example below):

~~~bash
$ oc get pods -n openshift-cnv | grep hostpath
hostpath-provisioner-chx6l                        1/1     Running   0          14s
hostpath-provisioner-n2xwq                        1/1     Running   0          14s
hostpath-provisioner-operator-584d87f57d-r89c4    1/1     Running   0          9m41s
~~~

We're now ready to configure a new `StorageClass` for the HostPath based storage:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hostpath-provisioner
provisioner: kubevirt.io/hostpath-provisioner
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF

storageclass.storage.k8s.io/hostpath-provisioner created
~~~

You'll note that this storage class **does** have a provisioner (as opposed to the previous use of `kubernetes.io/no-provisioner`), and therefore it can create persistent volumes dynamically when a claim is submitted by the user, let's validate that by creating a new hostpath based PVC and checking that it creates the associated PV:


~~~bash
$ cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "centos8-hostpath"
  labels:
    app: containerized-data-importer
  annotations:
    cdi.kubevirt.io/storage.import.endpoint: "http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2"
spec:
  volumeMode: Filesystem
  storageClassName: hostpath-provisioner
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF

persistentvolumeclaim/centos8-hostpath created
~~~

We use CDI to ensure the volume we're requesting uses the same Centos8 image we used before:

~~~bash
$ oc get pods
NAME                        READY   STATUS    RESTARTS   AGE
importer-centos8-hostpath   1/1     Running   0          15s
~~~

> **NOTE**: You can watch the output of this importer pod with `$ oc logs -f importer-centos8-hostpath`.  Didn't see any pods? You likely just missed it. To be sure the PV was created continue to the next command.

Once that pod has finished let's check the status of the PV's:

~~~bash
$ oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                             STORAGECLASS           REASON   AGE
nfs-pv1                                    10Gi       RWO,RWX        Delete           Bound       default/centos8-nfs                               nfs                             33m
nfs-pv2                                    10Gi       RWO,RWX        Delete           Available                                                     nfs                             33m
pvc-2c361fda-f643-45bb-8c53-d714ff2cfa0f   100Gi      RWO            Delete           Bound       openshift-image-registry/image-registry-storage   standard                        20h
pvc-cf71987e-a9ff-4f2a-85de-124c578acaf8   29Gi       RWO            Delete           Bound       default/centos8-hostpath                          hostpath-provisioner            60s
~~~

> **NOTE**: The capacity displayed above lists the available space on the host, not the actual size of the persistent volume when being used.


Let's look more closely at the PV's. Describe the new hostpath PV (noting that you'll need to adapt for the `uuid` in your environment):

~~~bash
$ oc describe pv/pvc-cf71987e-a9ff-4f2a-85de-124c578acaf8
Name:              pvc-cf71987e-a9ff-4f2a-85de-124c578acaf8
Labels:            <none>
Annotations:       hostPathProvisionerIdentity: kubevirt.io/hostpath-provisioner
                   kubevirt.io/provisionOnNode: ocp-9pv98-worker-pj2dn
                   pv.kubernetes.io/provisioned-by: kubevirt.io/hostpath-provisioner
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      hostpath-provisioner
Status:            Bound
Claim:             default/centos8-hostpath
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          29Gi
Node Affinity:
  Required Terms:
    Term 0:        kubernetes.io/hostname in [ocp-9pv98-worker-pj2dn]
Message:
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /var/hpvolumes/pvc-cf71987e-a9ff-4f2a-85de-124c578acaf8
    HostPathType:
Events:            <none>
~~~

There's a few important details here worth noting, namely the `kubevirt.io/provisionOnNode` annotation, and the path of the volume on that node. In the example above you can see that the volume was provisioned on *ocp-9pv98-worker-pj2dn*, one our two worker nodes. 

Let's look more closely to verify that this truly has been created for us on the designated worker.

> **NOTE**: Adjust the following values to your environment. 

Create a debug pod and session on the node listed above

~~~bash
$ oc debug node/ocp-9pv98-worker-pj2dn
Starting pod/ocp-9pv98-worker-pj2dn-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.0.0.38
If you don't see a command prompt, try pressing enter.
sh-4.2# chroot /host
~~~

And review the PVC created in the /var/hpvolumes director

~~~bash
sh-4.4# ls -l /var/hpvolumes/pvc-cf71987e-a9ff-4f2a-85de-124c578acaf8/disk.img
-rw-r--r--. 1 root root 10737418240 Jul 21 02:30 /var/hpvolumes/pvc-cf71987e-a9ff-4f2a-85de-124c578acaf8/disk.img

sh-4.4# file /var/hpvolumes/pvc-cf71987e-a9ff-4f2a-85de-124c578acaf8/disk.img
/var/hpvolumes/pvc-cf71987e-a9ff-4f2a-85de-124c578acaf8/disk.img: DOS/MBR boot sector
~~~

And don't forget to exit and terminate the debug pod.

~~~bash
sh4.4# exit
sh4.2# exit
Removing debug pod ...

$ oc whoami
system:serviceaccount:workbook:cnv
~~~

Make sure that you've executed the two `exit` commands above - we need to make sure that you're back to the right shell before continuing, and aren't still inside of the debug pod.