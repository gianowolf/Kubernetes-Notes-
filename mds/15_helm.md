## Section 1: Introduction to Helm

- helm.sh/docs
- bitnami.com/stacks/helm

### Problemas de utilizar Kubernetes sin Helm

- Exceso de archivos YAML 
- archivos estaticos
- Helm simplifca el proceso de deploy
- En K8s cada vez que hacemos cambios, se requiere un re-deployment de todos los objetos del cluster

### Ventajas de Utilizar Helm

Si queremos crear una app MongoDB debemos configurar todos los manifest files de los objetos necesarios, tales como Deployment, Services, ConfigMap, y configurar containers, volumenes ,etc.

- Los expertos de cada aplicacion/herramienta se encargan de configurar estos paquetes. 
- Helm utiliza todos los manifest files y crea un template llamado Values.yaml.
- Las upgrades se realizan directamente en base a estos archivos 




