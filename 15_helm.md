# 15: Helm for Cubernetes

## Section 1: Introduction

- helm.sh/docs
- bitnami.com/stacks/helm

### Problemas de utilizar Kubernetes sin Helm

- Exceso de archivos YAML 
- archivos estaticos
- Helm simplifca el proceso de deploy
- En K8s cada vez que hacemos cambios, se requiere un re-deployment de todos los objetos del cluster

### Que es Helm

- Helm is un package manager as apt, npm or yum.
- Helm es un package manager para el cluster

```sh
helm install apache bitnami/apache --namesoace=web
helm upgrade apache bitnami/apache --namespace=web
helm rollback apache 1 --namespace=web
helm uninstall apache
```


### Ventajas de Utilizar Helm

Si queremos crear una app MongoDB debemos configurar todos los manifest files de los objetos necesarios, tales como Deployment, Services, ConfigMap, y configurar containers, volumenes ,etc.

Los expertos de cada aplicacion/herramienta se encargan de configurar estos paquetes.

#### Dynamic Config

Helm utiliza todos los manifest files y crea un template llamado Values.yaml.

#### Consistency

Las upgrades se realizan directamente en base a estos archivos 

#### Hooks 

#### Security

### Charts 

