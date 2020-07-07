# Instalando Kubernetes no CentOS 8

## Pre-requisitos

- Ter acesso ao usuário root dos servidores

## 1. Ajuste o arquivo `hosts` de cada um dos servidores para enxergar todos nós do cluster

``` sh
cat <<EOF>> /etc/hosts
192.168.0.47 master-node
192.168.0.48 node-1 worker-node-1
192.168.0.49 node-2 worker-node-2
EOF
```

## 2. Desabilitar o SELinux

Desabilita na sessão

```   sh
setenforce 0
```
Desabilita definitivamente

``` sh
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

## 3. Definir o Bridge para os pacotes nas Regas do IPTables

``` sh
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
Carregue as novas regras
``` sh
sudo sysctl --system
```

## 4. Disabilite o Firewall do CentOS

``` bash
systemctl stop firewalld
systemctl disable firewalld
```

## 5. Desabilitar o Swap
Desabilite o Swap da sessão
```sh
swapoff -a
```
Desabilite comentando a partição do swap no arquivo `fstab`
```
vi /etc/fstab
```

## 6. Reinicie o servidor

``` sh
reboot
```

## 7. Instale o Docker

Instale a dependencia do ContainerD que não existe no CentOS 8

```sh
dnf install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm
```
Instale o Docker com o o script shell disponivel no site do Docker

``` sh
curl -fsSL https://get.docker.com | bash
```
Ajuste o CGroup do Docker para SystemD, pré-requisitos do Kubernetes nas ultimas versões.
Liberar o repositório `Nexus` pois o mesmo não esta configurado `HTTPS`

```sh 
mkdir /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "insecure-registries" : ["tdbsbcsvr612.tdb.com.br:5000"],
  "dns": ["172.18.1.30","172.18.16.25"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
```
``` sh
mkdir -p /etc/systemd/system/docker.service.d
```
Inicializar o Docker após os ajustes
``` sh
systemctl daemon-reload
systemctl enable docker
systemctl start docker
```
Verifique se o Docker esta rodando
```sh
systemctl status docker
```
## 8. Instalando componenetes do Kubernetes

Adicionando o repositorio do Kubernetes
```sh
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
Instalar o Kubeadm Kubelet e o Kubectl
```sh
dnf install kubeadm kubelet kubectl -y 
```
Iniciando o Kubelet
```sh
systemctl enable kubelet
systemctl start kubelet
```

## 9. Iniciado o Primeiro Master Node
```sh
kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs
```
Copie e guarde os comandos de `JOIN` dos demais `masters` e dos `workers`

Instale o Pod Network Controller do cluster, nesse caso vamos utilizar o Weave que foi recomendado na documentação oficial do Kubernetes
```sh
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
Verifique se os pods estão sendo criados e estão em `running`, e se o Nó esta `Ready`
```sh
kubectl get pods --all-namespaces
kubectl get nodes
```
### 10. Iniciando os Demais Master Nodes
execute o comando exibido na saida de inicialização do primeiro master node
```sh
kubeadm join LOAD_BALANCER_DNS:6443 ....
```

## 11. Iniciando os Workers Nodes
execute o comando exibido na saida de inicialização do primeiro master node
```sh
kubeadm join  ....
```

## 12. Instalar Helm no Cluster
Utilizado a documentação oficial do [Helm](https://helm.sh/docs/intro/quickstart/)

Baixe o Helm
```sh
wget https://get.helm.sh/helm-v2.16.9-linux-amd64.tar.gz
tar -xvzf helm-v2.16.9-linux-amd64.tar.gz
```
Copie o executável do Helm para a pasta `/usr/local/bin`
```sh
cp helm-v2.16.9-linux-amd64/helm /usr/local/bin
```
Inicialize o Repositório de Chart do Helm
```
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```
Vamos verificar se esta funcionando, listando os charts disponiveis
```sh
helm repo update
helm search repo stable
```
## 13. Instalar Prometheus através do Helm

## 14. Instalar Grafana através do Helm

## 15 Instalar Ingress através do Helm



