---
title: Replication
linkTitle: "Replication"
description: >
  Installing Replication via Container Storage Modules Operator
--- 

1. ##### **Review the SyncIQ configuration on both the source and target PowerScale**
   
   <br>

   a. Use the below command to verify the SyncIQ is licensed on both PowerScale.  

   <div></div>

   ```bash
    isi license list
   ``` 

   b. Use the below command to review the SyncIQ configuration on the source PowerScale 

   <div></div>

      ```terminal
     ps01-1# isi sync settings view

                                       Service: on
                                 Source Subnet: -
                                   Source Pool: -
                               Force Interface: No
                       Restrict Target Network: No
                             Tw Chkpt Interval: -
                                Report Max Age: 1Y
                              Report Max Count: 2000
                                    RPO Alerts: Yes
                           Max Concurrent Jobs: 24
      Bandwidth Reservation Reserve Percentage: 1
        Bandwidth Reservation Reserve Absolute: -
                           Encryption Required: Yes
                        Cluster Certificate ID: 809c57b723f765b33a4a1a9905fd5837c12ae0ebe5f75ffd5aa3353cd83536e8
                    OCSP Issuer Certificate ID: 
                                  OCSP Address: 
                        Encryption Cipher List: 
                           Elliptic Curve List: 
                          Renegotiation Period: 8H
                       Service History Max Age: 1Y
                     Service History Max Count: 2000
                          Use Workers Per Node: No
                           Preferred RPO Alert: Never
                                  Password Set: No
      ``` 
   
   <br>

   c. Use this command to review the SyncIQ configuration on the target PowerScale 

   <div></div> 

    ```terminal
    ps02-1# isi sync settings view
                                     Service: on
                               Source Subnet: -
                                 Source Pool: -
                             Force Interface: No
                     Restrict Target Network: No
                           Tw Chkpt Interval: -
                              Report Max Age: 1Y
                            Report Max Count: 2000
                                  RPO Alerts: Yes
                         Max Concurrent Jobs: 24
    Bandwidth Reservation Reserve Percentage: 1
      Bandwidth Reservation Reserve Absolute: -
                         Encryption Required: Yes
                      Cluster Certificate ID: 1e3def272e919debfb3cb5bfd1a8de2be09d4b0dfe9a0af1b3b26eab16477e80
                  OCSP Issuer Certificate ID: 
                                OCSP Address: 
                      Encryption Cipher List: 
                         Elliptic Curve List: 
                        Renegotiation Period: 8H
                     Service History Max Age: 1Y
                   Service History Max Count: 2000
                        Use Workers Per Node: No
                         Preferred RPO Alert: Never
                                Password Set: No 
    ```
<br> 

2. ##### **Install and Configure repctl utility** 
    
    <br>

    a. Use the below commands to download and install the repctl utiltiy.
    ```bash
    wget -O repctl https://github.com/dell/csm-replication/releases/download/<version>/repctl-linux-amd64
    chmod +x repctl
    mv repctl /usr/local/bin/
    ``` 
    
    b. Verify the repctl utility is installed.
    ```terminal
    repctl -v

    repctl version v1.11.0
    ````
    
    c. Add the source OpenShift cluster to the repctl configuration.
    ```bash
    repctl cluster add -f kubeconfig -n ocp01
    ```
    
    d. Add the target OpenShift cluster to the repctl configutation
    ```bash
    repctl cluster add -f kubeconfig -n ocp02
    ``` 
    
    e. Verify both the source and target OpenShift clusters are added to the repctl.
    ```terminal
    repctl cluster get 

    +---------------+
    | Cluster       |
    +---------------+
    ClusterId       Version URL
    ocp01           v1.30   https://api.ocp01.vdi.xtremio:6443
    ocp02           v1.30   https://api.ocp02.vdi.xtremio:6443  
    ```
 
<br> 

3. ##### **Create the replication CRDs** 

   <br> 

   Use these commands to create the replication CRD in the OpenShift Cluster. The repctl command will create the CRDs on both source and target OpenShift Cluster.
   ```bash
   git clone -b <version> https://github.com/dell/csm-replication.git
   repctl create -f csm-replication/deploy/replicationcrds.all.yaml 
   ```

<br> 

4. ##### **Inject the service account’s configuration into the clusters.**
   
   <br>

   Use this command to inject the service account configuration to both the clusters.
   ```bash
   repctl cluster inject --use-sa 
   ``` 

5. ##### **Install CSM Operator only on the source OpenShift Cluster** 

   <br> 

   a. On the OpenShift console, navigate to **OperatorHub** and use the keyword filter to search for **Dell Container Storage Modules.** 

   b. Click **Dell Container Storage Modules** tile 

   c. Keep all default settings and click **Install**.

   </br>
   

   Verify that the operator is deployed  

   ```terminal 
   oc get operators

   NAME                                                          AGE
   dell-csm-operator-certified.openshift-operators               2d21h
   ```  

   ```terminal
   oc get pod -n openshift-operators

   NAME                                                       READY   STATUS       RESTARTS      AGE
   dell-csm-operator-controller-manager-86dcdc8c48-6dkxm      2/2     Running      0             2d21h
   ``` 

<br>

6. ##### **Create Project in both the source and target OpenShift cluster**
   
    <br> 

    Use this command to create new project. You can use any project name instead of `isilon`.

    ```bash 
    oc new-project isilon
    ```


7. ##### **Create config secret on both source and target OpenShift Cluster** 

    <br>   
    
    Create a file called `config.yaml` or use [sample](https://github.com/dell/csi-powerscale/blob/main/samples/secret/secret.yaml): 
   
    Example: 
    <div style="margin-bottom: -1.8rem">


    ```yaml
    cat <<EOF> config.yaml
    isilonClusters:
    - clusterName: "ps01"
      username: "csmadmin"
      password: "P@ssw0rd123"
      endpoint: "ps01.vdi.xtremio"
      skipCertificateValidation: true
      replicationCertificateID: "809c57b723f765b33a4a1a9905fd5837c12ae0ebe5f75ffd5aa3353cd83536e8" 
    - clusterName: "ps02"
      username: "csmadmin"
      password: "P@ssw0rd123"
      endpoint: "ps02.vdi.xtremio"
      skipCertificateValidation: true
      replicationCertificateID: "1e3def272e919debfb3cb5bfd1a8de2be09d4b0dfe9a0af1b3b26eab16477e80"
    EOF
    ```
   </div>

 
    <br>

    Edit the file, then run the command to create the `isilon-config`.

    ```bash
    oc create secret generic isilon-config --from-file=config=config.yaml -n isilon --dry-run=client -oyaml > secret-isilon-config.yaml
    ```
    
    Use this command to **create** the config:

    ```bash 
    oc apply -f secret-isilon-config.yaml
    ```

    Use this command to **replace or update** the config:

    ```bash
    oc replace -f secret-isilon-config.yaml --force
    ```
  
    Verify config secret is created.

    ```terminal
    oc get secret -n isilon
     
    NAME                 TYPE        DATA   AGE
    isilon-config        Opaque      1      3h7m
    ```  
  </br>

8. ##### **Create PowerScale certificate secret in both source and target OpenShift Cluster**

     <br> 

      Example:   
      ```yaml 
      cat << EOF > secret-isilon-certs.yaml
      apiVersion: v1
      kind: Secret
      metadata:
         name: isilon-certs-0
         namespace: isilon
      type: Opaque
      data:
         cert-0: "" 
      EOF
      ```

      Use below command to create the powerscale certificate secret.
      ```bash
       oc apply -f secret-isilon-certs.yaml
      ```
  <br>


9. ##### **Create custom resource ContainerStorageModule for PowerScale only on the source OpenShift Cluster** 

    <br>

    Use this command to create the **ContainerStorageModule Custom Resource**:

    ```bash
    oc create -f csm-isilon.yaml
    ```

    Example:
    <div style="margin-bottom: -1.8rem">


    ```yaml
    cat <<EOF> csm-powerscale.yaml
    apiVersion: storage.dell.com/v1
    kind: ContainerStorageModule
    metadata:
      name: isilon
      namespace: isilon
    spec:
      driver:
        csiDriverType: isilon
        configVersion: v2.13.0
        authSecret: isilon-config
        replicas: 1
        csiDriverSpec:
          fSGroupPolicy: File
          storageCapacity: true
        common:
          envs:
          - name: X_CSI_ISI_AUTH_TYPE
            value: "1"
        sideCars:
        - args:
          - --volume-name-prefix=ocp01
          name: provisioner
      modules:
      - name: replication
        enabled: true
        components:
        - name: dell-replication-controller-manager
          envs:
          - name: TARGET_CLUSTERS_IDS
            value: ocp02
    EOF 
    ``` 
    </div>
   
   <br>

   Check if ContainerStorageModule CR is created successfully in the source OpenShift Cluster:

   ```terminal
   oc get csm isilon -n isilon

   NAME        CREATIONTIME   CSIDRIVERTYPE   CONFIGVERSION    STATE
   isilon      3h             isilon          {{< version-docs key="PScale_latestVersion" >}}          Succeeded      
   ```
   <br> 

   Verify the CSM Pods are running in the Source OpenShift Cluster
   
   ```terminal
   oc get pod -n isilon

   NAME                                     READY   STATUS    RESTARTS      AGE
   isilon-controller-77f8f74d4f-9vwnm       7/7     Running   0             22h
   isilon-node-5xcxz                        2/2     Running   0             24h
   isilon-node-fpct7                        2/2     Running   0             24h 
   ``` 
   <br>

   Verify the CSM Pods are running in the target OpenShift cluster.
   
   ```terminal
   oc get pod -n isilon

   NAME                                     READY   STATUS    RESTARTS   AGE
   isilon-controller-77f8f74d4f-rrhr9       7/7     Running   0          2d4h
   isilon-node-l59l7                        2/2     Running   0          2d4h
   isilon-node-smv9s                        2/2     Running   0          2d4h
   ```
   
   <br> 

   Verify the replication controller is running in both source and target OpenShift cluster.

   ```terminal 
   oc get pod -n dell-replication-controller
   
   NAME                                                   READY   STATUS    RESTARTS   AGE
   dell-replication-controller-manager-795cd7fbd6-t6s6c   1/1     Running   0          2d5h
   ```

<br> 

10. ##### **Create Storage Class** 
    
    <br>

    a. Create storage class in the source OpenShift Cluster.
       
       <div></div>

       ```bash 
       oc apply -f sc-isilon-replication.yaml 
       ``` 

       ```yaml 
       cat  <<EOF> sc-isilon-replication.yaml
       apiVersion: storage.k8s.io/v1
       kind: StorageClass
       metadata:
         name: isilon-replication
       provisioner: csi-isilon.dellemc.com
       reclaimPolicy: Delete
       allowVolumeExpansion: true
       volumeBindingMode: Immediate
       mountOptions: ["vers=4"]
       parameters:
         replication.storage.dell.com/remoteStorageClassName: "isilon-replication"
         replication.storage.dell.com/isReplicationEnabled: "true"
         replication.storage.dell.com/ignoreNamespaces: "false"
         replication.storage.dell.com/remoteRootClientEnabled: "false"
         replication.storage.dell.com/remoteClusterID: "ocp02"
         replication.storage.dell.com/volumeGroupPrefix: "csi"
         replication.storage.dell.com/remoteSystem: "ps02"
         replication.storage.dell.com/remoteAccessZone: "ps02-az01"
         replication.storage.dell.com/remoteAzServiceIP: "ps02-az01.example.com"
         replication.storage.dell.com/rpo: "Five_Minutes"
         ClusterName: ps01
         AccessZone: ps01-az01
         AzServiceIP : ps01-az01.example.com
         IsiPath: /ifs/data/az01/csi
         IsiVolumePathPermissions: "0775"
         RootClientEnabled: "false"
       EOF
       ``` 

    b. Create Storage Class in the target OpenShift cluster.

      <div></div>

      ```bash
      oc apply -f sc-isilon-replication.yaml
      ``` 

      ```yaml 
      cat <<EOF> sc-isilon-replication.yaml
      apiVersion: storage.k8s.io/v1
      kind: StorageClass
      metadata:
        name: isilon-replication
      provisioner: csi-isilon.dellemc.com
      reclaimPolicy: Delete
      allowVolumeExpansion: true
      volumeBindingMode: Immediate
      mountOptions: ["vers=4"]
      parameters:
        replication.storage.dell.com/remoteStorageClassName: "isilon-replication"
        replication.storage.dell.com/isReplicationEnabled: "true"
        replication.storage.dell.com/ignoreNamespaces: "false"
        replication.storage.dell.com/remoteRootClientEnabled: "false"
        replication.storage.dell.com/remoteClusterID: "ocp01"
        replication.storage.dell.com/volumeGroupPrefix: "csi"
        replication.storage.dell.com/remoteSystem: "ps01"
        replication.storage.dell.com/remoteAccessZone: "ps01-az01"
        replication.storage.dell.com/remoteAzServiceIP: "ps01-az01.example.com"
        replication.storage.dell.com/rpo: "Five_Minutes"
        ClusterName: ps02
        AccessZone: ps02-az01
        AzServiceIP : ps02-az01.example.com
        IsiPath: /ifs/data/az01/csi
        IsiVolumePathPermissions: "0775"
        RootClientEnabled: "false"
      EOF
      ``` 
      
     <br> 

      Verify the Storage Class is created 

      ```terminal 
      oc get sc

      NAME                     PROVISIONER              RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
      isilon                   csi-isilon.dellemc.com   Delete          Immediate           true                   21h
      isilon-replication       csi-isilon.dellemc.com   Delete          Immediate           true                   21h
      ```
      <br> 

11. ##### **Create Deployment with replicated Persistent Volume Claim.**
   
    <br>

    Create replicated Persistent Volume Claim
    ```bash 
    oc apply -f pvc-isilon-replication.yaml
    ``` 
    
    Example:
    ```yaml
    cat <<EOF> pvc-isilon-replication.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: pvc-isilon-replication
    spec:
      accessModes:
      - ReadWriteMany
      volumeMode: Filesystem
      resources:
        requests:
          storage: 8Gi
      storageClassName: isilon-replication  
     EOF
     ``` 

     Verify the PVC is created. 
     ```terminal 
     oc get pvc -n default

     NAME                     STATUS   VOLUME             CAPACITY   ACCESS MODES   STORAGECLASS         VOLUMEATTRIBUTESCLASS   AGE
     pvc-isilon-replication   Bound    ocp01-f8eb93c213   8Gi        RWX            isilon-replication   <unset>                 2d2h 
     ```  

     <br>

     Create deployment which uses Persistent Volume Claim with replicated storage class
     
     
     ```bash 
     oc apply -f deploy-isilon-replication.yaml
     ```  

    Example: 
    ```yaml
    cat <<EOF> deploy-isilon-replication.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: deploy-demo
      namespace: default
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: demo
      template:
        metadata:
          labels:
            app: demo
        spec:
          containers:
          - name: ubi
            image: registry.access.redhat.com/ubi9/ubi
            env:
            - name: NODENAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: PODNAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            command: [ "sh", "-c" ]
            args: [ "while true; do touch /data/$NODENAME-$PODNAME-$(date +%T); sleep 5; done;" ]
            volumeMounts:
            - name: data
              mountPath: /data
          volumes:
          - name: data
            persistentVolumeClaim:
              claimName: pvc-isilon-replication
     EOF
     ```  
     
    
     Verify the Pod is running
     ```terminal 
     oc get pod -n default
 
     NAME                          READY   STATUS    RESTARTS   AGE
     deploy-demo-cbf859cff-8qjbb   1/1     Running   0          2d2h 

     ```
     
     <br>
     
     Verify the Replication Group is created
     ```terminal 
     repctl get rg

     [2025-03-19 19:33:27]  INFO listing replication groups
     [2025-03-19 19:33:27]  INFO Cluster: ocp01
     +-----+
     | RG  |
     +-----+
     Name                                    State   rClusterID      Driver                  RemoteRG                                IsSource        LinkState
     rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 Ready   ocp02           csi-isilon.dellemc.com  rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 true            SYNCHRONIZED
     [2025-03-19 19:33:27]  INFO 
     [2025-03-19 19:33:27]  INFO Cluster: ocp02
     +-----+
     | RG  |
     +-----+
     Name                                    State   rClusterID      Driver                  RemoteRG                                IsSource        LinkState
     rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 Ready   ocp01           csi-isilon.dellemc.com  rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 false           SYNCHRONIZED
     [2025-03-19 19:33:27]  INFO
     ```
     
     <br> 

     Verify the SyncIQ policies from the PowerScale cluster

     ```terminal 
     ps01-1# isi sync policies list

     Name                                       Path                                                        Action  Enabled  Target          
     ----------------------------------------------------------------------------------------------------------------------------------------------------------
     csi-default-ps02-example-com-Five_Minutes /ifs/data/az01/csi/csi-default-ps02.example.com-Five_Minutes sync    Yes      ps02.example.com
     ----------------------------------------------------------------------------------------------------------------------------------------------------------
     Total: 1
     ``` 
    <br> 

    Stop the application in the source site, if you are doing a planned failover.
    
    ```bash 
    oc scale deploy deploy-demo --replicas=0 -n default                                                                       
    deployment.apps/deploy-demo scaled    
    ```

    ```terminal
    oc get deploy -n default 
                                                                        
    NAME          READY   UP-TO-DATE   AVAILABLE   AGE                                                                                                       
    deploy-demo   0/0     0            0           2d21h
    ```
    <br> 

    Use the below commands to perform a failover from source cluster to target cluster. 
  
    ```terminal 
    repctl failover --rg rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 --target ocp02
    [2025-03-20 16:29:46]  INFO RG (rg-80a563ca-5760-44db-b656-b0d23ee2b0b6), successfully updated with action: failover
    ```

    <br> 
    Verify the status of the Replication Group after the failover 

    ```terminal 
    repctl get rg
    [2025-03-20 16:31:49]  INFO listing replication groups
    [2025-03-20 16:31:49]  INFO Cluster: ocp01
    +-----+
    | RG  |
    +-----+
    Name                                    State   rClusterID      Driver                  RemoteRG                                IsSource        LinkState
    rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 Ready   ocp02           csi-isilon.dellemc.com  rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 true            FAILEDOVER
    [2025-03-20 16:31:49]  INFO 
    [2025-03-20 16:31:49]  INFO Cluster: ocp02
    +-----+
    | RG  |
    +-----+
    Name                                    State   rClusterID      Driver                  RemoteRG                                IsSource        LinkState
    rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 Ready   ocp01           csi-isilon.dellemc.com  rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 false           FAILEDOVER
    [2025-03-20 16:31:49]  INFO
    ``` 
    <br>

    Create PVC in the target cluster. 

    ```bash 
    repctl create pvc --rg rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 -t default --clusters ocp02
    ``` 

    Verify the PVC in the target cluster 

    ```terminal 
    oc get pvc -n default

    NAME                     STATUS   VOLUME             CAPACITY   ACCESS MODES   STORAGECLASS             VOLUMEATTRIBUTESCLASS   AGE
    pvc-isilon-replication   Bound    ocp01-f8eb93c213   8Gi        RWX            isilon-replication       <unset>                 98s
    ```
    
    <br>
    Start the application in the target OpenShift cluster

    ```bash
    oc apply -f deploy-isilon-replication.yaml
    ``` 

    Verify the pod is running in the target OpenShift cluster. 

    ```terminal 
    oc get pod -n default

    NAME                          READY   STATUS    RESTARTS   AGE
    deploy-demo-cbf859cff-g9qfh   1/1     Running   0          22s
    
    ```
    
    <br> 

    Reprotect the application.

    ```bash 
    repctl reprotect --rg rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 --at ocp02
    [2025-03-24 12:43:03]  INFO RG (rg-80a563ca-5760-44db-b656-b0d23ee2b0b6), successfully updated with action: reprotect
    ```  
    

    Verify the replication group status.
    ```terminal 
    repctl get rg
    
    [2025-03-24 12:43:30]  INFO listing replication groups
    [2025-03-24 12:43:30]  INFO Cluster: ocp01
    +-----+
    | RG  |
    +-----+
    Name                                    State   rClusterID      Driver                  RemoteRG                                IsSource        LinkState
    rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 Ready   ocp02           csi-isilon.dellemc.com  rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 false           SYNCHRONIZED
    [2025-03-24 12:43:30]  INFO 
    [2025-03-24 12:43:30]  INFO Cluster: ocp02
    +-----+
    | RG  |
    +-----+
    Name                                    State   rClusterID      Driver                  RemoteRG                                IsSource        LinkState
    rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 Ready   ocp01           csi-isilon.dellemc.com  rg-80a563ca-5760-44db-b656-b0d23ee2b0b6 true            SYNCHRONIZED
    [2025-03-24 12:43:30]  INFO
    ````
    
    <br>

    Verify the SyncIQ policy from the PowerScale Cluster. 

    ```terminal 
    ps02-1# isi sync policies list

    Name                                       Path                                                        Action  Enabled  Target          
    ----------------------------------------------------------------------------------------------------------------------------------------------------------
    csi-default-ps02-example-com-Five_Minutes /ifs/data/az01/csi/csi-default-ps02.example.com-Five_Minutes sync    Yes      ps01.vdi.xtremio
    ----------------------------------------------------------------------------------------------------------------------------------------------------------
    Total: 1 
    ```