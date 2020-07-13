# Instalando Kubernetes no Ubuntu/Debian

## Pre-requisitos

- Ter acesso ao usuário root dos servidores

## 1. Ajuste o arquivo `hosts` de cada um dos servidores para enxergar todos nós do cluster

``` sh
cat <<EOF>> /etc/hosts
<<IP> <<MASTER HOSTNAME>>
<<IP> <<WORKER1 HOSTNAME>>
<<IP> <<WORKER2 HOSTNAME>>
EOF
```

## 2. Disabilite o Firewall do Servidor

``` bash
systemctl stop ufw
systemctl disable ufw
```

## 3. Desabilitar o Swap
Desabilite o Swap da sessão
```sh
swapoff -a
```
Desabilite comentando a partição do swap no arquivo `fstab`
```
vi /etc/fstab
```

## 4. Reinicie o servidor

``` sh
reboot
```

## 5. Instale o Docker

Instale o Docker com o o script shell disponivel no site do Docker

``` sh
curl -fsSL https://get.docker.com | bash
```
Ajuste o `CGroup` do Docker para SystemD, pré-requisitos do Kubernetes nas ultimas versões.

```sh 
mkdir /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
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
## 6. Instalando componenetes do Kubernetes

Adicionando o repositorio do Kubernetes
```sh
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
Instalar o Kubeadm Kubelet e o Kubectl
```sh
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
Iniciando o Kubelet
```sh
systemctl enable kubelet
systemctl start kubelet
```

## 7. Iniciado o Primeiro Master Node
```sh
kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs
```
Copie e guarde os comandos de `JOIN` dos demais `masters` e dos `workers` para utilizarmos a seguir.

Agora vamos configurar o `kubectl` para ele ter acesso ao Cluster. Na saida do `kubeadm` tem o seguintes comandos que devemos executar em cada `master` que realizarmos o `join` no cluster.

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Instale o Pod Network Controller do cluster, nesse caso vamos utilizar o Weave que foi recomendado na documentação oficial do Kubernetes
```sh
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
Verifique se os pods estão sendo criados e estão em `running`, e se o Nó esta `Ready`
```sh
kubectl get pods --all-namespaces
kubectl get nodes
```
### 8. Iniciando os Demais Master Nodes
execute o comando exibido na saida de inicialização do primeiro master node
```sh
kubeadm join LOAD_BALANCER_DNS:6443 ....
```

## 9. Iniciando os Workers Nodes
execute o comando exibido na saida de inicialização do primeiro master node
```sh
kubeadm join  ....
```

## 10. Instalar Helm no Cluster
Utilizado a documentação oficial do [Helm](https://helm.sh/docs/intro/quickstart/)

Baixe o Helm
```sh
wget https://get.helm.sh/helm-v3.2.4-linux-amd64.tar.gz
tar -xvzf helm-v3.2.4-linux-amd64.tar.gz
```
Copie o executável do Helm para a pasta `/usr/local/bin`, e ajuste a permissão para garantir a execução.
```sh
cp linux-amd64/helm /usr/local/bin
chmod 755 /usr/local/bin/helm
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
## 11. Instalar Prometheus Operator no Cluster par Monitoração
Primeiramente vamos criar um `namespace` com o nome `monitoring` no cluster para receber o deploy do `Prometheus Operator`.
```sh
kubectl create namespace monitoring
```
Após isso vamos executar o comando `helm` abaixo para instalar o `Prometheus Operator`
```sh
helm install prometheus-operator my-release stable/prometheus-operator --namespace monitoring
```
## 12 Instalar Ingress através do Helm
Para Ingres vamos utilizar o [Traefik](https://docs.traefik.io/).

Adicione repositorio do `Traefik` no `Helm`.

```sh
helm repo add traefik https://containous.github.io/traefik-helm-chart
```

Atualize a lista de repositorios do `Helm`.

```sh
helm repo update
```

Crie um `namespace` para instalarmos o `Traefik`, assim deixamos mais organizado e segredado os workloads de nosso cluster 
```sh
kubectl create namespace traefik
```

Agora vamos instalar o `Chart` do `Traefik`

```sh
helm install traefik traefik/traefik --namespace traefik
```
Verifique se os todos recursos do `Traefik` no cluster estão OK.

```sh
kubectl get all -n traefik
```
Se tudo estive OK, vamos acessar o `Dashboard` do `Traefik`

```sh
kubectl port-forward $(kubectl get pods --selector "app.kubernetes.io/name=traefik" --output=name) 9000:9000
```
Acesse a URL: http://127.0.0.1:9000/dashboard/





