# Criando um laboratório Kubernetes com Cluster API (CAPI) e Proxmox  

# O que é o Cluster API (CAPI)?

O **Cluster API (CAPI)** é um subprojeto do Kubernetes que permite criar, provisionar, atualizar, escalar e remover clusters Kubernetes utilizando a própria API do Kubernetes.

Em vez de criar máquinas virtuais manualmente, instalar Kubernetes e repetir esse processo sempre que precisar de um novo cluster, o CAPI transforma tudo isso em objetos do Kubernetes.

Em outras palavras:

> Você passa a "pedir" um cluster Kubernetes da mesma forma que cria um Deployment ou um Pod.

O Cluster API cuida automaticamente da criação das VMs, instalação do Kubernetes e gerenciamento do ciclo de vida do cluster.

---

# O que é o Kind?

Existe um projeto chamado **Kind (Kubernetes IN Docker)**.

O Kind executa um cluster Kubernetes completo dentro de containers Docker.

Ele foi criado inicialmente para testes, desenvolvimento e CI/CD, mas também é perfeito para servir como **Genesis Cluster** do Cluster API.

Neste laboratório, o Kind será responsável apenas por hospedar os componentes do Cluster API.

Os clusters criados pelo CAPI serão executados no Proxmox.

A arquitetura ficará assim:

```text
                 Bastion
                    │
         Docker + Kind + Cluster API
                    │
          (Genesis Cluster)
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
   Produção   Homologação   Laboratório
        (Clusters criados automaticamente no Proxmox)
```

---

# Por onde começar?

Antes de qualquer coisa você precisa de um servidor executando o Proxmox.

No meu laboratório utilizei um hardware relativamente simples, montado com peças de baixo custo.

## Hardware utilizado

| Componente | Especificação |
|------------|---------------|
| Placa-mãe | SZMZ |
| Memória | 64 GB DDR4 |
| Processador | Intel Xeon E5-2682 |
| Socket | LGA2011 |

Obviamente qualquer hardware compatível com o Proxmox poderá ser utilizado.

---

# O que fazer depois de instalar o Proxmox?

Depois do Proxmox funcionando, o próximo passo é criar uma máquina virtual que servirá como **Bastion**.

Toda a administração será feita a partir dessa máquina.

Ela será responsável por:

- instalar o Docker;
- instalar o Kind;
- instalar o kubectl;
- instalar o clusterctl;
- criar o Management Cluster;
- gerenciar todos os clusters Kubernetes.

---

# Setup da Bastion

## Atualizar o sistema

```bash
sudo apt-get update
```

---

## Instalar o Docker

```bash
sudo apt-get install -y docker.io

sudo systemctl enable --now docker
```

Verifique a instalação:

```bash
docker version
```

---

## Instalar o kubectl

O kubectl será utilizado para interagir com os clusters Kubernetes.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/
```

Verifique:

```bash
kubectl version --client
```

---

## Instalar o Kind

Baixe o binário:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
```

Permissões:

```bash
chmod +x ./kind
```

Mover para o PATH:

```bash
sudo mv ./kind /usr/local/bin/
```

Verifique:

```bash
kind version
```

---

## Instalar o clusterctl

O clusterctl é a ferramenta oficial do Cluster API.

Ela será utilizada para instalar os componentes do CAPI e gerar manifests para criação de clusters.

Download:

```bash
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.7.3/clusterctl-linux-amd64 -o clusterctl
```

Permissões:

```bash
chmod +x clusterctl
```

Mover para o PATH:

```bash
sudo mv clusterctl /usr/local/bin/
```

Verifique:

```bash
clusterctl version
```

---

# Próximo passo

Agora que a Bastion está pronta, o próximo passo será criar o **Cluster Genesis** utilizando o Kind.  

Esse cluster hospedará todos os componentes do Cluster API e será responsável por criar automaticamente os clusters Kubernetes dentro do Proxmox.  

Crie diretório para organizar as coisas:  
mkdir /opt/cluster-api/  
Crie um arquivo  com o seguinte conteúdo  
vim kind-config.yaml 

```bash
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
```

Em seguida crie o cluster  

kind create cluster --name capi-genesis --config kind-config.yaml  
<img width="593" height="264" alt="image" src="https://github.com/user-attachments/assets/e388599f-66c5-42cf-83c6-c6c5b8e1a522" />

Com o cluster criado valide só para sacar como fica:  

kubectl get nodes  

<img width="586" height="82" alt="image" src="https://github.com/user-attachments/assets/13be2289-c60f-4a25-a53f-f14733064632" />

Agora eu vou inicializar Cluster API passando o provider sendo proxmox ( se fosse vmware era so trocar o nome )  
clusterctl init --infrastructure proxmox  

<img width="844" height="252" alt="image" src="https://github.com/user-attachments/assets/8e1216a2-a04f-4ab3-b60a-417f314b1c83" />

Instale também 

clusterctl init --ipam in-cluster    



Legal, agora preciso criar o token no Proxmox que vai ser usado pelo cluster kind para se conectar ao proxmox (Tome nota do Token)
Gera conforme mostrado acima. (Clique na opção Datacenter conforme mostrado e localize a API Tokens)  

<img width="922" height="614" alt="image" src="https://github.com/user-attachments/assets/884a3e8b-8170-48f1-a461-a7c95f61a665" />

Logue no servidor do proxmox aqui no meu lab é 192.168.18.31 e altere as roles  
pveum acl modify /nodes --token 'root@pam!kubernetes' --roles Administrator  

## Configurando as credenciais do Proxmox

O **Cluster API Provider for Proxmox (CAPMOX)** não permite informar as credenciais de acesso (URL da API, Token ID e Token Secret) diretamente no manifesto do cluster.

Por questões de segurança, essas informações devem ser armazenadas em um **Secret** do Kubernetes, que será referenciado pelo recurso `ProxmoxCluster`.  

Crie um arquivo chamado `proxmox-credentials.yaml` com o seguinte conteúdo:  

vim proxmox-credentials.yaml  

```bash
apiVersion: v1
kind: Secret
metadata:
  name: proxmox-credentials
  namespace: default
type: Opaque
stringData:
  url: "https://192.168.18.31:8006/"
  token: "root@pam!kubernetes"
  secret: "766a07cb-a0bd-4fef-8581-e3ad7059c809"

```

<img width="360" height="182" alt="image" src="https://github.com/user-attachments/assets/ec83cc5f-dfa2-48f3-90e4-22cdeed58c2c" />  


**Aplique no Cluster** 


kubectl apply -f proxmox-credentials.yaml

# Criando o primeiro cluster Kubernetes no Proxmox

Com o **Management Cluster** e o **Cluster API Provider for Proxmox (CAPMOX)** já instalados, o próximo passo é criar o manifesto que será responsável por provisionar automaticamente o cluster Kubernetes no Proxmox.

---

## Criando o manifesto do cluster

Crie um arquivo chamado **`cluster-lab.yaml`**.

```bash
vim cluster-lab.yaml
```
Copie todo o conteúdo do arquivo que está na raiz do projeto.(Altere de acordo com suas necessidades)

> **Importante:**
Entenda os principais pontos e como eles funcionam:
> 
**Quando o manifesto é aplicado, a aplicação pega o primeiro endereço de IP do pool e fixa ele para ser usado na interface de rede. Esse IP é virtual e é por ele que a API vai ficar escutando.**
> 
>
> - Endereço IP da API (Control Plane)  
    controlPlaneEndpoint:  
    host: 192.168.18.202  
    port: 6443

> - DNS Serve para resolução de nomes externos e não do cluster DNS  
    dnsServers:  
    8.8.8.8  
    1.1.1.1  
  - 
> - O primeiro IP no caso o 203 vai ser alocado na interface de rede real para a vm que será o Control-Plane.
> - Os outros IP 204 e 205 serão alocados na interface de rede para as vms Workers e o gateway é o da rede padrão.
    dnsServers:
    ipv4Config:
    addresses:
    - 192.168.18.203
    - 192.168.18.204
    - 192.168.18.205
    gateway: 192.168.18.1
    prefix: 24
    

> - Gateway
> - DNS
> - Faixa de IPs das máquinas
> - Nome do nó do Proxmox (`sourceNode`)
> - ID do template (`templateID`)
> - Storage utilizado (`storage`)
> - Recursos de CPU e Memória
> - Versão do Kubernetes
> - Interface de rede utilizada pelo kube-vip (`vip_interface`)
> - Nome do Secret contendo as credenciais do Proxmox (`credentialsRef`)

```yaml
# Cole aqui o conteúdo completo do arquivo cluster-lab.yaml
```

---

## Estrutura do manifesto

O arquivo é composto pelos seguintes recursos:

| Recurso | Função |
|----------|--------|
| **Cluster** | Define o cluster Kubernetes que será criado. |
| **ProxmoxCluster** | Configura a infraestrutura do Proxmox, incluindo endpoint da API, DNS, gateway e credenciais. |
| **KubeadmControlPlane** | Configura os nós de Control Plane e o bootstrap do Kubernetes. |
| **ProxmoxMachineTemplate (Control Plane)** | Define como as VMs do Control Plane serão criadas no Proxmox. |
| **MachineDeployment** | Define a quantidade de Workers e gerencia seu ciclo de vida. |
| **KubeadmConfigTemplate** | Configuração utilizada durante o join dos Workers. |
| **ProxmoxMachineTemplate (Workers)** | Template utilizado para criação das VMs dos Workers. |

---

## Aplicando o manifesto

Após revisar o arquivo, aplique-o no Management Cluster.

```bash
kubectl apply -f cluster-lab.yaml
```

---

## Acompanhando a criação do cluster

O Cluster API começará a criar todos os recursos necessários.

Você pode acompanhar a evolução utilizando os comandos abaixo.

Verificando o Cluster:

```bash
kubectl get cluster -w
```

Verificando as máquinas:

```bash
kubectl get machines -w
```

Verificando o Control Plane:

```bash
kubectl get kubeadmcontrolplane -w
```

Verificando o Machine Deployment:

```bash
kubectl get machinedeployment -w
```

Verificando os recursos do Proxmox:

```bash
kubectl get proxmoxmachines -w
```

---

## Acompanhando pelo Proxmox

Durante a criação do cluster será possível observar automaticamente:

- Criação das máquinas virtuais;
- Clonagem do template informado;
- Configuração do Cloud-Init;
- Inicialização das VMs;
- Instalação do Kubernetes;
- Bootstrap do Control Plane;
- Entrada dos nós Workers no cluster.

Todo esse processo é realizado automaticamente pelo Cluster API.

---

## Obtendo o kubeconfig do cluster

Quando todas as máquinas estiverem em estado **Running**, obtenha o kubeconfig do novo cluster.

```bash
clusterctl get kubeconfig lab-kubeadm > lab-kubeadm.kubeconfig
```

---

## Acessando o cluster

Utilize o kubeconfig gerado para validar a criação do cluster.

```bash
kubectl --kubeconfig lab-kubeadm.kubeconfig get nodes
```

O resultado esperado será semelhante ao exemplo abaixo:

```text
NAME                              STATUS   ROLES           AGE   VERSION
lab-kubeadm-control-plane-xxxxx   Ready    control-plane   XXm   v1.36.2
lab-kubeadm-workers-xxxxx         Ready    <none>          XXm   v1.36.2
lab-kubeadm-workers-xxxxx         Ready    <none>          XXm   v1.36.2
```

Neste momento o cluster Kubernetes já estará operacional e pronto para receber aplicações.

          























