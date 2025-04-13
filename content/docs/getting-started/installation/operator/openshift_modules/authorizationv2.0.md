---
title: Authorization v2.0
linkTitle: "Authorization v2.0"
description: 
--- 

### Prerequisites

1. Verify the Cert-Manager is deployed and configured on the OpenShift Cluster. Please review the REd Hat document for the procedure. 

2. Hashicorp Vault is deployed. 

3. Configure Hashicorp Vault 

   ```bash 
   vault secrets enable -version=2 -path=csm-authorization/ kv
   ```

   ```bash
   vault auth enable kubernetes
   ``` 

   ```bash 
   vault write auth/kubernetes/config kubernetes_host=https://api.ocp01.example.com:6443
   ``` 
   
   ```bash 
   vault policy write csm-authorization - <<EOF
   path "csm-authorization/*" {
   capabilities = ["read"]
   }
   EOF 
   ``` 

   ```bash 
   vault write auth/kubernetes/role/csm-authorization token_ttl=1h token_max_ttl=1h token_explicit_max_ttl=10d   bound_service_account_names=storage-service ound_service_account_namespaces=authorization policies=csm-authorization 
   ``` 

   ```bash 
   vault kv put -mount=csm-authorization /storage/powerflex/pf04 username=demo password="P@ssw0rd123!"
   ``` 

### Operator Installation


1. On the OpenShift console, navigate to **OperatorHub** and use the keyword filter to search for **Dell Container Storage Modules.** 

2. Click **Dell Container Storage Modules** tile 

3. Keep all default settings and click **Install**.


   Verify that the operator is deployed 
   ```terminal 
   oc get operators

   NAME                                                          AGE
   dell-csm-operator-certified.openshift-operators               2d21h
   ```  

   ```terminal
   oc get pod -n openshift-operators

   NAME                                                       READY   STATUS       RESTARTS      AGE
   dell-csm-operator-controller-manager-86dcdc8c48-6dkxm      2/2     Running      21 (19h ago)  2d21h
   ```  

### Authorisation Proxy Server Installation 

1. ##### **Create project:**
   
   <br>

   Use this command to create new project. You can use any project name instead of `authorization`.

   ```bash 
   oc new-project authorization
   ``` 

2. ##### **Create karavi config secret:** 
   
   <br>

   Create a file called karavi-config.yaml

   Example: 
   
   ```bash 
   cat <<EOF> karavi-config.yaml
   web:
     jwtsigningsecret: randomString123
   EOF
   ``` 

   Use the below command to create the karavi-config-secret. 

   ```bash
   oc create secret generic karavi-config-secret --from-file=config.yaml=karavi-config.yaml -n authorization -n dry-run=client -oyaml > secret-karavi-config-secret.yaml
   ``` 
   
   Use this command to create the secret:
   
   ```bash
   oc apply -f secret-karavi-config-secret.yaml
   ``` 
    
   Use this command to replace or update the secret:
   
   ```bash
   oc replace -f secret-karavi-config-secret.yaml --force
   ```

   Verify the karavi config secret is created.
   
   ```terminal
   oc get secret -n authorization
    
   NAME                            TYPE           DATA   AGE
   karavi-config-secret            Opaque         1      1h7m 
   ```

3. ##### **Create csm config configmap** 

   <br>

   ```bash
   oc apply -f cm-csm-config-params.yaml
   ``` 

   Example:  

   ```yaml
   cat <<EOF> cm-csm-config-params.yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: csm-config-params
     namespace: authorization
   data:
     csm-config-params.yaml: |-
       CONCURRENT_POWERFLEX_REQUESTS: 10
       CONCURRENT_POWERSCALE_REQUESTS: 10
       LOG_LEVEL: debug
       STORAGE_CAPACITY_POLL_INTERVAL: 5m
   EOF
   ```
  
   Verify the configmap is created. 

   ```terminal 
   oc get cm csm-config-params -n authorization

   NAME                DATA   AGE
   csm-config-params   1      14h
   ```


4. ##### **Create custom resource ContainerStorageModule for Authorization Proxy Server** 
   
   <br>

   Use this command to create ContainerStorageModule custom resource.
 
   ```bash
   oc apply -f csm-authorization.yaml 
   ``` 

   Example:

   ```yaml 
   cat <<EOF> csm-authorization.yaml
   apiVersion: storage.dell.com/v1
   kind: ContainerStorageModule
   metadata:
     name: authorization
     namespace: authorization
   spec:
     modules:
     - name: authorization-proxy-server
       enabled: true
       configVersion: v2.1.0
       forceRemoveModule: true
       components:
       - name: nginx
         enabled: false
       - name: cert-manager
         enabled: false
       - name: proxy-server
         enabled: true
         proxyService: quay.io/dell/container-storage-modules/csm-authorization-proxy:v2.1.0
         proxyServiceReplicas: 1
         tenantService: quay.io/dell/container-storage-modules/csm-authorization-tenant:v2.1.0
         tenantServiceReplicas: 1
         roleService: quay.io/dell/container-storage-modules/csm-authorization-role:v2.1.0
         roleServiceReplicas: 1
         storageService: quay.io/dell/container-storage-modules/csm-authorization-storage:v2.1.0
         storageServiceReplicas: 1
         opa: docker.io/openpolicyagent/opa:0.70.0
         opaKubeMgmt: docker.io/openpolicyagent/kube-mgmt:8.5.7
         authorizationController: quay.io/dell/container-storage-modules/csm-authorization-controller:v2.1.0
         authorizationControllerReplicas: 1
         leaderElection: true
         controllerReconcileInterval: 5m
         certificate: ""
         privateKey: ""
         hostname: "csm-authorization.example.com"
         proxyServerIngress:
         - ingressClassName: nginx
           hosts: []
           annotations: {}
         openTelemetryCollectorAddress: ""
       - name: redis
         redis: docker.io/redis:7.4.0-alpine
         commander: docker.io/rediscommander/redis-commander:latest
         redisName: redis-csm
         redisCommander: rediscommander
         sentinel: sentinel
         redisReplicas: 3
       - name: vault
         vaultConfigurations:
         - identifier: vault01
           address: https://vault01.example.com
           role: csm-authorization
           skipCertificateValidation: true
           clientCertificate: ""
           clientKey: ""
           certificateAuthority: ""
   EOF
   ```

   Verify the authorization ContainerStorageModule is created sucessfully.

   ```terminal 
   oc get csm -n authorization

   NAME            CREATIONTIME   CSIDRIVERTYPE   CONFIGVERSION   STATE
   authorization   14h                                            Succeeded
   ```


   Verify the authorization Pods are running.

   ```terminal
   oc get pod -n authorization

   NAME                                        READY   STATUS    RESTARTS       AGE
   authorization-controller-5d74cc4f68-hmwrx   1/1     Running   0              2m27s
   proxy-server-687bcc879c-r8n97               3/3     Running   3 (109s ago)   2m28s
   redis-csm-0                                 1/1     Running   0              2m27s
   redis-csm-1                                 1/1     Running   0              2m4s
   redis-csm-2                                 1/1     Running   0              102s
   rediscommander-656c974cf9-mjm7c             1/1     Running   0              2m27s
   role-service-8644b95b87-cnngx               1/1     Running   0              2m28s
   sentinel-0                                  1/1     Running   0              2m27s
   sentinel-1                                  1/1     Running   0              83s
   sentinel-2                                  1/1     Running   0              81s
   storage-service-6cf45566f8-k4nrv            1/1     Running   3 (92s ago)    2m27s
   tenant-service-6c78f8c5c8-lqt7s             1/1     Running   3 (111s ago)   2m28s
   ```
   
   Verify the authorization route. 
   
   ```bash
   oc get route -n authorization

   NAME                 HOST/PORT                       PATH   SERVICES       PORT   TERMINATION     WILDCARD
   proxy-server-77btr   csm-authorization.example.com   /      proxy-server   http   edge/Redirect   None

   ```

### Authorization Proxy Configuration 


1. ##### **Install and configure dellctl utility.**

   <br>

   a. Use the below commands to download and install the dellctl utiltiy.

   ```bash 
   wget -O dellctl https://github.com/dell/csm/releases/download/<version>/dellctl
   chmod +x dellctl
   mv dellctl /usr/local/bin/
   ``` 

   b. Verify the dellctl utility is installed.

   ```terminal 
   dellctl -v
 
   dellctl version v0.10.0
   ```

   c. Add the OpenShift cluster to the dellctl configuration.

   ```bash 
   dellctl cluster add -n ocp01 -f kubeconfig 
   ```

   d. Verify the OpenShift cluster is added. 

   ```bash 
   dellctl cluster get
 
   CLUSTER ID  VERSION  URL                                 UID
   ocp01       v1.31    https://api.ocp01.example.com:6443
   ``` 

2. ##### **Configure Storage**

   <br> 

   Use the below command to create a storage system inside authorization proxy. 

   ```bash 
   oc apply -f storage-powerflex.yaml
   ``` 

   Example: 

   ```yaml 
   cat <<EOF> storage-pf04.yaml 
   apiVersion: csm-authorization.storage.dell.com/v1
   kind: Storage
   metadata:
     name: pf04
     namespace: authorization
   spec:
     type: powerflex
     endpoint: https://pf04.example.com
     systemID: 3bc028ccdc1dd90f
     skipCertificateValidation: true
     pollInterval: 30s
     vault:
       identifier: vault01
       kvEngine: csm-authorization
       path: storage/powerflex/pf04
   EOF 
   ``` 

   Verify the Storage is created.
   
   ```terminal 
   oc get storage -n auhtorization

   NAME   AGE
   pf04   16s 
   ``` 

3. ##### **Configure Tenant**

   <br>

   Use the below command to create a tenant.

   ```bash 
   oc apply -f csmtenant-ocp01.yaml
   ``` 

   Example: 

   ```yaml 
   cat <<EOF> csmtenant-ocp01.yaml
   apiVersion: csm-authorization.storage.dell.com/v1
   kind: CSMTenant
   metadata:
     name: ocp01
     namespace: authorization
   spec:
     roles: pf04-sp01
     approveSdc: false
     revoke: false
     volumePrefix: o01
   EOF
   ``` 

   Verify the CSM Tenant is created.
   
   ```terminal 
   oc get csmtenant -n authorization

   NAME    AGE
   ocp01   10s 
   ``` 

4. ##### **Create Tokens**  

   <br>

   Use the below command to create a admin token.

   ```bash 
   dellctl admin token --name admin --access-token-expiration 1m30s --refresh-token-expiration 720h --jwt-signing-secret randomString123 > token-admin.yaml
   ``` 

   Use the below command to create a tenant token. 

   ```bash 
   dellctl generate token --admin-token token-admin.yaml --addr csm-authorization.example.com --insecure true --tenant ocp01 --access-token-expiration 30m0s --refresh-token-expiration 1480h0m0s > token-tenant-ocp01.yaml
   ```



### Integrate PowerFlex Driver with Authorization Proxy Server 



1. ##### **Create Project for deploying the PowerFlex driver**  

   <br>

   ```bash 
   oc new-project vxflexos
   ``` 

2. ##### **Create the tenant token secret in the driver namespace.** 

   <br>

   Storage admin will generate and provide the tenant token for the OpenShift cluster.  

   ```bash 
   oc apply -f token-tenant-ocp01.yaml
   ``` 

   Verify the secret is created.
   
   ```terminal
   oc get secret -n vxflexos

   NAME                 TYPE     DATA   AGE
   proxy-authz-tokens   Opaque   2      146m 
   ``` 

3. ##### **Create karavi authorization config secret.** 
   <br>
 
   Create the configuration file 

   ```json 
   cat <<EOF> karavi-authorization-config.json
   [
     {
       "intendedEndpoint": "https://pf04.vdi.xtremio",
       "endpoint": "https://localhost:9400",
       "systemID": "3bc028ccdc1dd90f",
       "skipCertificateValidation": true,
     }
   ] 
   EOF 
   ```  

   Use below command to create the karavi-authorization-config secret in the powerflex driver namespace. 

   ```bash 
   oc create secret generic karavi-authorization-config --from-file=config=karavi-authorization-config.json -n vxflexos --dry-run=client -oyaml > secret-karavi-authorization-config.yaml
   ```

   Use below command to create the secret.
   
   ```bash 
   oc apply -f secret-karavi-authorization-config.yaml
   ``` 

   Verify the secret is created.
   
   ```terminal 
   oc get secret -n vxflexos

   NAME                          TYPE     DATA   AGE
   karavi-authorization-config   Opaque   1      7s
   proxy-authz-tokens            Opaque   2      3h24m
   ``` 

4. ##### **Create authorization proxy server root certificate secret**
   
   <br>
   
   Use the below command to generate the secret manifest file.
   
   ```bash
   oc create secret generic proxy-server-root-certificate --from-file=rootCertificate.pem=rootca.crt -n vxflexos --dry-run=client -oyaml > secret-proxy-server-root-certificate.yaml  
   ```

   Use the below command to create the secret. 

   ```bash
   oc apply -f secret-proxy-server-root-certificate.yaml  
   ``` 

   Verify the proxy server root certificate secret is created. 

   ```terminal 
   oc get secret -n vxflexos

   NAME                            TYPE     DATA   AGE
   karavi-authorization-config     Opaque   1      5m41s
   proxy-authz-tokens              Opaque   2      3h30m
   proxy-server-root-certificate   Opaque   1      6s 
   ```
  
   
5. ##### **Create the powerflex driver config secret** 

   <br>

   Create the config file.
  
   ```yaml 
   cat <<EOF> config.yaml
   - username: "xxxx"
     password: "xxxx"
     systemID: "3bc028ccdc1dd90f"
     endpoint: "https://localhost:9400"
     skipCertificateValidation: true
     mdm: "10.181.99.169"
     isDefault: true
   EOF 
   ``` 

   Use the below command to generate the secret manifest file.
   
   ```bash 
    oc create secret generic vxflexos-config --from-file=config=config.yaml -n vxflexos --dry-run=client -oyaml > secret-vxflexos-config.yaml 
   ```  

   Use the below command to create the secret. 

   ```bash 
   oc apply -f secret-vxflexos-config.yaml
   ``` 

   Verify the driver config secret is created.

   ```terminal 
   oc get secret -n vxflexos

   NAME                            TYPE     DATA   AGE
   karavi-authorization-config     Opaque   1      12m
   vxflexos-config                 Opaque   1      7s
   proxy-authz-tokens              Opaque   2      3h36m
   proxy-server-root-certificate   Opaque   1      6m27s 
   ```

6. ##### **Create the PowerFlex ContainerStorageModule.** 

   <br>

   Use the below command to create the CSM. 

   ```bash 
   oc apply -f csm-vxflexos.yaml
   ``` 

   Example: 
   ```yaml 
   cat <<EOF> csm-powerflex.yaml
   apiVersion: storage.dell.com/v1
   kind: ContainerStorageModule
   metadata:
     name: vxflexos
     namespace: vxflexos
   spec:
     driver:
       csiDriverType: "powerflex"
       configVersion: v2.13.0
       replicas: 2
       sideCars:
       - name: provisioner
         args: ["--volume-name-prefix=ocp01"]
       node:
         envs:
         - name: X_CSI_RENAME_SDC_ENABLED
           value: "true"
         - name: X_CSI_RENAME_SDC_PREFIX
           value: "sdc"
     modules:
     - name: authorization
       enabled: true
       components:
       - name: karavi-authorization-proxy
         envs:
         - name: "PROXY_HOST"
           value: "csm-authorization.example.com"
   EOF
   ``` 

   Verify the PowerFlex CSM is created.

   ```terminal 
   oc get csm -n vxflexos 

   NAME        CREATIONTIME   CSIDRIVERTYPE   CONFIGVERSION   STATE
   vxflexos    7m49s          powerflex       v2.13.0         Succeeded 
   ``` 

   Review the driver pods.

   ```terminal 
   oc get pod -n vxflexos

   NAME                                   READY   STATUS    RESTARTS        AGE
   vxflexos-controller-d5bcbc7dc-2pw95   6/6     Running   2 (3m13s ago)   3m15s
   vxflexos-controller-d5bcbc7dc-m58jf   6/6     Running   2 (3m4s ago)    3m15s
   vxflexos-node-b69vw                   3/3     Running   2 (3m1s ago)    3m15s
   vxflexos-node-blf6k                   3/3     Running   2 (3m ago)      3m15s
   vxflexos-node-mlvrf                   3/3     Running   2 (3m ago)      3m15s
   ``` 


 










