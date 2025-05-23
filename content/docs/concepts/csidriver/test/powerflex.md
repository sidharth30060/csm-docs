---
title: Test PowerFlex CSI Driver
linktitle: PowerFlex
description: Tests to validate PowerFlex CSI Driver installation
---

This section provides multiple methods to test driver functionality in your environment.

**Note**: To run the test for CSI Driver for PowerFlex, install Helm 3.

## Test deploying a simple pod with PowerFlex storage

Test the deployment workflow of a simple pod on PowerFlex storage.

**Prerequisites**

In the source code, there is a directory that contains examples of how you can use the driver. To use these examples, you must create a _helmtest-vxflexos_ namespace, using `kubectl create namespace helmtest-vxflexos`, before you can start testing. HELM 3 must be installed to perform the tests.

The `starttest.sh` script is located in the `csi-vxflexos/test/helm` directory. This script is used in the following procedure to deploy helm charts that test the deployment of a simple pod.

**Steps**

1. Navigate to the test/helm directory, which contains the `starttest.sh` and the _2vols_ directories. This directory contains a simple Helm chart that will deploy a pod that uses two PowerFlex volumes.
*NOTE:* Helm tests are designed assuming users are using the _storageclass_ names (_vxflexos_ and _vxflexos-xfs_). If your _storageclass_ names differ from these values, please update the templates in 2vols accordingly (located in `test/helm/2vols/templates` directory). You can use `kubectl get sc` to check for the _storageclass_ names.
2. Run `sh starttest.sh 2vols` to deploy the pod. You should see the following:
```
Normal Pulled  38s kubelet, k8s113a-10-247-102-215.lss.emc.com Successfully pulled image "docker.io/centos:latest"
Normal Created 38s kubelet, k8s113a-10-247-102-215.lss.emc.com Created container
Normal Started 38s kubelet, k8s113a-10-247-102-215.lss.emc.com Started container
/dev/scinib 8125880 36852 7653216 1% /data
/dev/scinia 16766976 32944 16734032 1% /data
/dev/scinib on /data0 type ext4 (rw,relatime,data=ordered)
/dev/scinia on /data1 type xfs (rw,relatime,attr2,inode64,noquota)
```
3. To stop the test, run `sh stoptest.sh 2vols`. This script deletes the pods and the volumes depending on the retention setting you have configured.

**Results**

An outline of this workflow is described below:
1. The _2vols_ helm chart contains two PersistentVolumeClaim definitions, one in `pvc0.yaml` , and the other in `pvc1.yaml`. They are referenced by the `test.yaml` which creates the pod. The contents of the `Pvc0.yaml` file are described below:
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvol
  namespace: helmtest-vxflexos
spec:
  accessModes:
  - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: vxflexos
```

2. The _volumeMode: Filesystem_ requires a mounted file system, and the _resources.requests.storage_ of 8Gi requires an 8 GB file. In this case, the _storageClassName: vxflexos_ directs the system to use a storage class named _vxflexos_. This step yields a mounted _ext4_ file system. You can create the _vxflexos_ and _vxflexos-xfs_ storage classes by using the yamls located in samples/storageclass.  
3. If you compare _pvol0.yaml_ and _pvol1.yaml_, you will find that the latter uses a different storage class; _vxflexos-xfs_. This class gives you an _xfs_ file system.
4. To see the volumes you created, run kubectl get persistentvolumeclaim –n helmtest-vxflexos and kubectl describe persistentvolumeclaim –n helmtest-vxflexos.
>*NOTE:* For more information about Kubernetes objects like _StatefulSet_ and _PersistentVolumeClaim_ see [Kubernetes documentation: Concepts](https://kubernetes.io/docs/concepts/).

## Test creating snapshots

Test the workflow for snapshot creation.  
>*NOTE:* Starting with version 2.0, CSI Driver for PowerFlex helm tests are designed to work exclusively with v1 snapshots.  

**Steps**

1. Start the _2vols_ container and leave it running.
    - Helm tests are designed assuming users are using the  _storageclass_ names (_vxflexos_ and _vxflexos-xfs_). If your _storageclass_ names differ from these values, update the templates in 2vols accordingly (located in `test/helm/2vols/templates` directory). You can use `kubectl get sc` to check for the _storageclass_ names.
    - Helm tests are designed assuming users are using the _snapshotclass_ name: _vxflexos-snapclass_ If your _snapshotclass_ name differs from the default values, update `snap1.yaml` and `snap2.yaml` accordingly.
2. Run `sh snaptest.sh` to start the test.

This will create a snapshot of each of the volumes in the container using _VolumeSnapshot_ objects defined in `snap1.yaml` and `snap2.yaml`. The following are the contents of `snap1.yaml`:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: pvol0-snap1
  namespace: helmtest-vxflexos
spec:
  volumeSnapshotClassName: vxflexos-snapclass
  source:
    persistentVolumeClaimName: pvol0
```

**Results**

The `snaptest.sh` script will create a snapshot using the definitions in the `snap1.yaml` file. The _spec.source_ section contains the volume that will be snapped. For example, if the volume to be snapped is _pvol0_, then the created snapshot is named _pvol0-snap1_.

*NOTE:* The `snaptest.sh` shell script creates the snapshots, describes them, and then deletes them. You can see your snapshots using `kubectl get volumesnapshot -n helmtest-vxflexos`.

Notice that this _VolumeSnapshot_ class has a reference to a _snapshotClassName: vxflexos-snapclass_. The CSI Driver for PowerFlex installation does not create this class. You will need
to create instance of _VolumeSnapshotClass_ from one of default samples in `samples/volumesnapshotclass' directory.

## Test restoring from a snapshot

Test the restore operation workflow to restore from a snapshot.

**Prerequisites**

Ensure that you have stopped any previous test instance before performing this procedure.

**Steps**

1. Run `sh snaprestoretest.sh` to start the test.

This script deploys the _2vols_ example, creates a snap of _pvol0_, and then updates the deployed helm chart from the updated directory _2vols+restore_. This then adds an additional volume that is created from the snapshot.

*NOTE:*
- Helm tests are designed assuming users are using the _storageclass_ names (_vxflexos_ and _vxflexos-xfs_). If your _storageclass_ names differ from these values, update the templates for snap restore tests accordingly (located in `test/helm/2vols+restore/template` directory). You can use `kubectl get sc` to check for the _storageclass_ names.
- Helm tests are designed assuming users are using the _snapshotclass_ name: _vxflexos-snapclass_ If your _snapshotclass_ name differs from the default values, update `snap1.yaml` and `snap2.yaml` accordingly.

**Results**

An outline of this workflow is described below:
1. The snapshot is taken using `snap1.yaml`.
2. _Helm_ is called to upgrade the deployment with a new definition, which is found in the _2vols+restore_ directory. The `csi-vxflexos/test/helm/2vols+restore/templates` directory contains the newly created `createFromSnap.yaml` file. The script then creates a _PersistentVolumeClaim_, which is a volume that is dynamically created from the snapshot. Then the helm deployment is upgraded to contain the newly created third volume. In other words, when the `snaprestoretest.sh` creates a new volume with data from the snapshot, the restore operation is tested. The contents of the `createFromSnap.yaml` are described below:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restorepvc
  namespace: helmtest-vxflexos
spec:
  storageClassName: vxflexos
  dataSource:
    name: pvol0-snap1
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
```

*NOTE:* The _spec.dataSource_ clause, specifies a source _VolumeSnapshot_ named _pvol0-snap1_ which matches the snapshot's name in `snap1.yaml`.

## Test creating NFS volumes
**Steps**

1. Navigate to the test/helm directory, which contains the `starttest.sh` and the _1vol-nfs_ directories. This directory contains a simple Helm chart that will deploy a pod that uses one PowerFlex volumes for NFS filesystem type.

*NOTE:*
- Helm tests are designed assuming users are using the _storageclass_ name: _vxflexos-nfs_. If your _storageclass_ names differ from these values, please update the templates in 1vol-nfs accordingly (located in `test/helm/1vol-nfs/templates` directory). You can use `kubectl get sc` to check for the _storageclass_ names.

3. Run `sh starttest.sh 1vol-nfs` to deploy the pod. You should see the following:
```
Normal  Scheduled  default-scheduler, Successfully assigned helmtest-vxflexos/vxflextest-0 to worker-1-zwfjtd4eoblkg.domain
Normal  SuccessfulAttachVolume  attachdetach-controller, AttachVolume.Attach succeeded for volume "k8s-e279d47296"
Normal  Pulled  13s   kubelet, Successfully pulled image "docker.io/centos:latest" in 791.117427ms (791.125522ms including waiting)
Normal  Created  13s   kubelet, Created container test
Normal  Started  13s   kubelet, Started container test
10.x.x.x:/k8s-e279d47296   8388608  1582336   6806272  19% /data0
10.x.x.x:/k8s-e279d47296 on /data0 type nfs4 (rw,relatime,vers=4.2,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=10.x.x.x,local_lock=none,addr=10.x.x.x)
```
3. To stop the test, run `sh stoptest.sh 1vol-nfs`. This script deletes the pods and the volumes depending on the retention setting you have configured.

**Results**

An outline of this workflow is described below:
1. The _1vol-nfs_ helm chart contains one PersistentVolumeClaim definition in `pvc0.yaml`. It is referenced by the `test.yaml` which creates the pod. The contents of the `pvc0.yaml` file are described below:
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvol0
  namespace: helmtest-vxflexos
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: vxflexos-nfs
```

2. The _volumeMode: Filesystem_ requires a mounted file system, and the _resources.requests.storage_ of 8Gi requires an 8 GB file. In this case, the _storageClassName: vxflexos-nfs_ directs the system to use a storage class named _vxflexos-nfs_. This step yields a mounted _nfs_ file system. You can create the _vxflexos-nfs_ storage classes by using the yaml located in samples/storageclass.
3. To see the volumes you created, run `kubectl get persistentvolumeclaim -n helmtest-vxflexos` and `kubectl describe persistentvolumeclaim -n helmtest-vxflexos`.
>*NOTE:* For more information about Kubernetes objects like _StatefulSet_ and _PersistentVolumeClaim_ see [Kubernetes documentation: Concepts](https://kubernetes.io/docs/concepts/).

## Test restoring NFS volume from snapshot
Test the restore operation workflow to restore NFS volume from a snapshot.

**Prerequisites**

Ensure that you have stopped any previous test instance before performing this procedure.

**Steps**

1. Run `sh snaprestoretest-nfs.sh` to start the test.

This script deploys the _1vol-nfs_ example, creates a snap of _pvol0_, and then updates the deployed helm chart from the updated directory _1vols+restore-nfs_. This adds an additional volume that is created from the snapshot.

*NOTE:*
- Helm tests are designed assuming users are using the _storageclass_ name: _vxflexos-nfs_. If your _storageclass_ names differ from these values, update the templates for 1vols+restore-nfs accordingly (located in `test/helm/1vols+restore-nfs/template` directory). You can use `kubectl get sc` to check for the _storageclass_ names.
- Helm tests are designed assuming users are using the _snapshotclass_ name: _vxflexos-snapclass_ If your _snapshotclass_ name differs from the default values, update `snap1.yaml` accordingly.

**Results**

An outline of this workflow is described below:
1. The snapshot is taken using `snap1.yaml`.
2. _Helm_ is called to upgrade the deployment with a new definition, which is found in the _1vols+restore-nfs_ directory. The `csi-vxflexos/test/helm/1vols+restore-nfs/templates` directory contains the newly created `createFromSnap.yaml` file. The script then creates a _PersistentVolumeClaim_, which is a volume that is dynamically created from the snapshot. Then the helm deployment is upgraded to contain the newly created third volume. In other words, when the `snaprestoretest-nfs.sh` creates a new volume with data from the snapshot, the restore operation is tested. The contents of the `createFromSnap.yaml` are described below:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restorepvc
  namespace: helmtest-vxflexos
spec:
  storageClassName: vxflexos-nfs
  dataSource:
    name: pvol0-snap1
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
```

*NOTE:* The _spec.dataSource_ clause, specifies a source _VolumeSnapshot_ named _pvol0-snap1_ which matches the snapshot's name in `snap1.yaml`.