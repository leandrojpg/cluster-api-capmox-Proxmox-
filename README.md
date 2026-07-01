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

Ele foi criado inicialmente para testes, desenvolvimento e CI/CD, mas também é perfeito para servir como **Management Cluster** do Cluster API.

Neste laboratório, o Kind será responsável apenas por hospedar os componentes do Cluster API.

Os clusters criados pelo CAPI serão executados no Proxmox.

A arquitetura ficará assim:

```text
                 Bastion
                    │
         Docker + Kind + Cluster API
                    │
          (Management Cluster)
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
   Produção   Homologação   Laboratório
        (Clusters criados automaticamente no Proxmox)
```

---

# Por onde começar?

Antes de qualquer coisa você precisa possuir um servidor executando o Proxmox.

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

Em seguida crie o cluster 
kind create cluster --name capi-genesis --config kind-config.yaml  
<img width="593" height="264" alt="image" src="https://github.com/user-attachments/assets/e388599f-66c5-42cf-83c6-c6c5b8e1a522" />

Como cluster criado valide só para sacar como fica:
kubectl get nodes
<img width="586" height="82" alt="image" src="https://github.com/user-attachments/assets/13be2289-c60f-4a25-a53f-f14733064632" />

Agora eu vou inicializar Cluster API passando o provider sendo Proxmox ( se fosse vmware era so trocar o nome )  
clusterctl init --infrastructure proxmox
<img width="844" height="252" alt="image" src="https://github.com/user-attachments/assets/8e1216a2-a04f-4ab3-b60a-417f314b1c83" />

Legal agora preciso criar o token no Proxmox que vai ser usado pelo cluster kind que tem o promox se conectar ao proxmox (Tome nota do Token)

<img width="922" height="614" alt="image" src="https://github.com/user-attachments/assets/884a3e8b-8170-48f1-a461-a7c95f61a665" />

Logue no servidor do proxmox aqui no meu lab é 192.168.18.31 e altere as roles  
pveum acl modify /nodes --token 'root@pam!kubernetes' --roles Administrator  

O provedor do Proxmox da IONOS Cloud (cluster-api-provider-proxmox) não aceita receber a URL e o Token diretamente, sendo assim precisamos criar um secret 
crie um arquivo proxmox-credentials.yaml
vim proxmox-credentials.yaml

<img width="360" height="182" alt="image" src="https://github.com/user-attachments/assets/ec83cc5f-dfa2-48f3-90e4-22cdeed58c2c" />




















