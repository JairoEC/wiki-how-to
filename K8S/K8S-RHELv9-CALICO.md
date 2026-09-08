
**REQUERIMIENTOS DEL SISTEMA**
- SO    : RHEL 9.8 LTS
- CPU   : 2vCPU
- RAM   : 4GB
- SSD   : 25GB
- K8S   : v1.35
- CRI-O : v1.35
- CALICO: ---
otras aplicaciones:
- conntrack

0. 
```
dnf update && dnf upgrade
```
# INSTALACIÓN BÁSICA DE KUBENETES
1. Modificar el hostname de cada nodo

```
hostnamectl set-hostname [HOSTNAME]
```
2. Agregar archivo de configuracion

```
cat <<EOF >> /etc/hosts
192.168.19.226 [HOSTNAME-MASTER] master
192.168.19.227 [HOSTNAME-WORKER-1] worker1
192.168.19.228 [HOSTNAME-WORKER-2] worker2
EOF
```

. Desactivamos la memoria swap

```
swapoff -a
```

Tambien editar el archivo fstab
 ```
 vim /etc/fstab
 ```

Comentar la línea swap en el fstab para que este no vuelva  a funcionar desde de cada reinicio

```
#/dev/mapper/rhel-swap   none   swap   defaults   0 0
```

## INSTALACIÓN DE CONTAINER RUNTIME

Se pueden usar como container runtime Docker, Cotainerd y CRI-O.

### HABILITAR PUERTOS DE CONEXION Y KERNEL

1. Habilitamos los módulos de kernel.

```
sudo modprobe overlay
sudo modprobe br_netfilter
```

> - ***modprobe***: Comando para cargar módulos del kernel en un sistema Linux.
> - ***overlay***: Módulo que permite que los contenedores utilicen la tecnología de capas para compartir eficazmente imágenes de contenedores y archivos base, lo que reduce el uso de espacio en disco.
> - ***br_netfilter***: Módulo para configurar la funcionalidad de Network Address Translation (NAT) y el filtrado de paquetes en la red de contenedores del clúster.

2. Creamos un archivo de configuración sysctl para habilitar el reenvío de IP y la configuración de netfilter de forma persistente.

```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
*Comprobar la configuración*
```
sysctl --system
```

> - ***net.bridge.bridge-nf-call-ip6tables***: Controla si los paquetes IPv6 que pasan por puente de red deben ser pasados a través de la tabla ip6tables para procesamiento.
> - ***net.bridge.bridge-nf-call-iptables***: Controla si los paquetes que pasan por un puente de red deben ser pasados a través de la tabla iptables para procesamiento.
> - ***net.ipv4.ip_forward***: Habilita o deshabilita el reenvío de paquetes IP en el kernel de Linux.

3. Verificación de ....

```
lsmod | grep overlay
lsmod | grep br_netfilter
```

#### INSTALACIÓN DE CRI-O

1. 

> En RHEL no existe un equivalente a apt-transport-https, ya que dnf y yum soportan conexiones https de forma nativa.

2. Aplicar el modo permisivo
```
sudo setenforce 0
```
> Los cambios habilita de forma inmediata el modo permisivo pero para hacerlo persistente se debe de cambiar el archivo de configuración:
```
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
> Una opción es editar el archivo de forma manual, se encuentra en la ruta /etc/selinux/config

```
vim /etc/selinux/config
```
Cambiar el valor de SELINUX

```
SELINUX=permissive
```

Agregando la version de Kubernetes
```
CRIO_VERSION=v1.35

cat <<EOF | sudo tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/rpm/repodata/repomd.xml.key
EOF
```

3. Instalar CRI-O y sus dependencias
```
sudo dnf install -y container-selinux
sudo dnf install -y cri-o
```

4. Habilitar y arrancar servicio

```
sudo systemctl daemon-reload
sudo systemctl enable --now crio
sudo systemctl start crio
```
5. Verificar que CRI-O está activado
```
systemctl status crio
ls -l /var/run/crio/crio.sock
```

#### INSTALAR CONTRACK
Elegir una forma
**(OPCIONAL - CON CONTRACK)**
6. Paquete que le ayuda a kubernetes buscar el paquete de crio

```
dnf install -y conntrack
```

**(OPCIONAL - SIN CONTRACK)**
6. Confiugrar y modificar el archivo por defecto crictl:

```
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///var/run/crio/crio.sock
image-endpoint: unix:///var/run/crio/crio.sock
timeout: 10
debug: false
EOF
```
7. Asegurar que CRI-O está encendido antes del init
```
sudo systemctl restart crio
```

## INSTALACIÓN DE KUBERNETES
**En todos los nodos**

1. Agregar el repositorio de kubernetes
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
2. Instalar kubelet, kubeadm y kubectl
```
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

2.1. Instalar plugin para congerlar version

```
dnf install 'dnf-command(versionlock)'
```
Tambien puedes usar:

    dnf install python3-dnf-plugins-core

3. Fijar/congelar la version
```
dnf versionlock add kubelet kubeadm kubectl
```

***(OPCIONAL) Si no quieres usar un plugin*** 

a. Abrir el archivo en el path

```
vim /etc/dnf/dnf.conf
```

b. Al final del bloque main agregar una lina:

```
excludepkgs=kubelet kubeadm kubectl
```

4. Habilitar el servicio
```
systemctl enable --now kubelet
```

## APERTURA DE PUERTOS

***Configuracion en el master***
```
# Puertos esenciales del Control Plane
sudo firewall-cmd --permanent --add-port=6443/tcp      # Kubernetes API Server
sudo firewall-cmd --permanent --add-port=2379-2380/tcp # etcd server client API & peer
sudo firewall-cmd --permanent --add-port=10250/tcp    # Kubelet API
sudo firewall-cmd --permanent --add-port=10257/tcp    # kube-controller-manager
sudo firewall-cmd --permanent --add-port=10259/tcp    # kube-scheduler


# Recargar el firewall para aplicar cambios
sudo firewall-cmd --reload
```

***Configuracion en los workers***

```
# Puertos esenciales de los Workers
sudo firewall-cmd --permanent --add-port=10250/tcp       # Kubelet API
sudo firewall-cmd --permanent --add-port=30000-32767/tcp # Rango para servicios NodePort

# Recargar el firewall para aplicar cambios
sudo firewall-cmd --reload
```

***A. EN CASO DE USAR IPIP***
```
sudo firewall-cmd --permanent --add-port=179/tcp      # Puerto que usa Calico
sudo firewall-cmd --permanent --add-rich-rule='rule protocol value="4" accept'
```


***B. EN CASO DE USAR VXLAN***
```
sudo firewall-cmd --permanent --add-port=4789/udp
sudo firewall-cmd --permanent --add-masquerade

```

***Verificar que puerto está usando CALICO***

```
kubectl get ippools -o custom-columns=NAME:.metadata.name,IPIP:.spec.ipipMode,VXLAN:.spec.vxlanMode

kubectl get ippools -o yaml | grep cidr
```
salida:

```
cidr: 172.16.0.0/16
```
**Es necesario señalar el rango que indica el CIDR como tráfico de confianza**
```
firewall-cmd --permanent --zone=trusted --add-source=172.16.0.0/16
firewall-cmd --reload
```


## CONFIGURAR KUBERNETES

**Solo en nodo master**

1. En el nodo master ejecutar el comando kubeadm init que preparará todo para el cluster.
```
kubeadm init
```

> Al final de la configuracion proporcionará el comando para unir o registrar los nodos workers (kubeadm join)

2. Agregar las variables de entorno
```
export KUBECONFIG=/etc/kubernetes/admin.conf
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

3. Agregamos el plugin de red para la comunicacion entre pods. Aqui se usara Calico

```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.32.2/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

4. Calico necesita permitir otros tipo de comunicacion que usa calico como medidas de seguridad:

```
firewall-cmd --permanent --zone=trusted --add-interface=cali+
```

> Kubernetes crea una interfaz veth que es usada como una clave virtual que conecta el Pod con SO de la VM. Las interfaces siempre se llaman cali segudio de un id (ej. `cali123abc` )

```
firewall-cmd --permanent --zone=trusted --add-interface=tunl0
```

> Calico usa `tun10` para encapsular paquetes cuando un Pod en un nodo necesita comunicarse con un Pod de otro nodo

```
firewall-cmd --permanent --zone=trusted --add-source=172.17.0.0/16
```

> Bloque de ips que Kubernetes reserva para asignar Pods.

```
firewall-cmd --permanent --zone=trusted --add-source=10.96.0.0/12
```

> Envoy proxy se comunica atraves de su propio DNS o IP de sevicio.

```
firewall-cmd --reload
```
Solo reiniciar el firewall

## CONFIGURACIÓN DE NODOS DE KUBERNETES

**Solo en los nodos workers**

1. En cada worker hacer un join usando el token y certificados obtenidos en el nodo master al realizar el `kubeadm init`

2. Ingresar el token y ssh generados. *ingresar el join obtenido*
```
# CODIGO DE EJEMPLO
kubeadm join 192.168.19.226:6443 --token 3tgjos.0lqayhrp3b7wcmlf \
--discovery-token-ca-cert-hash sha256:1c83c56ec2eefd538966464513798a0595bd73b3be637352b46fbcb231f55467
```

## VALIDACIÓN DEL CLUSTER DE KUBERNETES

1. En el nodo master validar el estado de los nodos

```
kubectl get nodes
```

