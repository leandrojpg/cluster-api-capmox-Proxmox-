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

Agora que a Bastion está pronta, o próximo passo será criar o **Management Cluster** utilizando o Kind.

Esse cluster hospedará todos os componentes do Cluster API e será responsável por criar automaticamente os clusters Kubernetes dentro do Proxmox.

No próximo capítulo criaremos o primeiro cluster Kind e inicializaremos o Cluster API.
