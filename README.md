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

Agora eu vou inicializar Cluster API passando o provider sendo Proxmox ( se fosse vmware era so trocar o nome )  
clusterctl init --infrastructure proxmox
<img width="844" height="252" alt="image" src="https://github.com/user-attachments/assets/8e1216a2-a04f-4ab3-b60a-417f314b1c83" />

Instale também 
clusterctl init --ipam in-cluster


Legal, agora preciso criar o token no Proxmox que vai ser usado pelo cluster kind para se conectar ao proxmox (Tome nota do Token)
Gera conforme mostrado acima.

<img width="922" height="614" alt="image" src="https://github.com/user-attachments/assets/884a3e8b-8170-48f1-a461-a7c95f61a665" />

Logue no servidor do proxmox aqui no meu lab é 192.168.18.31 e altere as roles  
pveum acl modify /nodes --token 'root@pam!kubernetes' --roles Administrator  

## Configurando as credenciais do Proxmox

O **Cluster API Provider for Proxmox (CAPMOX)** não permite informar as credenciais de acesso (URL da API, Token ID e Token Secret) diretamente no manifesto do cluster.

Por questões de segurança, essas informações devem ser armazenadas em um **Secret** do Kubernetes, que será referenciado pelo recurso `ProxmoxCluster`.

Crie um arquivo chamado `proxmox-credentials.yaml` com o seguinte conteúdo:
crie um arquivo proxmox-credentials.yaml
vim proxmox-credentials.yaml

<img width="360" height="182" alt="image" src="https://github.com/user-attachments/assets/ec83cc5f-dfa2-48f3-90e4-22cdeed58c2c" />

kubectl apply -f proxmox-credentials.yaml

Agora crie um arquivo chamado cluster-lab.yaml com o conteúdo abaixo:
vim cluster-lab.yaml

apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: lab-kubeadm
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
        - 10.244.0.0/16

  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
    kind: ProxmoxCluster
    name: lab-kubeadm

  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: lab-kubeadm-control-plane

---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: ProxmoxCluster
metadata:
  name: lab-kubeadm
  namespace: default
spec:
  controlPlaneEndpoint:
    host: 192.168.18.202
    port: 6443

  dnsServers:
    - 8.8.8.8
    - 1.1.1.1

  ipv4Config:
    addresses:
      - 192.168.18.203
      - 192.168.18.204
      - 192.168.18.205
    gateway: 192.168.18.1
    prefix: 24

  credentialsRef:
    name: proxmox-credentials
    namespace: default

---
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: lab-kubeadm-control-plane
  namespace: default
spec:
  replicas: 1
  version: v1.36.2

  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
      kind: ProxmoxMachineTemplate
      name: lab-kubeadm-control-plane

  kubeadmConfigSpec:
    initConfiguration:
      nodeRegistration:
        kubeletExtraArgs:
          provider-id: "proxmox://{{ ds.meta_data.instance_id }}"

    clusterConfiguration:
      networking:
        podSubnet: "10.244.0.0/16"

      apiServer:
        certSANs:
          - "192.168.18.202"
          - "192.168.18.203"
          - "lab-kubeadm-control-plane"

    files:
      - path: /etc/kubernetes/manifests/kube-vip.yaml
        owner: root:root
        permissions: "0644"
        content: |
          apiVersion: v1
          kind: Pod
          metadata:
            name: kube-vip
            namespace: kube-system
          spec:
            containers:
              - name: kube-vip
                image: ghcr.io/kube-vip/kube-vip:v0.8.7
                args:
                  - manager
                env:
                  - name: vip_arp
                    value: "true"
                  - name: port
                    value: "6443"
                  - name: vip_interface
                    value: ens18
                  - name: cp_enable
                    value: "true"
                  - name: cp_namespace
                    value: kube-system
                  - name: vip_leaderelection
                    value: "true"
                  - name: vip_leaseduration
                    value: "5"
                  - name: vip_renewdeadline
                    value: "3"
                  - name: vip_retryperiod
                    value: "1"
                  - name: address
                    value: "192.168.18.202"
                securityContext:
                  capabilities:
                    add:
                      - NET_ADMIN
                      - NET_RAW
                      - SYS_TIME
                volumeMounts:
                  - mountPath: /etc/kubernetes/admin.conf
                    name: kubeconfig

            hostAliases:
              - hostnames:
                  - kubernetes
                ip: "192.168.18.203"

            hostNetwork: true

            volumes:
              - hostPath:
                  path: /etc/kubernetes/admin.conf
                name: kubeconfig

    postKubeadmCommands:
      - |
        until curl -sk https://192.168.18.202:6443/healthz >/dev/null 2>&1; do
          echo "Aguardando VIP 192.168.18.202..."
          sleep 5
        done

      - kubectl --kubeconfig /etc/kubernetes/admin.conf apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml

      - |
        until kubectl --kubeconfig /etc/kubernetes/admin.conf get daemonset calico-node -n kube-system >/dev/null 2>&1; do
          echo "Aguardando DaemonSet calico-node ser criado..."
          sleep 3
        done

      - |
        kubectl --kubeconfig /etc/kubernetes/admin.conf set env daemonset/calico-node -n kube-system FELIX_HEALTHHOST=127.0.0.1

      - kubectl --kubeconfig /etc/kubernetes/admin.conf delete pods -n kube-system -l k8s-app=calico-node --ignore-not-found

---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: ProxmoxMachineTemplate
metadata:
  name: lab-kubeadm-control-plane
  namespace: default
spec:
  template:
    spec:
      sourceNode: proxmox
      templateID: 9000
      storage: DATASTORE-04-NVME

      memoryMiB: 4096
      numCores: 2
      numSockets: 1

      format: qcow2
      full: true

      disks:
        bootVolume:
          disk: scsi0
          sizeGb: 40

      network:
        default:
          bridge: vmbr0
          model: virtio

---
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: lab-kubeadm-workers
  namespace: default
spec:
  clusterName: lab-kubeadm
  replicas: 2

  selector:
    matchLabels:
      cluster.x-k8s.io/deployment-name: lab-kubeadm-workers

  template:
    metadata:
      labels:
        cluster.x-k8s.io/deployment-name: lab-kubeadm-workers

    spec:
      clusterName: lab-kubeadm
      version: v1.36.2

      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: lab-kubeadm-worker

      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
        kind: ProxmoxMachineTemplate
        name: lab-kubeadm-worker

---
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfigTemplate
metadata:
  name: lab-kubeadm-worker
  namespace: default
spec:
  template:
    spec:
      joinConfiguration:
        nodeRegistration:
          kubeletExtraArgs:
            provider-id: "proxmox://{{ ds.meta_data.instance_id }}"

---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: ProxmoxMachineTemplate
metadata:
  name: lab-kubeadm-worker
  namespace: default
spec:
  template:
    spec:
      sourceNode: proxmox
      templateID: 9000
      storage: DATASTORE-04-NVME

      memoryMiB: 4096
      numCores: 2
      numSockets: 1

      format: qcow2
      full: true

      disks:
        bootVolume:
          disk: scsi0
          sizeGb: 40

      network:
        default:
          bridge: vmbr0
          model: virtio

          























