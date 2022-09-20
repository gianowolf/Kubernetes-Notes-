# 14: The Kubernetes API

## High Level View

- K8s is API centric: everything is about the API, everything goes through the API and API server.
- Clients send reqs to K8s to manage objects -> you will use kubectl but you can craft the requests or use API testing and dev tools to generate them. __No matter how the requests is generated if they go authenticated and authorized.__

### JSON Serialization

_Serialization is the process of converting an object into a string or stream of bytes, so it can be sent over a network and persisted to a data store._

_Kubernetes serializes objects_, such Pods or Services, as __JSON strings to be sent over HTTP__. The process happens in both directions.

The Kubernetes serialization is the process of converting an object into a JSON string to be sent over HTTP and persisted to the cluster store usually based on the etcd database.

Kubernetes also supps Protobuf serialization schema
- Faster, Scales better, efficient.
- No user-friendly

## API Server

API Server exposes the API over a secure RESTful interface using HTTPS. 

- All ```kubectl``` commands go to the API server
- All noide Kubelets watch the API server for new tasks and report status to the API server
- All control plane services communicate with each other via API Server

API Server is a Kubernetes control plane service. If you buikld and manage your own Kubernetes cluster, you need to make sure the control plane is highly-available and has enough performance to keep API server up-and-running and responding quickly to requests.

Tha main job of the API server is to make the API available to clients inside and outside the cluster.

The API is RESTful. IT accepts CRUS-style requests via standard HTTP methods. CRUD-style are simple _create, read, update, delete_ operations, and the common HTTP methods include POST, GET, PUT, PATCH and DELETE.

| HTTP method | K9s CRUD | kubectl example | 
| -- | -- | -- |
| POST | create | ```kubectl create -f <filename>``` | 
| GET | get, list, watch | ```kubectl get pods``` | 
| PUT/PATCH | update | ```kubectl edit deployment <deployment-name>``` | 
| DELETE | delete | ```kubectl delete ingress <ig-name>``` |

- API server common exposed on port 443 or 6443. It can be configured it..
- ```kubectl cluster-info``` to get control plane, backend, KubeDNS, metrics, etc.

### REST and RESTful

- REST: REpresentational State Transfer is the standard for communicating with web-based APIs. Systems that use REST are referred as RESTful.
- REST requests comprise a __verb__ and a __path__ to a resource.
  - verbs are related to actions and are the standard HTTP methods 
  - paths are URI path to the resource in the API

```sh
kubectl proxy --port 9000 &
curl -v http://localhost:9000/api/v1/namespaces/shield/pods
```

## API

The API is where all Kubernetes resources are defined. Large, modular and RESTful. 

At highest level, there are two types of API group
- core group
- named groups

### core API group

- mature objects that were created in early K8s days before the API was divided into groups.
- Pods, nodes, Services, Secrets, ServiceAccounts
- located in api/v1

| Resource | REST Path |
| -- | -- | 
| Pods | /aapi/v1/namespaces/{namespace}/pods/ | 
| Services | /api/v1/namespaces/{namespace}/services/ |
| Nodes | /api/v1/nodes/ |
| Namespaces | /api/v1/namepsaces |

### Named API groups

- Named as subgroups
- each new resources added to subgroups/named groups
- apps group: resources that manage application workload: Deployments, ReplicaSets, DaemonSets, StatefulSets.
- networking.k8s.io: Ingresses, Ingress Classes, Network Policies, network-related resources.
- live below the /apis/{group-name}/{version}/path

```sh
# See which resources are available on the cluster
kubectl api-resources

# show which API versions are supported on your cluster
kubectl api-versions
```

## Accessing the API

- kubectl
- API development tools
- commands: curl, wget, Invoke-WebRequest
- Web browser 

```sh
# Simplest way
kubectl proxy --port 9000 &

# explore the API
curl http://localhost:9000/api
curl httpL//localhost:9000/apis
```

### Resource Deprecation

- Stable/GA: are expected to be long-lived. When deprecated, supported for 12 months or 3 releases, whichever's longest 
- Beta: have 9-month windown to eigher graduate to stable or release a newer beta version.

### eXTENDING THE api

- You can extend Kubernetes by adding own resources and controllers
- Example 3rd parties extending the K8s API can be seen in the storage world where vendors expose advanced freatures

Kubernetes has a CustomResourceDefinition (CRD) object that lets you create new resources in the API that look, smell and feel like native K8s resources.