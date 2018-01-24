# galaxy-kubernetes-htc-condor

## This is an attempt to replicate Galaxy docker swarm/compose implementation on kubernetes Deployed over ACS

### Pre-requisites

ACS kubernetes cluster with one master and preferably two or more agent nodes

- The following links shall help with cluster creation and management
  - Deploy Kubernetes cluster for Linux containers
    - https://docs.microsoft.com/en-us/azure/container-service/kubernetes/container-service-kubernetes-walkthrough
  - Using the Kubernetes web UI with Azure Container Service
    - https://docs.microsoft.com/en-us/azure/container-service/kubernetes/container-service-kubernetes-ui

- Get the node info with 

```
[localhost]: kubectl get nodes
```

- Let the nodes be 
  - Node master-0 
  - Node agent-0 
  - Node agent-1 
  
### Steps

- Designate one node for storage through kubectl label command 
 ```
 [localhost]:kubectl label nodes <Node agent-0> type=store
 ```
- ssh to the storage node
  - To ssh to any node, first ssh to master node with the IP available on the azure portal (create user and add password to all nodes through the azure portal, makes things easier :P) then ssh to any desired node with
    ```
    [Node master-0]:-# ssh username@Node agent-0
    ```
  
- make directory export with
  ```
  [Node agent-0]: mkdir -p /export
  [Node agent-0]: chown nobody:nogroup /export
  ```
  
- install NFS server 
  ```
  [Node agent-0]: sudo apt install nfs-kernel-server
  ```
 
- add "/export" to list of directories eligible for nfs mount with both read and write privileges
    - You can configure the directories to be exported by adding them to the /etc/exports file. For example:
      ```
      /ubuntu  *(ro,sync,no_root_squash)
      /export  *(rw,sync,no_root_squash)
      ```
- edit /etc/hosts.allow
  - add
  ```
  ALL: ALL
  ```
    
- start service with
  ```
  [Node agent-0]: sudo systemctl start nfs-kernel-server.service
  ```
 
- ssh to all other nodes
  ```
  [Node any]:-# ssh username@Node agent-(all except 0)
  [Node agent-(all except 0)]: mkdir -p /export
  [Node agent-(all except 0)]: sudo mount <Node agent-0>:/export /export
  ```
  
- Install helm chart with
  ```
  [localhost]: helm install galaxy
  ```
  
- ssh to storage node using same steps above, then, in order to get rid of the permission denied error:
  ```
  [Node agent-0]: cd /export
  [Node agent-0]: chmod 777 -R * 
  ```
  
- check if galaxy is running with
  ```
  [localhost]: kubectl port-forward galaxy 8080:80
  ```
  and then browse to localhost:8080

#### Setup Htcondor

- get shell to galaxy-htcondor container
```
[localhost]:kubectl exec -it galaxy-htcondor -- /bin/bash
```
  - edit file /etc/hosts
  ```
  [root@galaxy-htcondor]: vi /etc/hosts
  ```
  - add 127.0.0.1   galaxy-htcondor
- get shell to galaxy-htcondor-executor container
```
[localhost]:kubectl exec -it galaxy-htcondor-executor -- /bin/bash
```
  - edit file /etc/hosts
  ```
  [root@galaxy-htcondor-executor]: vi /etc/hosts
  ```
  - add 127.0.0.1   galaxy-htcondor-executor 
- get shell to galaxy container
```
[localhost]:kubectl exec -it galaxy --container=galaxy  -- /bin/bash
```
  - edit file /etc/condor/condor_config.local
  ```
  [root@galaxy-htcondor]: vi /etc/condor/condor_config.local
  ```
  - add the following lines
  ```
  HOSTALLOW_READ = *
  HOSTALLOW_WRITE = *
  HOSTALLOW_NEGOTIATOR = *
  HOSTALLOW_ADMINISTRATOR = *
  ```
  - restart condor with 
  ```
    [root@galaxy]:condor_restart
  ```
 - to assign a static public IP to galaxy and galaxy-proftpd server run 
 ```
 [localhost]: kubectl expose pod galaxy --type=LoadBalancer
 [localhost]: kubectl expose pod galaxy-proftpd --type=LoadBalancer
 ```
