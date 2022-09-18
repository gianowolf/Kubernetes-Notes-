# 11 ConfigMaps and Secrets

## Example of Chaos: Coupled-World

A component deploys modern apps to K8s: Dev, Test and Prod envs. TEsting is performed in dev, further testing is done in test environment with stringent rules and policies applied. Finallym stable componentes graduate to the prod environment 

Each enviroment has subtle differences such number and config of nodes, network an security policies, credentials, certificates, etc. You have to perform all of the following for every business application

- **build** three disctinct images (dev, test, prod)
- **store** the images in three distinct repositories (dev, test, prod)
- **run** each version in a specific enviromnment  

every change of the app even smallers like fix a typo needs to build, test and store three disctinct images, every image has different repositories, different credentials and permissions, prod images may contain sensitive config data, sensitive pass, encription keys, etc.

## De-coupled world

- build a single app image shared across three environments 
- store the single image in a single repo
- run the single version of the image in all environments

### Culture

- Build the app images as generically as possible
  - No embedded configuration
- Create configurations in separate objects and apply to the app when you run

## ConfigMap Theory

- Kubernetes provides an __object called__ ConfigMap: 
  - Stores configuration data __outside__ a Pod.
  - It makes easy to inject the config into Pods at run-time
- first-class object in Kubernetes API under core API group
  - stable, operates under kubectl, deployed via YAML manifest
- Typically used to store non-sensitive configuration data as:
  - ENV vars
  - config files (webserver config, database config)
  - Hostnames
  - Service ports
  - account names
- __NOT USED__ to store __SENSITIVE__ data such: (Secret object)
  - certificates
  - passwords 

At _High-level_ ConfigMap is a place to store config data that can be injected into containers at runtime

_Behind the scenes_ ConfigMaps are a map of key-vale pairs called __entry__.

- __Key__: arbitrary name that can be created from alphanumerics, dashes, dots and underscores
- __Values:__ anything, include multiple lines with carriage returns
- ```key:value```
  - ```db-port:13306```
  - ```hostname:msb-prd-db1```
  - A more complex example:
```yaml
# Complex example
key: conf
value:

directive in;
main block;
http {
    server {
        listen      80 default_server;
        server_name *.nigelpoulton.com;
        root        /var/www/nigelpoulton.com;
        index       index.html
        location / {
            root    /usr/share/nginx/html;
            index   index.html;
        }
    }
}
```

- Once stored in a ConfigMap it can be injected into containers at run-time via:
    - Env vars
    - Files in a Volume (most flexible)
    - Arguments to the container's startup command (most limited)

Kubernetes-native applications (one that knows it's running on Kubernetes and can talk to the Kubernetes API) can access ConfigMap data via API without needing env vars or volumes.

## hands-on ConfigMaps

### Imperatively 

```sh
kubectl create configmap testmap1 \
--from-literal shortname=AOS \
--from-literal longname="Agents of Shield"
```


Observe the obejct is a map of key-value pairs dressed up as Kubernetes object.

```sh
# create from file 
kubectl create cm testmap2 --from-file cmfile.txt
```

```sh
kubectl get cm 
kubectl describe cm testmap1
kubectl describe cm testmap2
kubectl cm testmap1 -o yaml #describe entire object
```

in testnamp2 key is the name of the input file and value is the content.

### declaratively

don't have spec section. Insthead have a data section that defines the key-value pairs.

```yaml
kind: ConfigMap
apiVErsion: v1
metadata:
  name: multimap
data:
  given: Nigel
  famly: Poulton
```

```sh
kubectl apply -f multimap.yml
```

Follow YAML creates a ConfigMape with a single map entry. After the name of key uses a pipe to tells Kubernetes that everything following the pipe is to be rteated as a single literal value.

```yaml
kind: ConfigMap
apiVersion: v1
metadata: 
  name: test-config
data:
  test.conf |
  env = plex-test
  endpoint = 0.0.0.0:310001
  char = utf8
  vault = PLEX/test
  log-size = 512M
```

```sh
kubectl apply -f singlemap.yml
```

ConfigMaps are flexible and cab be used to inser complex configurations including JSON files or scripts into containers at run-time.

### Injecting ConfigMap data into Pods and containers

#### Environment Variables

```yaml
apiVersion: v1
kind: Pod
...
containers:
  - name: ctr1
    env:
      - name: FIRSTNAME
      valueFrom:
        configMapKeyRef:
          name: multimap
          key: given
      - name: LASTNAME
        valueFrom:
          configMapKeyRef:
            name: multimap
            key: famly
...
```

```sh
kubectl apply -f envpod.yml
kubectl exec envpod -- env | grep NAME
```

- ConfigMaps with environment variables are static: updates made to the map don't get reflected in running containers.
  
#### Startup Comands

```yaml
spec: 
  containers:
    - name: args1
      image:  busybox
        command: [  "/bin/sh",  "-c", "echo First Name $(FIRSTNAME) last name $(LASTNAME") ]
      env:
        - name: FIRSTNAME
          valueFrom:
            configMapKeyRef:
              name: multimap
              key:  given
        - name: LASTNAME...
        ...
```

```sh
kubectl apply -f startuppod.yml
```

Same limitations as environment variables

#### ConfigMaps and Volumes

- Most flexible option:
  - reference entire config files
  - make updtes to the ConfgMap: make changes to entries after deployed the Pod

1. Create a *ConfigMap*
2. Create a *CongMap volume* in the Pod template
3. Mount the *ConfigMap volume* into the container
4. Entries in the *ConfigMap* will appear in the container as individual files

- ```spec.volumes``` creates a volume valled **volmap** based on the **multimap** ConfigMap
- ```spec.containers.volumeMounts``` mounts the volmap volume to ```/etc/name```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cmvol
spec:
  volumes:
    - name: volmap
      configMap:
        name: multimap
  containers:
    - name: ctr
      image: nginx
  volumeMounts:
    - name: volmap
        mountPath: /etc/name
```

```sh
kubectl apply -f cmpod.yml
kubectl exec cmvol -- ls /etc/name
```

```sh
# Change an re-run
kubectl edit cm multimap
kubectl exec cmvol --cat /etc/name/...
```

<div class="pagebreak"> </div>

## Secrets

- almost identical run-time injection as ConfigMaps.
- Designed for sensitive data such pass, certs and tokens
  
### Security

- Kubernetes does not encrypt Secrets in the cluster store
- It is possible to configure encryption-at-rest with EncryptionConfiguration objects
- It is possible use 3rd-party tools such HashiCorp's Vault

1. Secret is rceated and persisted to the cluster store as an unenctryped object
2. Pod uses it 
3. Secret is transferred over the network, un-encrypted to the node
4. kubelet on the node starts Pod and its containers
5. Secret is mounted into the container via in-memory tmpfs and decoded from base64 to plain text
6. App consumes it

__Secrets are not very secure, by can take extra steps to make them secure.__

### Creating Secrets

```sh
kubectl create secret generic creds --from-literal \
user=nigelpoulton \
--from-literal pwd=Password1234
```

```sh
kubectl get secret creds -o yaml
kind: Secret
data: #base64 encoded
  pwd: UGFksdxTjrrfX4
  user: bbi23D3dfjfdkdj

# decoding
echo UGFzc3svcmQxMjM= | base64 -d
Password123 #base 64 is not secure 
```

```yaml
apiVErsion: v1
kind: Secret
metadata:
  name: tkb-secret
  labels:
    chapter: configmaps
type: Opaque
data:
  username: ...
  password: ...
```

```sh
kubectl apply -f tkb-secret.yml
```

### Using Secrets in Pods

- Most flexible way: _Secret Volume_

```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: secret-pod
  labels:
    topic: secrets
spec:
  volumes:
  - name: secret-vol
    secret:
      secretName: tkb-secret
  containers:
  - name: secret-ctr
    image: nginx
    volumeMounts:
    - name: secret-vol
      mountPath: "/etc/tkb"
```

- Secret vols are auto monuted as Read-Only to prevent muntating

```yaml
kubectl apply -f secretpod.yml
kubectl exec secret-pod -- ls /etc/tkb
password
username
kubectl exec secret-pod --cat /etc/tkb/password
```

there are more ways to use ConfigMaps and secrets.

