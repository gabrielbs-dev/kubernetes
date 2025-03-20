# Instalação e Configuração do Kubernetes no Ubuntu 22

Este documento descreve o passo a passo para instalar e configurar um cluster Kubernetes em um sistema Linux, configurando a rede, o runtime de containers, o cgroup, o swap, e outros componentes essenciais.

## Instalando os requisitos

Antes de instalar o Kubernetes, você precisa instalar alguns pacotes essenciais que permitem a instalação e configuração adequadas.

```bash
apt install -y apt-transport-https ca-certificates curl gpg
```

## Instalando o Kubernetes

O Kubernetes será instalado a partir do repositório oficial. Primeiro, você deve adicionar a chave GPG do repositório do Kubernetes e configurar a lista de fontes de pacotes.

1\. Adicione a chave GPG:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

2\. Adicione o repositório do Kubernetes:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

3\. Atualize os pacotes:

```bash
apt update
```

4\. Instale o Kubernetes:

```bash
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

5\. Habilite o kubelet para iniciar com o sistema:

```bash
systemctl enable kubelet
```

## Configurando a rede

Configurações de rede são necessárias para que os pods se comuniquem corretamente dentro do cluster Kubernetes

1\. Carregue os módulos necessários:

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

2\. Carregue os módulos de rede:

```bash
modprobe overlay
modprobe br_netfilter
```

3\. Configure as permissões de rede:

```bash
tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

4\. Atualize as configurações de sistema:

```bash
sysctl --system
```

## Instalando o Container Runtime

O Kubernetes precisa de um runtime de container para executar os pods. Você pode escolher entre Containerd ou Docker. O exemplo a seguir detalha a instalação e configuração do Containerd.

1\. Instalando Dependências Necessárias:

```bash
apt install -y gnupg lsb-release
```

2\. Criando o Diretório para as Chaves GPG:

```bash
mkdir -p /etc/apt/keyrings
```

3\. Baixando e Armazenando a Chave GPG do Docker:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

4\. Adicionando o Repositório Docker:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

5\. Atualizando os Repositórios:

```bash
apt update
```

6\. Instalando o Containerd:

```bash
apt install -y containerd.io
```

## Configure a versão do SystemdCgroup no Containerd

1\. Limpe o arquivo de configuração do Containerd:

```bash
truncate -s 0 /etc/containerd/config.toml
```

2\. Edite a configuração do Containerd:

```bash
nano /etc/containerd/config.toml
```

3\. Habilite o uso de SystemdCgroup:

```toml
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
   [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
```

4\. Reinicie o containerd:

```bash
systemctl restart containerd
```

## Desativando o SWAP

O Kubernetes exige que o swap esteja desativado para garantir a estabilidade do cluster.

```bash
sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
swapoff -a
```

## Alterando IP do node

Por padrão, o Kubernetes usa o IP da placa de rede padrão do Linux. Caso deseje definir um IP específico, siga os passos abaixo:

1\. Edite o arquivo de variáveis de ambiente do Kubelet:

```bash
nano /etc/default/kubelet
```

2\. Adicione a linha abaixo:

```bash
Environment="--node-ip=IP_DA_SUA_INTERFACE"
```

*Substituia "IP_DA_SUA_INTERFACE" pelo IP desejado.*

## Finalizando e reiniciando

Após as configurações, é necessário reiniciar o sistema para garantir que todas as configurações entrem em vigor.

```bash
reboot
```

## Inicializando o Master Node

Para inicializar o master node, utilize o kubeadm com os parâmetros apropriados.

```bash
kubeadm init --control-plane-endpoint IP_NODE --apiserver-advertise-address IP_NODE --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=NumCPU
```

*Substituia "IP_NODE" pelo seu respectivo valor.*

## Instalando a Proxy

Flannel é uma solução de rede para Kubernetes que facilita a comunicação entre pods.

1\. Realize o download do arquivo de configuração do Flannel.

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

2\. Caso deseje alterar a interface que o Flannel utilizará para a comunicação, edite o arquivo abaixo. Do contrário, pule para etapa 4.

```bash
nano kube-flannel.yml
```

3\. Procure pelo DaemonSet "kube-flannel-ds" e em "containers" adicione o argumento `--iface=INTERFACE`, ficando algo como:

```yaml
containers:
  - name: kube-flannel
    image: quay.io/coreos/flannel:v0.10.0-amd64
    command:
    - /opt/bin/flanneld
    args:
    - --ip-masq
    - --kube-subnet-mgr
    - --iface=enp7s0
```

4\. Aplique a configuração:

```bash
kubectl apply -f kube-flannel.yml
```

## Vinculando o Worker ao cluster

Depois de configurar o master, você precisa vincular os worker nodes ao cluster com o token gerado no processo de inicialização.

```bash
kubeadm join IP_MASTER --token TOKEN --discovery-token-ca-cert-hash CERTIFICADO
```

*Substituia "IP_MASTER", "TOKEN" e "CERTIFICADO" pelos seus respectivos valores.*

## Instalando o Helm

O Helm é uma ferramenta que facilita o deploy.

1\. Instalar o Helm:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Instalando o Metallb (apenas no master)

Metallb fornece serviços de Load Balancer para clusters Kubernetes.

1\. Instalar o Metallb com o Helm:

```bash
helm install metallb metallb --repo https://metallb.github.io/metallb --namespace metallb-system --create-namespace
```

2\. Crie o arquivo de configuração:

```bash
nano metallb.yaml
```

3\. Cole o conteúdo abaixo substituindo "PUBLIC_IP" pelo IP público:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - [PUBLIC_IP]-[PUBLIC_IP]
```

4\. Aplique a configuração:

```bash
kubectl apply -f metallb.yaml
```

## Instalando o Ingress Nginx (apenas no master)

O Ingress Nginx é usado para gerenciar o acesso web externo aos serviços em Kubernetes.

1\. Instalar o Ingress Nginx com o Helm:

```bash
helm install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace
```

## Referências

- https://akyriako.medium.com/install-kubernetes-on-ubuntu-20-04-f1791e8cf799
- https://kubernetes.github.io/ingress-nginx/deploy/
- https://metallb.universe.tf/installation/
- https://helm.sh/docs/intro/install/
- https://stackoverflow.com/questions/75637481/k3s-metallb-metallb-controller-failed-to-allocate-ip-for-default-nginx-no
- https://gjhenrique.com/cgroups-k8s/
- https://stackoverflow.com/questions/47845739/configuring-flannel-to-use-a-non-default-interface-in-kubernetes
