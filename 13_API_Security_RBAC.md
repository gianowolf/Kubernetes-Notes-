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

## Authentication

- Authentication is about proving your identity
- All the requests to the API server have to include credentials 
- If verification fails, API server returns an HTTP 401.

### Checking Authentication Setup

```
C:\Users\<user>\.kube\config
/home/<user>/.kube/config
```

Many Kubernetes installations can auto-merge cluster endpoint details and credentials into existing kubeconfig.

```yaml
# kubeconfig file  example
# defines cluster and user => combines into a context
# sets the default context for kubectl commands 

