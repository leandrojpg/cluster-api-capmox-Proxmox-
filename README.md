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
## 🌐 Arquitetura de Rede do Cluster (CAPMOX)

Quando o manifesto é aplicado, o CAPMOX captura o primeiro endereço de IP disponível no pool de IPAM (neste caso, o `192.168.18.203`) e o fixa como um **IP Virtual (VIP)**. É através deste IP virtual que a API do Kubernetes (`kube-apiserver`) ficará escutando todo o cluster (bastion, workers e administradores) de forma estável.

## 📌 Configuração do Cluster (YAML)

> 💡 **Entendendo a Arquitetura e Funções de Rede:**
> * **`controlPlaneEndpoint` (.202):** É o **IP Virtual (VIP)** unificado. A API do Kubernetes (`kube-apiserver`) escuta por ele para garantir um acesso único e flutuante.
> * **Nó Control Plane (.203):** IP real alocado na interface física da VM que assume o papel de Control Plane.
> * **Nós Workers (.204 e .205):** IPs alocados sequencialmente nas interfaces reais das VMs Workers.
> * **`dnsServers` (8.8.8.8 / 1.1.1.1):** DNS externo para as VMs navegarem na internet; não afeta o CoreDNS interno do cluster.
> * **Versão do Kubernetes:** `v1.36.2`
>
> 🖥️ **`template.spec` (Dimensionamento das VMs no Proxmox):** Define as especificações de hardware das instâncias que serão clonadas a partir do Template ID `9000` no storage `DATASTORE-04-NVME`. Cada máquina nascerá configurada com 2 Cores, 1 Socket, 4GB de RAM, um disco de boot SCSI de 40GB e conectada à bridge de rede `vmbr0` usando o driver otimizado `virtio`.
>
> 🪪 **`provider-id: "proxmox://{{ ds.meta_data.instance_id }}"` (Identidade do Nó):** Funciona como o **"RG"** da máquina virtual. No boot, o Cloud-Init troca essa variável pelo ID real da VM no Proxmox (ex: `proxmox://101`). Isso vincula o nó do `kubectl get nodes` à VM física para que o CAPMOX saiba exatamente quem gerenciar, escalar ou destruir.
>
> 🛣️ **`clusterConfiguration` (Rede e SSL):** O `podSubnet` define a faixa de IPs que os Pods usarão internamente. O `certSANs` lista os IPs (`.202`, `.203`) e nomes autorizados no certificado SSL da API, evitando erros de TLS no `kubectl`.
>
> ⏱️ **`postKubeadmCommands` (Automação Pós-Boot):** Executa em lote uma trava de segurança que aguarda o VIP (`.202`) responder, instala o pacote do Calico CNI e aguarda os DaemonSets do Calico estarem criados para finalizar o deploy sem erros de concorrência.

```yaml
# Endpoint de comunicação unificado do Control Plane (VIP)
controlPlaneEndpoint:  
  host: 192.168.18.202  
  port: 6443

# Configuração de rede e alocação de IPs via IPAM (CAPMOX)
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

# Rede dos Pods e Certificados SSL da API
clusterConfiguration:
  networking:
    podSubnet: "10.244.0.0/16"
  apiServer:
    certSANs:
      - "192.168.18.202"
      - "192.168.18.203"
      - "lab-kubeadm-control-plane"

# Registro de identidade do Nó Inicial (Control Plane)
initConfiguration:
  nodeRegistration:
    kubeletExtraArgs:
      provider-id: "proxmox://{{ ds.meta_data.instance_id }}"

# Registro de identidade dos demais nós (Workers)
joinConfiguration:
  nodeRegistration:
    kubeletExtraArgs:
      provider-id: "proxmox://{{ ds.meta_data.instance_id }}"

# Comandos executados sequencialmente pós-inicialização
postKubeadmCommands:
  # 1. Aguarda o VIP do control plane ficar disponível
  - |
    until curl -sk [https://192.168.18.202:6443/healthz](https://192.168.18.202:6443/healthz) >/dev/null 2>&1; do
      echo "Aguardando VIP 192.168.18.202..."
      sleep 5
    done
  # 2. Instala o Calico CNI
  - kubectl --kubeconfig /etc/kubernetes/admin.conf apply -f [https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml)

  # 3. Aguarda o DaemonSet do Calico estar pronto
  - |
    until kubectl --kubeconfig /etc/kubernetes/admin.conf get daemonset calico-node -n kube-system >/dev/null 2>&1; do
      echo "Aguardando DaemonSet calico-node ser criado..."
      sleep 3
    done

# Configuração de hardware e origem das VMs no Proxmox
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

```

Agora que as principais opções foram explicadas basta aplicar o manifesto.    
kubectl apply -f cluster-lab.yaml  

O que acontece no proxmox depois de aplicado?  
O cluster que instalamos no bastion irá se comunicar com o Promox e irá criar nosso primeiro cluster utilizando os parametros que passamos no arquivo cluster-lab.yaml  
Observe que ele criou as 3 vms.  

<img width="1911" height="740" alt="image" src="https://github.com/user-attachments/assets/bba9185c-611e-4e36-9514-6807f0d43c74" />

Se voce digitar no bastion o comando abaixo, poderá ver as vms sendo criadas também assim que o manifesto é aplicado.  
Ponto importante as vms precisam ter o status de Running pois isso garente que toda comunicação e instrução passada está funcionando.
kubectl get machines -w

<img width="1145" height="100" alt="image" src="https://github.com/user-attachments/assets/322f82ba-908a-4802-86eb-7434cda24f69" />

## 📌 Migrando o Capi Kubernetes do Bastion para o Proxmox

O que foi feito até aqui?

> 💡 **Criamos um cluster simples chamado de Genesis por que precisamos partir de algum lugar para criar o cluster no proxmox e o kind é perfeito pra isso.**

> 💡 **Em seguida instalamos nesse cluster chamado capi-genesis todos os componentes do Capmox para que possa falar com o Cluster do Proxmox.**
> 
> 💡 **Criamos um arquivo chamado proxmox-credentials.yaml que tem o token usado pelo capi e outras configuracoes que garantem a autenticação pelo cluster genesis ao Proxmox**

> 💡 **Criamos um arquivo chamado cluster-lab.yaml  com instrucoes claras o que deve ser criado no Proxmox, como vms,configuracao da rede, controlplane,workers e etc..**

**"Até agora, toda a criação do nosso ambiente dependia de uma estrutura temporária (o cluster na máquina bastion). Isso funcionou para o começo, mas não é o jeito profissional. Aquele primeiro cluster foi só o nosso ponto de partida.** 
**O correto agora é migrarmos todo o controle para o novo Cluster definitivo que criamos dentro do Proxmox. A partir de agora, é esse cluster principal que vai gerenciar e gerar tudo sozinho. Se amanhã você precisar criar um ambiente de produção real, é essa nova** **estrutura que você vai usar.**

**"E como fazemos essa mudança? O processo funciona assim:**

**Pegamos a chave de acesso: Primeiro, nós pegamos o token (a credencial de acesso) do novo cluster que foi criado no Proxmox.**

**Instalamos as ferramentas (CAPMOX): Usamos essa chave para entrar nele e instalar os componentes do CAPMOX. Fazemos isso porque, até esse momento, ele é apenas um cluster Kubernetes comum e 'cego' — ele não sabe nada sobre as máquinas virtuais do Proxmox.**

**A transferência de poder: Ao instalar esses pacotes do CAPMOX dentro dele, damos a ele a estrutura base para que ele entenda o Proxmox. É nesse momento que ele ganha superpoderes e se torna oficialmente o 'dono do jogo', pronto para assumir o controle total e gerenciar a si mesmo e a outros clusters."**



🔑 **Pegamos a chave de acesso:** Primeiro, nós pegamos o token (a credencial de acesso) do novo cluster que foi criado no Proxmox.
A regra é sempre uma receita de bolo: [Nome exato do seu Cluster] + -kubeconfig.  

```bash
clusterctl get kubeconfig lab-kubeadm > lab-kubeadm.kubeconfig

```

Para evitar de ficar passando o arquivo toda hora podemos usar a variavel KUBECONFIG  assim todos os comandos que emitirmos com kubectl.. será diretamente no cluster que foi criado no Proxmox  
```bash
export KUBECONFIG=~/cluster-api/lab-kubeadm.kubeconfig
```
Experimente

```bash
 kubectl get nodes -A
```
<img width="579" height="105" alt="image" src="https://github.com/user-attachments/assets/0792f0ad-7186-46fc-9143-84142e005667" />

Validando que nao temos nada instalado no novo cluster  
kubectl get providers -A  
<img width="494" height="70" alt="image" src="https://github.com/user-attachments/assets/7a510a1f-9e9a-41bb-bfa0-7998e6309d79" />

Instalando:  
clusterctl init --infrastructure proxmox --ipam in-cluster  
<img width="874" height="164" alt="image" src="https://github.com/user-attachments/assets/091e18e5-afc2-46fd-b1a8-d2e5c41031c9" />

Repita o comando e veja a diferenca:  
kubectl get providers -A  

<img width="824" height="154" alt="image" src="https://github.com/user-attachments/assets/15b16a7a-e02c-45a3-9623-9f28aa5df9a1" />  


Voltar para o Management Cluster (Kind)  
unset KUBECONFIG  

Agora sim podemos fazer o move   
clusterctl move --to-kubeconfig lab-kubeadm.kubeconfig  

<img width="1736" height="309" alt="image" src="https://github.com/user-attachments/assets/aa33ea59-e20a-4610-abbd-a3b8a2da6c32" />




"Cuidado ao executar clusterctl move: após a migração, o cluster destino passa a ser o Management Cluster. Ele não deve ser destruído como workload, pois os controladores CAPI/CAPMOX residem nele."     













 




























