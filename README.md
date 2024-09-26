# Cluster API Control Plane Provider Kamaji
Kamaji is an Open-Source project offering hosted Kubernetes control planes.
the Control Plane is running in a management cluster as regular pods.


```
Prequirement :
1.  Make sure Kamaji cluster already create first.
    Documentation : https://kamaji.clastix.io/getting-started/
2.  Install clusterctl on Kamaji Cluster
    Documentation : https://release-1-7.cluster-api.sigs.k8s.io/user/quick-start#install-clusterctl
3.  Install Cluster API on Kamaji Cluster
    Documentation : https://release-1-7.cluster-api.sigs.k8s.io/user/quick-start
4.  Install Cluster API controlplane Kamaji on Kamaji Cluster
    Documentation : https://github.com/clastix/cluster-api-control-plane-provider-kamaji
    Documentation : https://github.com/clastix/cluster-api-control-plane-provider-kamaji/releases
```

## Install Cluster API on Kamaji Cluster
```
# Ref https://release-1-7.cluster-api.sigs.k8s.io/user/quick-start#install-clusterctl
- Export Kubeconfig:
export KUBECONFIG=/home/ubuntu/.kube/config

- Install clusterctl Binary:
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.7.6/clusterctl-linux-amd64 -o clusterctl
sudo install -o root -g root -m 0755 clusterctl /usr/local/bin/clusterctl
clusterctl version

- Initialize the Management cluster:
# Ref https://main.cluster-api.sigs.k8s.io/user/quick-start#initialize-the-management-cluster
export CLUSTER_TOPOLOGY=true

# This will install the latest cluster-api provider version
clusterctl init --infrastructure openstack

# Cluster API controlplane kamaji have compability matric that you can show the link below
# Ref https://github.com/clastix/cluster-api-control-plane-provider-kamaji?tab=readme-ov-file#-compatibility-matrix
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

## Install Cluster API controlplane Kamaji on Kamaji Cluster
```
# Ref https://github.com/clastix/cluster-api-control-plane-provider-kamaji/releases
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
### Prepare openstack clouds.yaml file

Login to horizon, klik Project > API Access > Downlaod OpenStack RF File > OpenStack clouds.yaml file
Copy file to ~/clouds.yaml, the file look like this

cat clouds.yaml
```
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

### Create secret in kamaji that store cloud.yaml file :
```
# Reference is here https://release-1-7.cluster-api.sigs.k8s.io/user/quick-start#required-configuration-for-common-providers
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

### Create workload cluster with .yaml file:
```
cat demome-capi.yaml 
kubectl apply -f demome-capi.yaml
```

Verification:
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
```

## Cluster Operation
### Get Kubeconfig:
```
clusterctl get kubeconfig demome -n kamaji-tcp > /home/ubuntu/cluster-prod/demome.kubeconfig

- Operation:
alias dm="kubectl --kubeconfig=/home/ubuntu/cluster-prod/demome.kubeconfig"
dm get nodes -o wide
dm get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
NAME                           TAINTS
demome-workers-9xpnb-jxmbm   [map[effect:NoSchedule key:node.cloudprovider.kubernetes.io/uninitialized value:true] map[effect:NoSchedule key:node.kubernetes.io/not-ready]]
demome-workers-9xpnb-m4j5q   [map[effect:NoSchedule key:node.cloudprovider.kubernetes.io/uninitialized value:true] map[effect:NoSchedule key:node.kubernetes.io/not-ready]]
demome-workers-9xpnb-vs7qr   [map[effect:NoSchedule key:node.cloudprovider.kubernetes.io/uninitialized value:true] map[effect:NoSchedule key:node.kubernetes.io/not-ready]]
```

### Install OCCM or Not
#### OPTION 1 : Not install OCCM
- If you dont want to install OCCM, you need to remove taint node.cloudprovider.kubernetes.io/uninitialized:NoSchedule- from all workers
- this action always need when new node are join to cluster, for example Upgrade.
otherwise, when install CNI or create deployment, all pod stuck in schduling

```
for i in $(dm get node --no-headers | awk '{print $1}'); do dm taint nodes $i node.cloudprovider.kubernetes.io/uninitialized:NoSchedule-; done
dm apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

#### OPTION 2 : Install OCCM
- If you install OCCM, taint node.cloudprovider.kubernetes.io/uninitialized:NoSchedule- will automaticaly remove when OCCM install.
- Create openstack application credential, reference https://docs.openstack.org/keystone/latest/user/application_credentials.html
- Reference : https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/openstack-cloud-controller-manager/using-openstack-cloud-controller-manager.md#config-openstack-cloud-controller-manager

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

## UPGRADE, SCALEUP and SCALEDOWN CLUSTER
### How to Upgrade Cluster ?
- When you create cluster, in .yaml file is create kind: OpenStackMachineTemplate that define flavor, image and sshKeyName
- You can't edit the yaml file and apply it, since OpenStackMachineTemplate are immutable, the recommended approach is to :
- https://release-1-7.cluster-api.sigs.k8s.io/tasks/upgrading-clusters#how-to-upgrade-the-underlying-machine-image
- Create new OpenStackMachineTemplate, Modify the values that need changing, such as instance type or image ID.

cat template/osmt-kubernetes-1.29.7.yaml
```
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

Create the new OpenStackMachineTemplate on the management cluster.
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
Answer: Edit .yaml file in section :
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
ANSWER: 
- Downgrading workers version is posible as if supported version by TCP and CAPI.
- But, downgrading TCP its self is not possible as Kamaji TCP is not allow to donwgrade.
- #cannot create or update TenantControlPlane: admission webhook "vtenantcontrolplane.kb.io" denied the request: unable to downgrade a TenantControlPlane from 1.30.2 to 1.29.7

### Can you change workers to different Availability Zone ?
ANSWER: YES
- on kind: OpenStackCluster, change the controlPlaneAvailabilityZones, network, subnets to different AZ,
- on kind: MachineDeployment, change failureDomain: AZ_Public01_JBBK to another AZ, 
- if you move from AZ that required flavor like GP.4C8G-intel or GP.4C8G-amd: you must create new OpenStackMachineTemplate with the new flavor

### Can you Upgrade/Downgrade the workers Flavor ?
ANSWER: Create new OpenStackMachineTemplate with your flavor: #Flavor desired.

### Is there anything I should pay attention to when edit OpenStackMachineTemplate ? 
ANSWER: when you edit OpenStackMachineTemplate to change flavor, image, keypair. Cluster API will create new instance, so it will change IP Address of new nodes.


## Machine Health Check
Ref https://release-1-7.cluster-api.sigs.k8s.io/tasks/automated-machine-management/healthchecking

### What is a MachineHealthCheck?
ANSWER :
- A MachineHealthCheck is a resource within the Cluster API which allows users to define conditions under which Machines within a Cluster should be considered unhealthy. 
A MachineHealthCheck is defined on a management cluster and scoped to a particular workload cluster.
- When defining a MachineHealthCheck, users specify a timeout for each of the conditions that they define to check on the Machine’s Node. 
If any of these conditions are met for the duration of the timeout, the Machine will be remediated. 
- By default, the action of remediating a Machine should trigger a new Machine to be created to replace the failed one, but providers are allowed to plug in more sophisticated external remediation solutions.


### Creating a MachineHealthCheck
Note : 
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
- If you accidentally delete instance from OpenStack Horizon. For example, if workload cluster contain 3 instance and you delete 2 instance.
- MachineHealthCheck will check the instace and found that instance is Missing, it will delpoy new instance and automatically joining to the cluster.
- In example i define nodeStartupTimeout: 10m, if create instance not join the cluster that you define, CAPI will create new instance for you.
- If new instance will state NodeReady for 10m [you not install CNI on it], it will create new instance to, as MachineHealthCheck detect the node as NodeReady


## Cluster Autoscaler on Cluster API

Ref https://release-1-7.cluster-api.sigs.k8s.io/tasks/automated-machine-management/autoscaling

### Using the Cluster Autoscaler
This section applies only to worker Machines. Cluster Autoscaler is a tool that automatically adjusts the size of the Kubernetes cluster based on the utilization of Pods and Nodes in your cluster. For more general information about the Cluster Autoscaler, please see the project documentation.

The following instructions are a reproduction of the Cluster API provider specific documentation from the Autoscaler project documentation.
Cluster Autoscaler on Cluster API
The cluster autoscaler on Cluster API uses the cluster-api project to manage the provisioning and de-provisioning of nodes within a Kubernetes cluster.

Note :
- You can create autoscaller after workload cluster is Ready, because autoscaller must have kubeconfig of workload cluster.
- One workload cluster required 1 autoscaller.
- You can use autoscaler for specific workload cluster only.
- This autoscaler can do scale-up and scale-down size of node.


### Create Cluster Autoscaler on Cluster API
```
# cat demome-autoscaler.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clusterautoscaler-demome
  namespace: kamaji-tcp
  labels:
    app: clusterautoscaler-demome
spec:
  selector:
    matchLabels:
      app: clusterautoscaler-demome
  replicas: 1
  template:
    metadata:
      labels:
        app: clusterautoscaler-demome
    spec:
      containers:
      - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.29.0
        name: clusterautoscaler-demome
        command:
        - /cluster-autoscaler
        args:
        # Define CAPI Cloud Provider
        - --cloud-provider=clusterapi
        - --node-group-auto-discovery=clusterapi:namespace=kamaji-tcp,clusterName=demome
        # Kubeconfig of Workload Cluster inside the pod of autoscaler
        - --kubeconfig=/mnt/kubeconfig/demome.kubeconfig
        - --clusterapi-cloud-config-authoritative
        - -v=1
        # Mount Kubeconfig of workload cluster
        volumeMounts:
        - mountPath: /mnt/kubeconfig
          name: kubeconfig
          readOnly: true
      serviceAccountName: clusterautoscaler-demome
      terminationGracePeriodSeconds: 10
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/control-plane
      volumes:
        - name: kubeconfig
          secret:
            secretName: demome-kubeconfig
            items:
              - key: value
                path: demome.kubeconfig
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: clusterautoscaler-demome-workload
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: clusterautoscaler-demome-workload
subjects:
- kind: ServiceAccount
  name: clusterautoscaler-demome
  namespace: kamaji-tcp
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: clusterautoscaler-demome-management
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: clusterautoscaler-demome-management
subjects:
- kind: ServiceAccount
  name: clusterautoscaler-demome
  namespace: kamaji-tcp
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: clusterautoscaler-demome
  namespace: kamaji-tcp
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: clusterautoscaler-demome-workload
rules:
  - apiGroups:
    - ""
    resources:
    - namespaces
    - persistentvolumeclaims
    - persistentvolumes
    - pods
    - replicationcontrollers
    - services
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - ""
    resources:
    - nodes
    verbs:
    - get
    - list
    - update
    - watch
  - apiGroups:
    - ""
    resources:
    - pods/eviction
    verbs:
    - create
  - apiGroups:
    - policy
    resources:
    - poddisruptionbudgets
    verbs:
    - list
    - watch
  - apiGroups:
    - storage.k8s.io
    resources:
    - csinodes
    - storageclasses
    - csidrivers
    - csistoragecapacities
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - batch
    resources:
    - jobs
    verbs:
    - list
    - watch
  - apiGroups:
    - apps
    resources:
    - daemonsets
    - replicasets
    - statefulsets
    verbs:
    - list
    - watch
  - apiGroups:
    - ""
    resources:
    - events
    verbs:
    - create
    - patch
  - apiGroups:
    - ""
    resources:
    - configmaps
    verbs:
    - create
    - delete
    - get
    - update
  - apiGroups:
    - coordination.k8s.io
    resources:
    - leases
    verbs:
    - create
    - get
    - update
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: clusterautoscaler-demome-management
rules:
  - apiGroups:
    - cluster.x-k8s.io
    resources:
    - machinedeployments
    - machinedeployments/scale
    - machines
    - machinesets
    - machinepools
    verbs:
    - get
    - list
    - update
    - watch
  - apiGroups:
    - infrastructure.cluster.x-k8s.io
    resources:
    - openstackmachinetemplates
    verbs:
    - get
    - list
    - update
    - watch
```

Apply Autoscaler
```
kubectl apply -f demome-autoscaler.yaml
```

Test Cluster Autoscaler on Cluster API
Note : Run this using kubeconfig of workload cluster

Deploy an example Kubernetes application. For example, the one used in the Kubernetes HorizontalPodAutoscaler Walkthrough.
```
kubectl apply -f https://k8s.io/examples/application/php-apache.yaml 
```
Increase the amount of replicas of the application to trigger a scale-up event
```
kubectl scale deployment php-apache --replicas 100
```

It will create 100 replica pods, and many pod are stuck at pending, autoscaller will monitor the pending of pods, and autoscaller the size of Instance using MachineDeployment

Decreate the amount of replicas of the application to trigger a scale-down event
```
kubectl scale deployment php-apache --replicas 10
```
As before with 100 replica pods with many nodes, when you resize replica to 10 pods, kube-scheduler will Terminating 90 pods,
As of that, there will many node with unused resource, autoscaler will detect it and resize the number of node to fit the size of Running pods.