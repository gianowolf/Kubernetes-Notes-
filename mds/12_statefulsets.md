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
- ```terminationGracePeriodSeconds``` to manage the scgitaling down delay

__Deleting StatefulSet object not terminate Pods in order -> you may want to scale StatefulSet to 0 replicas before deleting it__

### StatefulSets and Volumes

- In a StetefulSet Pod any volume is created and named in a special way
  - tkb-sts-0 -> vol-tkb-sts-0
  - tkb-sts-X -> vol-tkb-sts-X
- __Volume has separate lifecycles to Pods__ -> allowing to survive Pod failures and termination ops.
  - This allow replace Pods to attach the the exact same storage

### Network ID and Headless Services

- As result of statefulStets other parts of the app, or other apps, may need to connect directly to individual Pods.
  - Stateful sets use a __headless Service__ to create predictable DNS hostnames for very Pod replica
  - StatefulSets use a __headless Service__ to create predictable DNS hostnames

```yaml
apiVersion: v1
kind: Service
metadata:
    name: mong-prod
spec: 
    clusterIP: None # HEADLESS SERVICE
    selector:
        app: mongo
        env: prod
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
    name: sts-mongo
spec: 
    serviceName: mongo-prod # Governing Service
```

__Headless Service__ is a regular Kubernetes Service object __without IP address__ (clusterIP: None) => Becomes a StatefulSet governing service when in the SS config spec.serviceName is listed.

- The service will create DNS SRV records for _every Pod_ matchign the label selector of the headless Service
- Other Pods and Apps can find them using NDS lookups against the name of the headless service.

<div class="pagebreak"> </div>

## Hands-On with StatefulSets

you will deploy:
- StorageClass
- HeadlessService
- StatefulSet

Not intented as prod-grade config. YAML file example designed for GKE.
### StorageClass

- StatefulSets that use volumes need to be able to create them dynamically. To do this you need
  - SC: StorageClass
  - PVC: PersistentVolumeClaim

Defines a StorageClass object clled ```flash``` that will dynamically provision SSD volumes (```type=pd-ssd```) from Google CLould GKE persistent disk CSI driver (```pd.csi.storage.gke.io```). Only work on Kubernetes clusters on GCP or GKE with CSI driver enabled.
  
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: flash
provisioner: pd.csi.storage.gke.io
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: pd-ssd
```

```sh
kubectl apply -f gcp-sc.yml
kubectl get sc
```

### Headless Service

- A Service without an IP address: the head is the stable IP address and the tail is the list of POds it sends traffic to.
- Clients can use DNS to reach Pods directly indtead of via Service's Cluster IP

The follows file describes a headless Service with no IP address

```yaml
apiVersion: v1
kind: Service
metadata: 
  name: dullahan
  labels:
    app: web
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: web
# The difference to a regular Service is None clusterIP Address
```

When combined with a StatefulSet, headless Service create predictable stable DNS entries for every Pod matching the StatefulSet's _label selector_.

```sh
kubectl apply -f headless-svc.yml
kubectl get svc
```

### Deploy StatefulSet

With the StorageClass and headless Service define the StatefulSet (non production grade deployment intented)

```yaml
apiVErsion: apps/v1
kind: StatefulSet
metadata:
  name: tkb-sts
spec:
  replicas: 3
  selector: 
    matchLabels:
      app: web
  serviceName: "dullahan"
  template:
    metadata:
      labels:
        app: web
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: ctr-web
        image: nginx:latest
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: webroot
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: webroot
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "flash"
      resources:
        requests:
          storage: 1Gi
```

- name: the ```spec.replicas``` will be named tkb-sts-x. They will be created in numerical order, and StatefulSet controller will wait for each replica to be running and ready.
- ```spec.serviceName``` designates the _governing Service_. **This is the name of the headless Service created in the previous step**. It will create the DNS SRV for each replica.
- ```spec.template``` defines Pod template that will be used to stamp out Pod replicas. (container image, ports, etc)
- ```spec.volumeClaimTemplates``` is for create a PVC. **Each pod in a StatefulSet needs its own unique storage. Each one needs its own PVC**. __A _VolumeClaimTemplate_ dynamically creates a PVC each time the StatefulSet controller spawns a new Pod replica.__

Example volumeClaimTemplate: defines a claim template called ```webroot``` requesting 10GB volume from ```flash``` StorageClass

```yaml
volumeClaimTemplates:
- metadata:
  - name: webroot
- spec:
  - accessModes: [ "ReadWriteOnce"]
    sotrageClassName: "flash"
    resources: 
      requests:
        storage: 10Gi
```

```sh
# Watch StatefulSet up to 3 running replicas
kubectl get sts --watch
NAME  READY AGE
tkb-sts 0.3 14s

# check the PVCs
kubectl get pvc
```

- Each PVC is created for each StatefulSet Pod replica.
- the name of each PVC is based on the name of the StatefulSet and the Pod associated with
  - tkb-sts-0 <> webroot-tkb-sts-0
- The StatefulSet PODs are headless Service so we should one DNS SRV Record for each pod.

### DNS and Peer Discovery

- By default K8s places all objects within ```cluster.local``` DNS subdomain
- Within that domain, K8s constructurs subdomains as:
  - ```<object-name>.<service-name>.<namespace>.svc.cluster.local

```sh
# Testing
kubectl apply -f jump-pod.yml # Pod with DNS dig utility pre-installed
kubectl exec -it jump-pod --bash

# as root
dig SRV service-name.namespace.svc.cluster.local
# Returns fully qualified DNS names of each Pod
```

## Scaling StatefulSets

- Scaling up -> a Pod and a PVC is created
- Scaling down -> Pod is terminated but the PVC is not.
  - Future scale-up opertations only need to create a new Pod, then is connected to the Surviving PVC
  - This is to protect the resiliency of the app and integrity of any data

## StatefulSet Rollouts

- Stateful Sets support rolling updates.

1. Update the image in the YAML
2. re-post to the API server
3. is Authenticated and authorized
4. Controller replaces old Pods with new ones.
5. Controller waits for each new Pod to be running and ready
  
For more info run ```kubectl explain sts.spec.updateStrategy```.

## Deleting StatefulSets

1. Scale StatefulSet to 0 replicas and confirm the operation ```kubectl scale sts tkb-sts --replicas=0```
2. ```kubectl get sts tkb-sts```
3. ```kubectl delete sts tkb-sts```
4. ```kubectl delete -f sts.yml```

At this point SS is deleted, but volumes, StorageClass and headless Service still exists and should be deleted manually.