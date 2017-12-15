# galaxy-kubernetes-htc-condor
## This is an attempt to replicate Galaxy docker swarm/compose implementation on kubernetes Deployed over ACS
### Pre-requisites
ACS kubernetes clusrter with one master and preferably two or more agent nodes
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
 [localhost]:kubectl label pods <Node agent-0> type=store
 ```
- ssh to the storage node
  - To ssh to any node, first ssh to master node with the IP available on the web portal (create user and add password to all nodes through the portal, makes things easier :P)
  - then ssh to any desired node with
  ```
  [Node master-0]:-# ssh username@Node agent-0
  ```
  - make directory export with
  ```
  [Node agent-0]: mkdir -p /export
  ```
    - add "/export" to list of directories eligible for nfs mount with both read and write privileges
    - You can configure the directories to be exported by adding them to the /etc/exports file. For example:
    ```
    /ubuntu  *(ro,sync,no_root_squash)
    /export    *(rw,sync,no_root_squash)
    ```
- ssh to all other nodes
```
[Node any]:-# ssh username@Node agent-(all except 0)
[Node agent-(all except 0)]: mkdir -p /export
[Node agent-(all except 0)]: sudo mount <Node agent-0>:/export /export
```
- expose required pods(services) with kubectl expose
```
[localhost]: kubectl expose pod galaxy-postgres
```
- create galxy webserver with command
```
[localhost]: kubectl create -f galaxy-web.yaml
```
- ssh to storage node using same steps above 
```
[Node agent-0]: cd /export
[Node agent-0]: chmod 777 -R * 
```
  - this step needs to be done in order to get rid of the permission denied error

#### Setup Htcondor
- create galaxy-htcondor with
```
[localhost]: kubectl create -f galaxy-htcondor.yaml
```
- expose service using
```
[localhost]: kubectl expose pod galaxy-htcondor
```
- create htcondor-executor and htcondor-executor-big with
```
[localhost]: kubectl create -f galaxy-htcondor-executor.yaml -f galaxy-htcondor-executor-big.yaml
```
- get shell to galaxy-htcondor container
```
[localhost]:kubectl exec -it galaxy-htcondor -- /bin/bash
```
  - edit file /etc/hosts
  ```
  [root@galaxy-htcondor]: vi /etc/hosts
  ```
  - add 127.0.0.1   galaxy-htcondor
  - restart condor with 
  ```
    condor_restart
  ```
