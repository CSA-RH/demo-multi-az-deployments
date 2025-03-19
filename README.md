# HA workloads demo. 

## Description

This play demonstrates how OpenShift's scheduling mechanisms work together to enable multi-zone deployments.

## Basic setup

Create projects:

```bash
oc new-project demo-ha
oc new-project demo-ha-load
```

In one terminal window, execute the following script, which will summarize the number of pods per node and will list them. 

```bash 
watch "echo "____"; echo "NODES";echo "----"; oc get pod -o=custom-columns=NODE:.spec.nodeName --no-headers | sort | uniq -c; echo "____"; echo "PODS"; echo "----"; oc get pods -owide"
```

Move to the demo-ha project

```bash
oc project demo-ha
```
Create a deployment with many replicas in the same node (picked randomly). 

```bash
NODE=$(oc get no -ojsonpath={.items} | jq -r '.[0].metadata.name') cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-load
  name: hello-load
  namespace: demo-ha-load
spec:
  replicas: 100
  selector:
    matchLabels:
      app: hello-load
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: hello-load
    spec:
      nodeName: $NODE
      containers:
      - image: openshift/hello-openshift
        imagePullPolicy: IfNotPresent
        name: hello-openshift
EOF
```

## Regular deployment without topology affinity rules

Create a regular deployment without constraints. The scheduler will try to evenly distribute the pods as they are Best Effort, but the pressure in one of the nodes caused by the previous operations will promote the other nodes, which have more free resources. This cannot be 100% accurate, as they are not request and limits in place. 

```bash
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello
  name: hello
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - image: openshift/hello-openshift
        imagePullPolicy: IfNotPresent
        name: hello-openshift
EOF
```

Scale the deployment to, for instance, 20 replicas, but you can play freely. 

```bash
oc scale deploy/hello --replicas=20
```

Delete the deployment:

```bash 
oc delete deploy/hello
```

## Topology example for AZs

Create the deployment with specific topology constraints to be applied at scheduling time. The following section will specify that we leverage the AZ tag to spread the replicas
```yaml
    ...(omitted)...
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: hello-az        
      containers:
    ...(omitted)...
```
```bash
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-az
  name: hello-az
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-az
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: hello-az
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: hello-az        
      containers: 
      - image: openshift/hello-openshift        
        imagePullPolicy: IfNotPresent
        name: hello-openshift
EOF
```

Scale the deployment to, for instance, 20 replicas, but you can play freely. 

```bash
oc scale deploy/hello-az --replicas=20
```

Delete the deployment:

```bash 
oc delete deploy/hello-az
```

## Topology example based on hostname

Create the deployment with specific topology constraints to be applied at scheduling time. The following section will specify that we leverage the hostname tag to spread the replicas

```bash
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-hostname
  name: hello-hostname
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-hostname
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: hello-hostname
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: hello-hostname
      containers:
      - image: openshift/hello-openshift
        imagePullPolicy: IfNotPresent
        name: hello-openshift
EOF
```
Scale the deployment to, for instance, 20 replicas, but you can play freely. 

```bash
oc scale deploy/hello-hostname --replicas=20
```

Delete de deployment: 
```bash
oc delete deploy/hello-hostname
```

## Cleanup

Delete projects
```bash
oc delete project demo-ha demo-ha-load
```
