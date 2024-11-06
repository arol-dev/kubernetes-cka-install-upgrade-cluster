
# Kubernetes Cluster Setup Lab - CKA Course

## Descripción del Laboratorio

El objetivo de este laboratorio es configurar un cluster Kubernetes nativo utilizando la herramienta `kubeadm`, conectarse al cluster desde nuestro entorno de desarrollo (IDE), desplegar la aplicación **BookInfo** de Istio y actualizar la versión del cluster de la 1.30 a la 1.31. Para ello, utilizaremos **Vagrant** para automatizar el despliegue de las máquinas virtuales (VMs) en **VirtualBox**.

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

## Paso 3: Configurar el Nodo Master

1. En el nodo master, ejecuta los siguientes comandos para configurar Kubernetes:
   ```bash
   sudo su
   # Deshabilitar swap
   sudo swapoff -a

   # Mantener el swap desactivado durante el reinicio
   (crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
   sudo apt-get update -y

   # Crear el archivo .conf para cargar los módulos al inicio
   cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   overlay
   br_netfilter
   EOF

   sudo modprobe overlay
   sudo modprobe br_netfilter

   # Configuración de sysctl para Kubernetes
   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-iptables  = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   net.ipv4.ip_forward                 = 1
   EOF

   # Aplicar los parámetros sysctl sin reiniciar
   sudo sysctl --system
   ```
2. Continúa con la instalación del runtime CRI-O y herramientas Kubernetes:
   ```bash
   sudo apt-get update -y
   sudo apt-get install -y apt-transport-https ca-certificates curl gpg
   sudo apt-get install -y software-properties-common

   # Declaración de versiones
   KUBERNETES_VERSION="v1.30"
   CRIO_VERSION="v1.30"
   KUBERNETES_INSTALL_VERSION="1.30.0-1.1"

   # Instalar CRI-O
   curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key | \
       gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

   echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
       tee /etc/apt/sources.list.d/cri-o.list

   sudo apt-get update -y
   sudo apt-get install -y cri-o

   sudo systemctl daemon-reload
   sudo systemctl enable crio --now
   sudo systemctl start crio.service
   echo "CRI runtime instalado con éxito"
   ```

3. Instalar las componentes de Kubernetes `kubelet`, `kubectl` y `kubeadm`:
   ```bash
   curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key | \
       gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

   echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" |
       tee /etc/apt/sources.list.d/kubernetes.list

   sudo apt-get update -y
   sudo apt-get install -y kubelet="$KUBERNETES_INSTALL_VERSION" kubectl="$KUBERNETES_INSTALL_VERSION" kubeadm="$KUBERNETES_INSTALL_VERSION"

   # Prevenir actualizaciones automáticas
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

## Paso 4: Configurar los Nodos Worker

1. En los nodos worker, utiliza el script `common.sh` presente en la carpeta `scripts` del repositorio. Copia este archivo en cada nodo worker.

2. Ejecuta los siguientes comandos en cada nodo worker:
   ```bash
   sudo su
   chmod +x common.sh
   ./common.sh
   ```

## Paso 5: Inicializar el Cluster en el Nodo Master

1. Ejecuta los siguientes comandos en el nodo master para inicializar el cluster:
   ```bash
   NODENAME=$(hostname -s)
   POD_CIDR="192.168.0.0/16"

   sudo kubeadm config images pull

   MASTER_PRIVATE_IP=$(ip addr show eth1 | awk '/inet / {print $2}' | cut -d/ -f1)
   sudo kubeadm init --apiserver-advertise-address="$MASTER_PRIVATE_IP" --apiserver-cert-extra-sans="$MASTER_PRIVATE_IP" --pod-network-cidr="$POD_CIDR" --node-name "$NODENAME" --ignore-preflight-errors Swap

   # Configurar kubeconfig
   mkdir -p "$HOME"/.kube
   sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
   sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config
   ```

2. También puedes exportar el kubeconfig para tener permisos de root:
   ```bash
   export KUBECONFIG=/etc/kubernetes/admin.conf
   ```

3. Instalar el plugin de red Calico:
   ```bash
   kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
   ```
   Verifica el estado de los pods:
   ```bash
   kubectl get pods -A
   ```

## Paso 6: Unir los Nodos Worker al Cluster

1. Copia el comando `kubeadm join` que recibiste después de inicializar el cluster y ejecútalo en cada nodo worker para unirlos al cluster.

## Paso 7: Configurar el IDE para Trabajar con el Cluster

1. Asegúrate de tener `kubectl` instalado en tu máquina local. Como IDE, utilizaremos Visual Studio Code.
2. Crea un nuevo archivo `KUBECONFIG` copiando el contenido del archivo `admin.conf` del nodo master en la ruta `/etc/kubernetes/admin.conf`.
3. Define la variable `KUBECONFIG` apuntando al archivo creado.
4. Verifica los contextos disponibles:
   ```bash
   kubectl config get-contexts
   ```
5. Cambia al contexto del nuevo cluster:
   ```bash
   kubectl config use-context kubernetes-admin@kubernetes
   ```
6. Verifica los nodos:
   ```bash
   kubectl get nodes
   ```

## Paso 8: Desplegar la Aplicación BookInfo de Istio

1. Crea el namespace para BookInfo:
   ```bash
   kubectl create ns bookinfo
   ```
2. Despliega la aplicación:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
   ```
3. Expone la aplicación localmente:
   ```bash
   kubectl port-forward svc/productpage 9080:9080 -n bookinfo
   ```
4. Prueba la aplicación desde el navegador:
   - [http://127.0.0.1:9080/productpage](http://127.0.0.1:9080/productpage)

5. Si la aplicación se ve lenta o incorrecta, reinicia el `coredns`:
   ```bash
   kubectl -n kube-system rollout restart deployment coredns
   ```

## Conclusión

Ahora has creado un cluster Kubernetes con un nodo de control y dos worker nodes, y has desplegado la aplicación BookInfo. Estás listo para comenzar a desplegar otras aplicaciones o experimentar con Kubernetes para obtener más experiencia.
