# 12: StatefulSets

_A **stateful application** is one that creates and saves valuable data._

An example might be an app that saves data about client sessions and uses it for future sessions. Other examples include databases and data stores.

## StatefulSets Theory

- Deployments and StatefulSets: c
    - controllers that loop-watching state of the cluster and comparing observed-desired states.
    - manage pods and bring self-healing, scaling, rollouts, etc.
- __StatefulSets__ guarantee:
  - Predictable and persistent Pod names
  - Predictable and persistent DNS hostnames
  - Predictable and persistent volume bindings
- The three properties form the __state__ of a Pod or __sticky ID__.
- StatefulSets persists state across failures, scaling, scheduling, etc
- Stateful sets are ideal for apps that require _unique reliable Pods_
- __Pods managed by a StatefulSet will be replaced by new Pods with the exact Pod name__.

```yaml
apiVersion: apps/v1
kind: StatefilSet
metadata:
    name: tkb-sts
spec:
    selector: matchLabels:
        app: mongo
    serviceName: "tkb-sts"
    replicas: 3
    template:
        metadata:   
            labels: 
                app: mongo
        spec:
            containers:
            -   name:   ctr-mongo
                image: mongo:latest
                ...
```

- name: tkb-sts
- defines three Pod replicas running mongo:latest image
- StatefulSet controller monitors the state of the cluster making sure observed matches desired.

### Pod Naming

- Format: <StatefulSetName>-<Integer>
  - integer: starting from 0
  - in yaml example: tkb-sts-0 to tkb-sts-2
- Needs to be __valid DNS name__.

### Creation and Deletion

- Ordered way 
- StatefulSet create on Pod at time, waiting for previous to be _running and ready_
  - different from ReplicaSet controller way
- Scaling up or down follows the same rules
- Will not be terminated in parallel
- ```terminationGracePeriodSeconds``` to manage the scaling down delay

__Deleting StatefulSet object not terminate Pods in order -> you may want to scale StatefulSet to 0 replicas before deleting it__

### StatefulSets and Volumes

- In a StetefulSet Pod any volume is created and named in a special way
  - tkb-sts-0 -> vol-tkb-sts-0
  - tkb-sts-X -> vol-tkb-sts-X
- __Volume has separate lifecycles to Pods__ -> allowing to survive Pod failures and termination ops.
  - This allow replace Pods to attach the the exact same storage

### Network ID and Headless Services