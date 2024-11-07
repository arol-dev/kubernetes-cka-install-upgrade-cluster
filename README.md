# Estrategias de Backup y Recuperación de etcd - Lab Práctico para CKA

Este laboratorio práctico está diseñado para ayudar a los aspirantes a la certificación CKA (Certified Kubernetes Administrator) a aprender a implementar estrategias de backup y restauración de etcd, un componente esencial en el control plane de Kubernetes. Se basa en el material de respaldo "Backing Up and Restoring etcd". Asegúrate de seguir cada paso cuidadosamente y practicar varias veces para estar completamente preparado para el examen.

## Requisitos

- Acceso a un nodo de control (controlplane) con permisos para ejecutar comandos de Kubernetes y etcd.
- Cliente `etcdctl` instalado.
- Conocimiento básico del uso de `kubectl` y edición de archivos YAML.

## Objetivos

1. Realizar una copia de seguridad (snapshot) de etcd.
2. Restaurar la copia de seguridad en un nuevo directorio de datos.
3. Modificar el manifiesto de etcd para usar el nuevo directorio.

## Prerrequisitos

- **VirtualBox** (se necesita **Python** y **pywin32** como prerrequisitos).
- **Vagrant**.
- **MobaXterm** para sesiones SSH.

## Contenido del Repositorio

Este repositorio incluye:

- Una carpeta `scripts` con dos scripts que proporcionan soporte durante el laboratorio.
- Un fichero `Vagrantfile` que permite automatizar el despliegue de tres VMs en VirtualBox.

Las VMs consisten en:

- 1 nodo master.
- 2 nodos worker.

## Paso 1: Despliegue de las VMs

1. Clona el repositorio en tu entorno local:

   ```bash
   git clone https://github.com/arol-dev/kubernetes-cka-install-upgrade-cluster.git
   cd kubernetes-cka-install-upgrade-cluster
   ```

2. Dentro del repositorio, ejecuta el siguiente comando para desplegar las VMs:

   ```bash
   vagrant up
   ```

   Esto comenzará a desplegar tres VMs en VirtualBox: un nodo master y dos worker nodes. Espera unos minutos para que el proceso termine.

3. Verifica el estado de las VMs con:

   ```bash
   vagrant status
   ```

   Asegúrate de que las tres máquinas estén en estado `running`.

4. Obtén la configuración SSH para conectarte a las máquinas:

   ```bash
   vagrant ssh-config
   ```

   Guarda los detalles proporcionados, ya que los necesitarás en el siguiente paso.

## Paso 2: Conectar a las VMs con MobaXterm

1. Abre **MobaXterm** y utiliza la configuración SSH obtenida anteriormente para conectarte a las tres máquinas.
   - No se requiere un usuario específico, deja el campo vacío.
   - Si se te solicita usuario o contraseña, utiliza la cadena `vagrant`.

### Paso 3: Instalación del Cliente etcdctl

Para realizar operaciones de backup y restauración, necesitas el cliente `etcdctl`. Instálalo con:

```bash
sudo apt install etcd-client
```

### Paso 4: Realizar una Copia de Seguridad de etcd

1. **Identificar el pod de etcd**: Ejecuta el siguiente comando para identificar el pod etcd en el cluster.

   ```bash
   kubectl get pods -n kube-system -o wide | grep etcd
   ```

2. **Obtener las ubicaciones de los certificados y la clave**: Necesitarás los certificados para realizar una copia de seguridad. Usa:

   ```bash
   kubectl describe pod etcd-master -n kube-system
   ```

   Busca las siguientes opciones: `cert-file`, `key-file`, `trusted-ca-file`, y `listen-client-urls`.

3. **Realizar la copia de seguridad**:

   Ejecuta el siguiente comando para realizar la copia de seguridad en un archivo snapshot.

   ```bash
   sudo ETCDCTL_API=3 etcdctl snapshot save /tmp/snapshot-pre-boot.db      --endpoints=https://127.0.0.1:2379      --cacert=/etc/kubernetes/pki/etcd/ca.crt      --cert=/etc/kubernetes/pki/etcd/server.crt      --key=/etc/kubernetes/pki/etcd/server.key
   ```

   Este comando crea un snapshot de etcd en la ruta `/tmp/snapshot-pre-boot.db`.

### Paso 5: Restaurar la Copia de Seguridad

1. **Eliminar el deployment anterior** (opcional):

   ```bash
   kubectl delete -f https://k8s.io/examples/controllers/nginx-deployment.yaml
   ```

2. **Restaurar la copia de seguridad**: Para restaurar la copia de seguridad, utiliza el siguiente comando:

   ```bash
   sudo ETCDCTL_API=3 etcdctl snapshot restore /tmp/snapshot-pre-boot.db      --name=master      --data-dir=/var/lib/etcd-from-backup      --initial-cluster=master=https://127.0.0.1:2380      --initial-cluster-token=etcd-cluster-1      --initial-advertise-peer-urls=https://127.0.0.1:2380      --cacert=/etc/kubernetes/pki/etcd/ca.crt      --cert=/etc/kubernetes/pki/etcd/server.crt      --key=/etc/kubernetes/pki/etcd/server.key
   ```

   Este comando restaurará el snapshot en el directorio `/var/lib/etcd-from-backup`. Ten en cuenta que estamos dando un nuevo nombre al clúster de etcd y reinicializando el token del clúster.

### Paso 6: Modificar el Manifiesto de etcd

Para aplicar la restauración en el clúster, debes modificar el manifiesto del pod etcd para que apunte al nuevo directorio de datos.

1. **Editar el manifiesto**:

   ```bash
   sudo vi /etc/kubernetes/manifests/etcd.yaml
   ```

2. **Cambiar el directorio de datos**: Modifica la línea `--data-dir` para que apunte al nuevo directorio restaurado:

   ```yaml
   - --data-dir=/var/lib/etcd-from-backup
   ```

3. **Guardar los cambios**: Cuando se guarda este archivo, el pod de etcd se destruirá y se volverá a crear automáticamente, utilizando los nuevos datos restaurados.

### Paso 7: Verificación

1. **Verificar el estado del pod**:

   ```bash
   kubectl get pods -n kube-system -o wide | grep etcd
   ```

   Asegúrate de que el pod esté en estado `Running`.

2. **Verificar los registros**:

   ```bash
   kubectl logs -n kube-system etcd-master
   ```

   Revisa los registros para asegurarte de que no haya errores relacionados con la restauración.

## Consejos Finales

- Practica el proceso varias veces para que puedas realizarlo rápidamente en el examen.
- Familiarízate con la estructura del archivo YAML de etcd.
- Recuerda siempre modificar el `--data-dir` en el manifiesto para aplicar la restauración.

¡Espero que este laboratorio te sea útil en tu preparación para el examen CKA! Si tienes alguna pregunta o deseas practicar más escenarios, no dudes en mencionarlo.