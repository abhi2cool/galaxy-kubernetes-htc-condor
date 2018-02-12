# galaxy-kubernetes-htc-condor

## This is an attempt to replicate Galaxy docker swarm/compose implementation on kubernetes Deployed over AKS

### Pre-requisites

AKS kubernetes cluster with preferably two or more agent nodes

- The following links shall help with cluster creation and management
  - Deploy Kubernetes cluster for Linux containers
    - https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough

- Get the node info with 

```
[localhost]: kubectl get nodes
```

- Let the nodes be 
  - Node agent-0 
  - Node agent-1 
  
### Steps

- Designate one node for storage through kubectl label command 
 ```
 [localhost]:kubectl label nodes <Node agent-0> type=store
 ```
- ssh to the storage node
  - https://gist.github.com/tsaarni/624d5406e442f08fe11083169c059a68
  
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
  [Node agent-0]: sudo systemctl stop nfs-kernel-server.service
  [Node agent-0]: sudo systemctl start nfs-kernel-server.service
  ``` 
- ssh to all other nodes (no need to generate IPs for all these nodes just ssh from the storage node)
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
  
  if you get nginx 404 error do not panic and just restart galaxy server with
  ```
  [localhost]: kubectl delete pods galaxy --force --grace-period=0
  [localhost]: kubectl delete services galaxy-web --force --grace-period=0
  [localhost]: kubectl create -f galaxy-web.yaml
  [localhost]: kubectl create -f galaxy-webservice.yaml
  ```
  Again
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
 To assign a public static ip follow the steps in
https://gist.github.com/tsaarni/624d5406e442f08fe11083169c059a68
to create a public Ip
Delete existing service with
```
kubectl delete services galaxy-web --force --grace-period=0
```
add newly generated IP to the value of LoadBalancerip in galaxy-webservice2
and run
```
[localhost]:kubectl create -f galaxy-webservice2.yaml
```
