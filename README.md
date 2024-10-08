# Cluster API Control Plane Provider Kamaji
In this guide i use [Cluster API Infrastructure Provider OpenStack](https://github.com/kubernetes-sigs/cluster-api-provider-openstack) and [ControlPlane Provider Kamaji](https://github.com/clastix/cluster-api-control-plane-provider-kamaji)

Kamaji is an Open-Source project offering hosted Kubernetes control planes, the Control Plane is running in a management cluster as regular pods.

`PREQUIREMENT :`
- Make sure [Kamaji](https://kamaji.clastix.io/getting-started/) Cluster Already create first.
- Install [clusterctl](https://release-1-7.cluster-api.sigs.k8s.io/user/quick-start#install-clusterctl) on Kamaji Cluster
- Install [Cluster API](https://release-1-7.cluster-api.sigs.k8s.io/user/quick-start) on Kamaji Cluster
- Install [Cluster API controlplane Kamaji](https://github.com/clastix/cluster-api-control-plane-provider-kamaji) on Kamaji Cluster

## Install [Cluster API](https://release-1-7.cluster-api.sigs.k8s.io/user/quick-start) on Kamaji Cluster
```
# Export Kubeconfig:
export KUBECONFIG=/home/ubuntu/.kube/config

# Install clusterctl Binary:
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.7.6/clusterctl-linux-amd64 -o clusterctl
sudo install -o root -g root -m 0755 clusterctl /usr/local/bin/clusterctl
clusterctl version

# Initialize the Management cluster:
# Reference https://main.cluster-api.sigs.k8s.io/user/quick-start#initialize-the-management-cluster
export CLUSTER_TOPOLOGY=true
clusterctl init --infrastructure openstack

# Cluster API controlplane kamaji have compability matric that you can show the link below
# Reference https://github.com/clastix/cluster-api-control-plane-provider-kamaji?tab=readme-ov-file#-compatibility-matrix
# CP provider   Cluster API               Kamaji      TCP API version
# v0.10.x       v1.5.x, v1.6.x, v1.7.x    ~v1.0.x     v1alpha1
# v0.10.x       v1.5.x, v1.6.x, v1.7.x    ~v1.0.x     v1alpha1
# v0.9.0        v1.5.x, v1.6.x            ~v0.6.x     v1alpha1
# v0.8.0        v1.5.x, v1.6.x            ~v0.5.x     v1alpha1
# v0.7.x        v1.5.x, v1.6.x            ~v0.4.0     v1alpha1
# v0.6.0        v1.5.x, v1.6.x            ~v0.4.0     v1alpha1

# So you will need to install component the meet compability matric instead of latest version
clusterctl init --core cluster-api:v1.7.5 --bootstrap kubeadm:v1.7.5 --control-plane kubeadm:v1.7.5 --infrastructure openstack:v0.10.2
```

## Install [Cluster API controlplane Kamaji](https://github.com/clastix/cluster-api-control-plane-provider-kamaji/releases) on Kamaji Cluster
```
cat ~/.cluster-api/clusterctl.yaml
providers:
- name: "kamaji"
  url: "https://github.com/clastix/cluster-api-control-plane-provider-kamaji/releases/v0.10.2/control-plane-components.yaml"
  type: "ControlPlaneProvider"

clusterctl init --control-plane kamaji
# Fetching providers
# Skipping installing cert-manager as it is already installed
# Installing Provider="control-plane-kamaji" Version="v0.10.2" TargetNamespace="kamaji-system"
```

## Create Workload Cluster
### Prepare OpenStack clouds.yaml file
Login to horizon, klik Project > API Access > Downlaod OpenStack RF File > OpenStack clouds.yaml file. <br/>
Copy file to ~/clouds.yaml, the file look like this

```
cat clouds.yaml
clouds:
  capi:
    auth:
      auth_url: https://endpoint/identity/v3
      username: "username"
      password: password
      project_id: projectid
      project_name: "projectname"
      user_domain_name: "Default"
    region_name: "RegionOne"
    interface: "public"
    identity_api_version: 3
```

### Create [Cloudconfig Secret](https://release-1-7.cluster-api.sigs.k8s.io/user/quick-start#required-configuration-for-common-providers) in kamaji that store cloud.yaml file
```
cat cloudconfig-clusterapi-gusriandi.yaml
---
apiVersion: v1
data:
  cacert: Cg==
  clouds.yaml: #REDACTED #BASE64 ENCODED
kind: Secret
metadata:
  labels:
    clusterctl.cluster.x-k8s.io/move: "true"
  name: cloudconfig-prj.clusterapi-user.gusriandi
  namespace: kamaji-tcp
---
kubectl apply -f cloudconfig-clusterapi-gusriandi.yaml
```

### Create workload cluster with .yaml file
```
cat demome-capi.yaml 
kubectl apply -f demome-capi.yaml
```

`Verification:`
```
kubectl get cl -A
NAMESPACE    NAME      CLUSTERCLASS   PHASE         AGE     VERSION
kamaji-tcp   demome                   Provisioned   7h26m

kubectl get openstackcluster -A
NAMESPACE    NAME                       CLUSTER   READY   NETWORK                                BASTION IP   AGE
kamaji-tcp   openstackcluster-demome    demome    true    f9881508-7de0-488e-a4ff-c437f4c8687f                7h27m

kubectl get ktcp -A
NAMESPACE    NAME          VERSION   INITIALIZED   READY   AGE
kamaji-tcp   tcp-demome    1.29.7    true          true    7h27m

kubectl get tcp -A
NAMESPACE    NAME          VERSION   STATUS   CONTROL-PLANE ENDPOINT   KUBECONFIG                     DATASTORE     AGE
kamaji-tcp   tcp-demome    v1.29.7   Ready    38.47.90.242:6443        tcp-demome-admin-kubeconfig    kamaji-etcd   7h27m

kubectl get md -A
NAMESPACE    NAME              CLUSTER   REPLICAS   READY   UPDATED   UNAVAILABLE   PHASE     AGE     VERSION
kamaji-tcp   demome-workers    demome    2          2       2         0             Running   7h27m   v1.29.7

kubectl get ms -A
NAMESPACE    NAME                    CLUSTER   REPLICAS   READY   AVAILABLE   AGE     VERSION
kamaji-tcp   demome-workers-k4ww7    demome    2          2       2           7h27m   v1.29.7

kubectl get ma -A
NAMESPACE    NAME                          CLUSTER   NODENAME                      PROVIDERID                                          PHASE     AGE     VERSION
kamaji-tcp   demome-workers-k4ww7-72vw7    demome    demome-workers-k4ww7-72vw7    openstack:///92d89223-681a-42ef-90c1-864de33808bc   Running   5h30m   v1.29.7
kamaji-tcp   demome-workers-k4ww7-xrq8x    demome    demome-workers-k4ww7-xrq8x    openstack:///8237f379-afbd-45be-8e5d-ee1a885a1077   Running   6h42m   v1.29.7

clusterctl describe cluster demome -n kamaji-tcp
NAME                                                                READY  SEVERITY  REASON  SINCE  MESSAGE                                                    
Cluster/demome                                                      True                     8h                                                                 
├─ClusterInfrastructure - OpenStackCluster/openstackcluster-demome                                                                                              
├─ControlPlane - KamajiControlPlane/tcp-demome                                                                                                                  
└─Workers                                                                                                                                                       
  └─MachineDeployment/demome-workers                                True                     6h46m                                                              
    └─2 Machines...                                                 True                     7h59m  See demome-workers-k4ww7-72vw7, demome-workers-k4ww7-xrq8x 
```

## Cluster Operation
### Get Kubeconfig
```
clusterctl get kubeconfig demome -n kamaji-tcp > /home/ubuntu/cluster-prod/demome.kubeconfig

alias dm="kubectl --kubeconfig=/home/ubuntu/cluster-prod/demome.kubeconfig"
dm get nodes -o wide
dm get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
NAME                           TAINTS
demome-workers-9xpnb-jxmbm   [map[effect:NoSchedule key:node.cloudprovider.kubernetes.io/uninitialized value:true] map[effect:NoSchedule key:node.kubernetes.io/not-ready]]
demome-workers-9xpnb-m4j5q   [map[effect:NoSchedule key:node.cloudprovider.kubernetes.io/uninitialized value:true] map[effect:NoSchedule key:node.kubernetes.io/not-ready]]
demome-workers-9xpnb-vs7qr   [map[effect:NoSchedule key:node.cloudprovider.kubernetes.io/uninitialized value:true] map[effect:NoSchedule key:node.kubernetes.io/not-ready]]
```

### Install OpenStack Cloud Controller Manager
#### OPTION 1 : Not install OCCM
If you dont want to install OCCM, you need to remove taint node.cloudprovider.kubernetes.io/uninitialized:NoSchedule- from all workers. <br/>
this action always need when new node are join to cluster, for example Upgrade.
otherwise, when install CNI or create deployment, all pod stuck in schduling. <br/>

```
for i in $(dm get node --no-headers | awk '{print $1}'); do dm taint nodes $i node.cloudprovider.kubernetes.io/uninitialized:NoSchedule-; done
dm apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

#### OPTION 2 : Install [OpenStack Cloud Controller Manager](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/openstack-cloud-controller-manager/using-openstack-cloud-controller-manager.md#config-openstack-cloud-controller-manager)
If you install OCCM, taint node.cloudprovider.kubernetes.io/uninitialized:NoSchedule- will automaticaly remove when OCCM install.

Create [OpenStack Application Credential](https://docs.openstack.org/keystone/latest/user/application_credentials.html) to Use with OCCM 

```
openstack application credential create occmcapi --unrestricted
cat cloud.conf 
---
[Global]
auth-url=https://endpoint/identity/v3
os-endpoint-type=public
application-credential-id=xxxxxx
application-credential-secret=xxxx-2bwOuxSolMwqNkYEJIH5LF7K9yrfYSmslYYtrHN3mt5cwkEgWOrihjBg
region=RegionOne
domain-name=Default
---

dm -n kube-system create secret generic cloud-config --from-file=cloud.conf
dm apply -f calico-occm/cloud-controller-manager-roles.yaml
dm apply -f calico-occm/cloud-controller-manager-role-bindings.yaml
dm apply -f calico-occm/openstack-cloud-controller-manager-ds.yaml
dm apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

### Create Pods and Service:
```
cat nginx-ds.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80       # Port exposed by the service
    targetPort: 80 # Port on the container

dm apply -f nginx-ds.yaml
dm get pods -o wide --selector app=nginx
dm get svc -o wide --selector app=nginx
curl publicip:31776
```

## Upgrade, Scale-up and Scale-down Workload CLUSTER
### How to [Upgrade Workload Cluster](https://release-1-7.cluster-api.sigs.k8s.io/tasks/upgrading-clusters#how-to-upgrade-the-underlying-machine-image) ?
When you create cluster, in .yaml file is create `kind: OpenStackMachineTemplate` that define flavor, image and sshKeyName. <br/>
You can't edit the yaml file and apply it, since `OpenStackMachineTemplate` are immutable, the recommended approach is to Create new `OpenStackMachineTemplate`, Modify the values that need changing, such as instance type or image ID.

```
cat template/osmt-kubernetes-1.29.7.yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: OpenStackMachineTemplate
metadata:
  name: osct-kubernetes-1.29.7
  namespace: kamaji-tcp
spec:
  template:
    spec:
      flavor: GP.4C8G-intel
      image:
        filter:
          name: "DKubes Worker v1.29.7"
      sshKeyName: rf-macos
```

Create the new `OpenStackMachineTemplate` on the management cluster.
```
kubectl apply -f template/osmt-kubernetes-1.29.7.yaml
# openstackmachinetemplate.infrastructure.cluster.x-k8s.io/osct-kubernetes-1.29.7 created

vim demome-capi.yaml
---
# Upgrade TCP
kind: KamajiControlPlane
spec:
  version: 1.29.7
# Upgrade Worker
kind: MachineDeployment
spec:
  replicas: 3
  template:
    spec:
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: OpenStackMachineTemplate
        name: osct-kubernetes-1.29.7
      version: 1.28.12
---
```

Apply for Upgrade
```
kubectl apply -f demome-capi.yaml
# cluster.cluster.x-k8s.io/demome unchanged
# kamajicontrolplane.controlplane.cluster.x-k8s.io/tcp-demome configured
# machinedeployment.cluster.x-k8s.io/demome-workers configured
# openstackmachinetemplate.infrastructure.cluster.x-k8s.io/osmt-demome unchanged
# kubeadmconfigtemplate.bootstrap.cluster.x-k8s.io/kubeadmconfigtemplate-demome configured
# The OpenStackCluster "openstackcluster-demome" is invalid: spec: Forbidden: cannot be modified
```

### How to scale up and scale down workers ?
`ANSWER :` Edit .yaml file in section :
```
vim demome-capi.yaml
---
kind: MachineDeployment
spec:
  replicas: 5  #Before 3
---
kubectl apply -f demome-capi.yaml
# If you scale up from 3 to 5, it will create new 2 instance for you, and joining it to cluster automaticaly.
# If you scale down from 3 to 2, it will remove 1 node from the cluster, and remove instance from openstack.
```

### Its posible to donwgrade TCP and workers ?
`ANSWER :`
- Downgrading workers version is posible as if supported version by TCP and CAPI.
- But, downgrading TCP its self is not possible as Kamaji TCP is not allow to donwgrade.
- #cannot create or update TenantControlPlane: admission webhook "vtenantcontrolplane.kb.io" denied the request: unable to downgrade a TenantControlPlane from 1.30.2 to 1.29.7

### Can you change workers to different Availability Zone ?
`ANSWER :` YES
- on kind: OpenStackCluster, change the controlPlaneAvailabilityZones, network, subnets to different AZ,
- on kind: MachineDeployment, change failureDomain: AZ_Public01_JBBK to another AZ, 
- if you move from AZ that required flavor like GP.4C8G-intel or GP.4C8G-amd: you must create new OpenStackMachineTemplate with the new flavor

### Can you Upgrade/Downgrade the workers Flavor ?
`ANSWER :` Create new OpenStackMachineTemplate with your flavor: #Flavor desired.

### Is there anything I should pay attention to when edit OpenStackMachineTemplate ? 
`ANSWER :` When you edit OpenStackMachineTemplate to change flavor, image, keypair. Cluster API will create new instance, so it will change IP Address of new nodes.



## Cluster API [Machine Health Check](https://release-1-7.cluster-api.sigs.k8s.io/tasks/automated-machine-management/healthchecking)

### What is a MachineHealthCheck?
- A MachineHealthCheck is a resource within the Cluster API which allows users to define conditions under which Machines within a Cluster should be considered unhealthy. 
A MachineHealthCheck is defined on a management cluster and scoped to a particular workload cluster.
- When defining a MachineHealthCheck, users specify a timeout for each of the conditions that they define to check on the Machine’s Node. 
If any of these conditions are met for the duration of the timeout, the Machine will be remediated. 
- By default, the action of remediating a Machine should trigger a new Machine to be created to replace the failed one, but providers are allowed to plug in more sophisticated external remediation solutions.


### Creating a MachineHealthCheck
`NOTE` : 
- Just add this MachineHealthCheck to .yaml file. 
- You can add this add first time deploy the workload cluster, or after workload cluster is create by adding this to .yaml file and apply it.

```
vim demome-capi.yaml  # Add this line to exsiting yaml file to bellow
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineHealthCheck
metadata:
  name: mhcs-demome-node-unhealthy-300s
  namespace: kamaji-tcp
spec:
  clusterName: demome
  # Optional - maxUnhealthy prevents further remediation if the cluster is already partially unhealthy
  maxUnhealthy: 100%
  # Optional - nodeStartupTimeout determines how long a MachineHealthCheck should wait for a Node to join the cluster, before considering a Machine unhealthy.
  # Defaults to 10 minutes if not specified. Set to 0 to disable the node startup timeout.
  # Disabling this timeout will prevent a Machine from being considered unhealthy when the Node it created has not yet registered with the cluster. 
  # This can be useful when Nodes take a long time to start up or when you only want condition based checks for Machine health.
  nodeStartupTimeout: 10m
  # selector is used to determine which Machines should be health checked
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: demome
  # Conditions to check on Nodes for matched Machines, if any condition is matched for the duration of its timeout, the Machine is considered unhealthy
  unhealthyConditions:
  - type: Ready
    status: Unknown
    timeout: 300s
  - type: Ready
    status: "False"
    timeout: 300s
```

### How to test Machine health check ?
- If you accidentally delete instance from OpenStack via Horizon/OpenStack Client. For example, if workload cluster contain 3 instance and you delete 2 instance.
- MachineHealthCheck will check the instace and found that instance is Missing, it will redelpoy new instance and automatically join it to cluster.
- In example define `nodeStartupTimeout: 10m`, if created instance not join the cluster, CAPI will recreate new instance for you.
- If new instance will state `NodeReady` for 10m [you not install CNI on it], it will recreate new instance to, as `MachineHealthCheck` detect the node as `NotReady`


## [Cluster Autoscaler](https://release-1-7.cluster-api.sigs.k8s.io/tasks/automated-machine-management/autoscaling) on Cluster API
### Using the Cluster Autoscaler
This section applies only to worker Machines. Cluster Autoscaler is a tool that automatically adjusts the size of the Kubernetes cluster based on the utilization of Pods and Nodes in your cluster. For more general information about the Cluster Autoscaler, please see the project documentation.

The following instructions are a reproduction of the Cluster API provider specific documentation from the Autoscaler project documentation.
Cluster Autoscaler on Cluster API
The cluster autoscaler on Cluster API uses the cluster-api project to manage the provisioning and de-provisioning of nodes within a Kubernetes cluster.

`NOTE` :
- You can create autoscaller after workload cluster is Ready, because autoscaller must have kubeconfig of workload cluster.
- One workload cluster required 1 autoscaller.
- You can use autoscaler for specific workload cluster only.
- This autoscaler can do scale-up and scale-down size of node.


### Create [Cluster Autoscaler on Cluster API](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/clusterapi/examples/deployment.yaml)
```
cat demome-autoscaler.yaml
kubectl apply -f demome-autoscaler.yaml
```

#### Test Cluster Autoscaler on Cluster API [Scale UP]
Note : Run this using kubeconfig of workload cluster

Deploy an example Kubernetes application. For example, the one used in the Kubernetes HorizontalPodAutoscaler Walkthrough.
```
kubectl apply -f https://k8s.io/examples/application/php-apache.yaml 
```
Increase the amount of replicas of the application to trigger a scale-up event
```
kubectl scale deployment php-apache --replicas 100
```

It will create 100 pods, as many pod are stuck at pending, autoscaller will monitor the pending of the pods, and autoscaller will resize of Instance using `MachineDeployment`

#### Test Cluster Autoscaler on Cluster API [Scale Down]
Decrease the amount of replicas of the deployment to trigger a scale-down event
```
kubectl scale deployment php-apache --replicas 10
```
As before with 100 replica pods with many nodes, when you resize replica to 10 pods, kube-scheduler will Terminating 90 pods. <br/>
As of that, there will many node with unused resource, autoscaler will detect it and resize the number of node and Scale down the cluster.


## Cluster API with Multi Availability Zone Workers Node
```
cat multiazcluster-alldc-capi.yaml
```

```
kubectl apply -f multiazcluster-alldc-capi.yaml
cluster.cluster.x-k8s.io/cluster-alldc created
kamajicontrolplane.controlplane.cluster.x-k8s.io/ktcp-cluster-alldc created
openstackcluster.infrastructure.cluster.x-k8s.io/osc-dc4-cluster-alldc created
kubeadmconfigtemplate.bootstrap.cluster.x-k8s.io/kubeadmconfigtemplate-cluster-alldc created
openstackmachinetemplate.infrastructure.cluster.x-k8s.io/osmt-dc2-cluster-alldc-2c4g-intel created
openstackmachinetemplate.infrastructure.cluster.x-k8s.io/osmt-dc2-cluster-alldc-8c16g-intel created
openstackmachinetemplate.infrastructure.cluster.x-k8s.io/osmt-dc3-cluster-alldc-2c4g-intel created
openstackmachinetemplate.infrastructure.cluster.x-k8s.io/osmt-dc4-cluster-alldc-2c4g-intel created
openstackmachinetemplate.infrastructure.cluster.x-k8s.io/osmt-dc4-cluster-alldc-8c16g-amd created
machinedeployment.cluster.x-k8s.io/md-dc2-cluster-alldc-2c4g-intel created
machinedeployment.cluster.x-k8s.io/md-dc2-cluster-alldc-8c16g-intel created
machinedeployment.cluster.x-k8s.io/md-dc3-cluster-alldc-2c4g-intel created
machinedeployment.cluster.x-k8s.io/md-dc4-cluster-alldc-2c4g-intel created
machinedeployment.cluster.x-k8s.io/md-dc4-cluster-alldc-8c16g-amd created
machinehealthcheck.cluster.x-k8s.io/mhcs-cluster-alldc-node-unhealthy-5m created
```

### Verification
```
kubectl get machine

NAME                                           CLUSTER         NODENAME                                       PROVIDERID                                          PHASE     AGE     VERSION
md-dc2-cluster-alldc-2c4g-intel-vwvc6-6qqzk    cluster-alldc   md-dc2-cluster-alldc-2c4g-intel-vwvc6-6qqzk    openstack:///45f4a0b1-0c77-47f5-a4d5-f02089f7e427   Running   43m     v1.29.7
md-dc2-cluster-alldc-2c4g-intel-vwvc6-wvd9t    cluster-alldc   md-dc2-cluster-alldc-2c4g-intel-vwvc6-wvd9t    openstack:///7b978e7c-c530-4381-bf13-561cec71a121   Running   43m     v1.29.7
md-dc2-cluster-alldc-8c16g-intel-244dg-8qm6q   cluster-alldc   md-dc2-cluster-alldc-8c16g-intel-244dg-8qm6q   openstack:///438ac373-91d4-429d-86d6-98b691ec230f   Running   43m     v1.29.7
md-dc2-cluster-alldc-8c16g-intel-244dg-svvx7   cluster-alldc   md-dc2-cluster-alldc-8c16g-intel-244dg-svvx7   openstack:///d88e4ec8-1fb2-426b-9870-a8b5a32b7f9f   Running   43m     v1.29.7
md-dc3-cluster-alldc-2c4g-intel-f75hc-9rlps    cluster-alldc   md-dc3-cluster-alldc-2c4g-intel-f75hc-9rlps    openstack:///ae6ee7df-e0f7-4df8-97e3-86d57c7b664d   Running   43m     v1.29.7
md-dc3-cluster-alldc-2c4g-intel-f75hc-c4d6b    cluster-alldc   md-dc3-cluster-alldc-2c4g-intel-f75hc-c4d6b    openstack:///3ebb37a9-3668-4fc3-bbed-6c5bc77cae02   Running   43m     v1.29.7
md-dc4-cluster-alldc-2c4g-intel-62jg5-c7mvd    cluster-alldc   md-dc4-cluster-alldc-2c4g-intel-62jg5-c7mvd    openstack:///26e34941-34c5-4ef6-8650-1e19e4986caa   Running   43m     v1.29.7
md-dc4-cluster-alldc-2c4g-intel-62jg5-vb46k    cluster-alldc   md-dc4-cluster-alldc-2c4g-intel-62jg5-vb46k    openstack:///665268ce-2156-4364-809a-e1ce6c248574   Running   43m     v1.29.7
md-dc4-cluster-alldc-8c16g-amd-7kwb8-7t4dn     cluster-alldc   md-dc4-cluster-alldc-8c16g-amd-7kwb8-7t4dn     openstack:///5a933a2e-b705-4a83-8854-4127d0d62a24   Running   43m     v1.29.7
md-dc4-cluster-alldc-8c16g-amd-7kwb8-w44ss     cluster-alldc   md-dc4-cluster-alldc-8c16g-amd-7kwb8-w44ss     openstack:///1b7f7723-6846-4318-a471-17de07663b80   Running   43m     v1.29.7
```

```
clusterctl describe cluster cluster-alldc
NAME                                                              READY  SEVERITY  REASON  SINCE  MESSAGE                                                                                        
Cluster/cluster-alldc                                             True                     61m                                                                                                    
├─ClusterInfrastructure - OpenStackCluster/osc-dc4-cluster-alldc                                                                                                                                  
├─ControlPlane - KamajiControlPlane/ktcp-cluster-alldc                                                                                                                                            
└─Workers                                                                                                                                                                                         
  ├─MachineDeployment/md-dc2-cluster-alldc-2c4g-intel             True                     58m                                                                                                    
  │ └─2 Machines...                                               True                     60m    See md-dc2-cluster-alldc-2c4g-intel-vwvc6-6qqzk, md-dc2-cluster-alldc-2c4g-intel-vwvc6-wvd9t    
  ├─MachineDeployment/md-dc2-cluster-alldc-8c16g-intel            True                     58m                                                                                                    
  │ └─2 Machines...                                               True                     60m    See md-dc2-cluster-alldc-8c16g-intel-244dg-8qm6q, md-dc2-cluster-alldc-8c16g-intel-244dg-svvx7  
  ├─MachineDeployment/md-dc3-cluster-alldc-2c4g-intel             True                     58m                                                                                                    
  │ └─2 Machines...                                               True                     60m    See md-dc3-cluster-alldc-2c4g-intel-f75hc-9rlps, md-dc3-cluster-alldc-2c4g-intel-f75hc-c4d6b    
  ├─MachineDeployment/md-dc4-cluster-alldc-2c4g-intel             True                     58m                                                                                                    
  │ └─2 Machines...                                               True                     60m    See md-dc4-cluster-alldc-2c4g-intel-62jg5-c7mvd, md-dc4-cluster-alldc-2c4g-intel-62jg5-vb46k    
  └─MachineDeployment/md-dc4-cluster-alldc-8c16g-amd              True                     58m                                                                                                    
    └─2 Machines...                                               True                     61m    See md-dc4-cluster-alldc-8c16g-amd-7kwb8-7t4dn, md-dc4-cluster-alldc-8c16g-amd-7kwb8-w44ss 
```