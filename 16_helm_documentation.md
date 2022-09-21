# Helm Documentation

Helm is the package manager for Kubernetes

- website: https://helm.sh/

## Quickstart Guide 

### Prerequisites

1. Kubernete cluster
   1. Kubernetes installed
   2. configured copy of kubectl
2. Deciding security configs to apply
3. installing and configuring Helm

### Install Helm

#### From Binary Releases

1. Download the desired version from https://github.com/helm/helm/releases
2. Unpack it ```tar -zxvf helm-v3.0.0-linux-amd64.tar.gz```
3. find ```helm``` binary, move it to destination ```mv linux-amd64/helm /usr/local/bin/helm
4. You should be able to run the client ```helm help```

#### From Script 

```sh
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

#### From Apt

```sh
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

#### From Snap

```sh
sudo snap install helm --classic
```

### Initialize a HElm Chart Repo

```sh
# See the list of chart repositories
helm repo add bitnami https://chart.bitnami.com/bitami

helm search repo bitami

# Install an Example Chart
helm repo update
helm show chart bitnami/mysql
helm install bitnami/mysql --generate-name 

helm uninstall my_chart_name
```

## Using Helm

- __Chart__ is a Helm package. It contains all the resource definitions necessary to run an application, tool or service inside of a Kubernetes cluster.
- __Release__ is an instance of a chart running in a Kubernetes cluster. One chart can be installed many times into the same cluster.

Helm instals charts into Kubernetes, creating a new release for each installation. To find new charts, you can earch Helm chart repositories.

```
helm search hub wordpress
helm search repo brigade
```

### Cmmon Commands

```sh
helm install release_name bitnami/wordpress
```

Helm client will print useful information about resources created. 

```sh
helm status release_name
shows the current state of the release
```

### Customizing a Chart Before Installing

```sh
helm show values bitami/wordpress

# override any of these settings in a YAML formatted file 
echo '{mariadb.auth.database: user0db, mariadb.auth.username: user0}' > values.yaml
helm install -f values.yaml bitnami/wordpress --generate-name
```

- Ways to pass data during install
  - ```--values or -f``` Specify a YAML file 
  - ```--set``` Specify overrides on the command line 
    - Values specified with --set are persisted in a ConfigMap. Can be viewed with ```helm get values <releae-name>```.

```yaml
# --set format YAML equivalents

#--set name=value
name: value

# --set a=b,c=d 
a: b
c: d

# --set outer.inner=value
outer:
    inner: value

# --set name={a, b, c}
name:
  - a
  - b
  - c

# --set servers[0].port=80,servers[0].host=example
servers:
  - port: 80
    host: example
```

### Upgrade and Rollback

- When a new version of a chart is released
- When want to change the configuration of the release

Helm tries to perform the least invasive upgrade. It will __only update things that have changed since last release__

```sh
helm upgrade -f panda.yaml happy-panda bitnami/wordpress
```

```sh
helm rollback happy-panda 1
```

Every time an install, upgrade or rollback happens, the revision number is incremented by 1. he ifrst revision number is always 1.

```sh
helm history <release>
```

### Options

- ```--timeout``` default 5m
- ```--wait``` waits until all Pods are in ready state, PVCs are bound, Deployments have minimum desired Pods in ready state and Services have an IP address before marking the release as successful.
- ```--no-hooks``` skips running hooks for the  command

### Uninstall 

```sh
helm uninstall happy-panda
```

```sh
helm list 
helm list --all
helm list --uninstalled
```
