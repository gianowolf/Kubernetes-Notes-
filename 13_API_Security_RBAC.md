# 13: API Security and RBAC 

API Server clients:
- Operators and developers using ```kubectl```
- Pods, Kubelets, Control plane services
- etc...

The credentials request to the API serveru uses TLS.

The user __grant-ward__ uses a kubectl apply Deployment command in X namespace. This generates a TLS request with user's credentials

1. __Authentication__ module determines if is grant-ward or an imposter
2. __Authorization__ module (RBAC) determines whether grant-ward is allwed to create a Deployment in X namespace
3. __Admission control__ checks and applies policies
4. The request is accepted and executed

## AuthN: Authentication

- Authentication is about proving your identity
- All the requests to the API server have to include credentials 
- If verification fails, API server returns an HTTP 401.

### Checking Authentication Setup

```txt
C:\Users\<user>\.kube\config
/home/<user>/.kube/config
```

Many Kubernetes installations can auto-merge cluster endpoint details and credentials into existing kubeconfig.

```yaml
# kubeconfig file  example
# defines cluster and user => combines into a context
# sets the default context for kubectl commands 

apiVersion: v1
kind: Config
clusters:
-   cluster:
    name: prod-shield
        server: https://<url-or-ip-address-of-api-server>:443
        certificate-authority-data: XXXX...
    users:
    -   name: njfury
        user: 
            as-user-extra: {}
                token: XXXX...
contexts:
-   context:
    name: shield-admin
        cluster: prod-shield
        user: njfury
        namespace: default
current-context: shield-admin
```

- __clusters__ section defines one or more K8s clusters. 
  - Friendly name
  - API server endpoint
  - public key of its CA 
  - The example is exposing HTTPS port
- __users__ section defines one or more users
  - Name
  - token (often X.509 cert) it has to be signed by the cluster's CA.
- __context__ section combines users  and clusters

###  AuthZ: Authorization (RBAC)

- It is posible run multiple authZ modules on a single cluster 
- RBAC: Role Based Access Control 
  - Users
  - Actions
  - Resources

_Which users can perform which actions against which resources_.

__Kubernetes only support allow rules__.

#### Users and Permissions

- Roles: define a set of permissions
- RoleBindings: grant those permissions to users

```yaml
# Resource Manifest 
# Defines a Role object

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: 
    namespace: shield
    name: read-deployments
rules:
-   apiGrops: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "watch", "list"]
```

Following RoleBinding grants the previous 'read-deployments' Role to a user called 'sky'

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
    name: read-deployments
    namespace: shield
subjects;
-   kind: user
    name: sky
    apiGroup: rbac.authorization.k8s.io
roleRef:
    kind: Role
    name: read-deployments
    apiGroup: rbac.authorization.k8s.io
```

- **apiGroups** and **resources** define the object
- **verbs** define the actions against Deployment objects 

__Kubernetes uses a standard set of verbs to describe the actions__.

| HTTP method | Kubernetes verbs | Common responses |
| -- | -- | -- | 
| POST | create | 201 created, 403 |
| GET | get, list, watch | 200 OK, 403 Access denied |
| PUT | update | 200, 403 |
| PATCH | patch | 200, 403 |
| DELETE | delete | 200, 403 |

A great help to build rule definitions: ```kubectl api-resources --sort-by name -o wide```

Use wildcard * to refer to all API groups, all resources and all verbs. Example: cluster admin

```yaml
# cluster admin 
rules:
-   apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

- RBAC objects:
  - Roles
  - ClusterRoles
  - RoleBindings
  - ClusterRoleBindings

__Role__ and __RoleBinding__ are namespaced objects: The can only be appllied to a single Namespace.

__ClusterRole__ and __ClusterRolebinsing__ are cluster-wide objects and apply to al Namespaces.

### Powerful Pattern

- define ClusterRoles (Roles at cluster level)
- bind via RoleBinding: bind them to specific Namespaces
- This lets define common roles once and re-use them across multiple Namespaces.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
    name: read-deployments
rules:
-   apiGroup: ["apps"]
    resources: ["deploymetns"]
    verbs: ["get", "watch", "list"]
```

most of clusters have a set of pre-created roles and bindings that grant permissions to an all-powerofull user.

Dokcer Desktop Kubernetes clusters run the API server in a Pod called kube-apiserver-docker-desktio in the kube-system Namespace. 

```sh
# shows Node and RBAC modules
kubectl describe pod kube-apiserver-docker-desktop \
--namespace kube-system | grep authorization --authorization-mode=Node,RBAC
```

Docker Desktoop also configures kubeconfig file with a user called docker-desktop.

```sh
kubectl config view
<Snip>
users:
- name: docker-desktop
  user:
    client-certificate-data: REDACTED
    client-key-dataL REDACTED
<Snip>
```

```sh
describe clusterrole cluster-admin
```

Docker Desktop configures the ```kubeconfig``` file with a client certificate signed by the cluster's certificate authority (CA). This means the cert is trusted by the cluster. It contains credentials for a user called "docker-for-desktop" that is a member of a group called "system:masters". A ```cluster-admin``` ClusterRole exists on the cluster that has admin rights to all objects in all Namespaces. This is bound to members of the system:masters group via ClusterRoleBinding that is also called ```cluster-admin```.

## Admission Control

- Admission control runs after successful authentication and authorization.
- All about policies
- Two types of admissions controllers:
  - Mutating
    - checks for compliance
    - can modify requests
    - Always run first 
  - Validating
    - checks for compliance 
    - cannot modify requests 

```
# Docker Desktop cluster
# Shows the API server 
kubectl describe pod/kube-apiserver-docker-desktop \
    --namespace kube-system | grep admission 
```

- There are a lots of admission controllers and are becoming more and more important in real-world production clusters.
If any admission controller rejects a requests, it is immediately rejected without checking other admission controllers. 
