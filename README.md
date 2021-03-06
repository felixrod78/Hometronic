# Hometronic

Desarrollo de un cluster de k8s con Raspberry,Kubespray bajo Ubuntu 20.Server,Grafana y Mongo-DB.
Sobre esta arquitectura la idea es montar un sistema de domótica casera con Hasssio,Zigbee y mas cosas...todo por llegar


## **Requisitos**

Ansible 2.9
Ubuntu 20.server

**Primeros pasos**



#### Raspberry     --------------------------------------------------------
Configurar tarjeta red ethernet-wifi

```bash
nano /boot/firmware/cmdline.txt
cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
```

```bash
sudo apt-get update -y && sudo apt-get --with-new-pkgs upgrade -y
sudo apt install python3-pip git -y
```

#### Controller (desde donde vas a controlar el cluster)-------------------------------------------------

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
git checkout v2.13.0
pip3 install -r requirements.txt
cp -rfp inventory/sample inventory/mycluster
declare -a IPS=(192.168.0.&)
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```
Editamos el playbook roles/bootstrap-os/tasks/bootstrap-debian.yml y borramos lo siguiente
 ```bash

- name: Install python
  raw:
    apt-get update && \
    #DEBIAN_FRONTEND=noninteractive apt-get install -y python-minimal
  become: true
  environment: {}
  when:
    - need_bootstrap.rc != 0
```

Lanzamos la instalación de k8s: 
```bash
    ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml \
    -e "ansible_distribution_release=bionic kube_resolv_conf=/run/systemd/resolve/resolv.conf local_path_provisioner_enabled=true"
```

**Info importante: **
 bionic since there are no docker images for 20.04 version
  /run/systemd/resolve/resolv.conf since core-dns will be crashed by loop protection
 dashboard_enabled=false useless application from my point of view
 We want to have ability to provision pv right after deployment

**Copiar kubeconfig **
```bash
ssh 192.168.0.$
sudo cp /root/.kube/config /home/ubuntu/
sudo chown ubuntu /home/ubuntu/config
mkdir /root/.kube/
scp ubuntu@192.168.0.$:/home/ubuntu/config ~/.kube/config
```

**Instalacion cliente kubectl**
```bash
    curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    mv ./kubectl /usr/local/bin/kubectl
```


**Hasta que tengamos mas nodos en el cluster  TENER EN CUENTA**
 we have only one node,  we don't need to more one pod for core-dns
we will remove dns-autoscaler
```bash
kubectl delete deployment dns-autoscaler --namespace=kube-system
```
scale current count of replicas to 1
```bash
kubectl scale deployments.apps -n kube-system coredns --replicas=1
```

to be able to recieve incoming connection to K8S we need to install metallb
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
```
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
```

create configmap and apply it
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.0.21-192.168.0.25
```

**Solo en primera instalación**
```bash
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
**En Controller**
```bash
curl -LO https://get.helm.sh/helm-v3.2.1-linux-amd64.tar.gz
tar -zxvf helm-v3.2.1-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
# install ingress we must set image ARM64
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress ingress-nginx/ingress-nginx --set controller.image.repository="quay.io/kubernetes-ingress-controller/nginx-ingress-controller-arm64"
```

**Primer deployment**
```bash
kubectl create namespace deployment
kubectl create service nodeport nginx --tcp=80:80 -n deployment
kubectl create deployment nginx --image=nginx  -n deployment
# Usaremos traefik en este caso para instalar y configurar nuestro ingress
#https://doc.traefik.io/traefik/v1.7/
kubectl apply -f https://raw.githubusercontent.com/containous/traefik/v1.7/examples/k8s/traefik-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/containous/traefik/v1.7/examples/k8s/traefik-ds.yaml
kubectl create -f deployment_ing.yaml
