# 🏢 Manual de Implementação: Servidor All-in-One para Pequenas Empresas

Este repositório contém o guia passo a passo para a montagem, configuração e deploy de um servidor corporativo baseado no hipervisor **Proxmox VE**, utilizando armazenamento virtualizado e containers integrados.

## 📌 Arquitetura do Projeto

O ecossistema é centralizado em um único servidor físico executando o **Proxmox VE 9.2**. A topologia de dados e serviços divide-se em:
1. **Armazenamento de Arquivos:** Uma VM dedicada rodando **TrueNAS SCALE** recebe, via mapeamento direto por ID físico (Passthrough de bloco), dois HDDs mecânicos para gerenciar um array ZFS espelhado.
2. **Serviços e Containers:** O próprio Proxmox hospeda os containers (LXC). A placa de vídeo física (GPU) é compartilhada diretamente com os containers do host para aceleração de hardware e transcodificação de mídia.
3. **Orquestração de Aplicações:** Dentro do container principal, o **Docker** e o **Portainer** gerenciam os microsserviços da empresa, utilizando **volumes persistentes amarrados ao RAID de dados do TrueNAS**.

### 🌐 Topologia de Rede Híbrida (Resiliência Total)
Para garantir máxima tolerância a falhas e desempenho, o ambiente utiliza duas pontes de rede virtuais (Bridges):
*   **`vmbr0` (Rede Local / Acesso Externo):** Atrelada à placa de rede física do servidor. Permite que você acesse a interface do TrueNAS e os computadores da empresa consumam os serviços dos containers.
*   **`vmbr1` (Rede de Dados Isolada):** Uma ponte puramente virtual (sem placa física atrelada). O tráfego NFS entre o Docker e o TrueNAS trafega por aqui na velocidade da memória RAM. **Se o switch físico da empresa queimar ou for desligado, os containers e o TrueNAS continuam conversando internamente sem interrupções.**

---

## 🛠️ Especificações de Armazenamento e Hardware

*   **SO Host (Proxmox VE):** 1x SSD SATA de 120GB destinado ao Proxmox VE.
*   **Armazenamento de Alta Velocidade (`nvme250`):** 1x SSD NVMe destinado a armazenar imagens ISO, templates e os discos virtuais rápidos.
*   **Armazenamento de Dados (Mídia/Arquivos):** 2x HDDs de 2TB Western Digital (WD Purple - foco em CFTV/Alta durabilidade), mapeados diretamente para a VM de armazenamento.
*   **Gráficos:** 1x GPU AMD Radeon RX 580 256-bits (Destinada ao pass-through de LXC via Host Kernel).

---

## 🚀 Cronograma de Implementação

- [x] **Fase 1:** Instalação do Proxmox VE 9.2 no SSD SATA & Ajuste de Rede Híbrida (`vmbr0` e `vmbr1`).
- [x] **Fase 2:** Criação da VM do TrueNAS SCALE (VM 100) utilizando o Storage NVMe para Boot (32GB).
- [x] **Fase 3:** Passthrough físico por ID dos 2x HDDs WD Purple de 2TB para a VM 100 + Emulação de Seriais.
- [x] **Fase 4:** Configuração do Armazenamento (ZFS Mirror) no TrueNAS SCALE e Exportação NFS via Rede Isolada.
- [x] **Fase 5:** Configuração do Driver AMD no Host Proxmox e Passthrough de GPU para Containers LXC.
- [ ] **Fase 6:** Instalação do Docker + Portainer com Persistência no Storage RAID do TrueNAS via Rede Isolada.
---
## 📖 Passo a Passo de Infraestrutura

### Fase 1: Configuração das Pontes de Rede no Proxmox Host

#### 🔍 1. Identificar o Nome Real da Interface Física
Antes de editar o arquivo de rede, verifique o identificador exato que o Linux atribuiu à sua placa de rede física. No terminal do Proxmox Host, execute:
```bash
ip a
```
### 2. Configuração das Pontes de Rede no Proxmox Host
Para criar o ambiente com a rede física padrão e a rede interna isolada, acesse o shell do host Proxmox e edite as interfaces:
```bash
nano /etc/network/interfaces
````
### 3. Deixe o bloco da ponte principal (vmbr0) com a seguinte estrutura:
```text
auto lo
iface lo inet loopback

iface enp3s0 inet manual

# REDE LOCAL DA EMPRESA (Acesso aos Painéis e Serviços)
auto vmbr0
iface vmbr0 inet dhcp
        bridge-ports enp3s0
        bridge-stp off
        bridge-fd 0

# REDE VIRTUAL ISOLADA (Tráfego de Dados NFS Interno - Anti-Queda)
auto vmbr1
iface vmbr1 inet manual
        bridge-ports none
        bridge-stp off
        bridge-fd 0
```
### 4. Aplique a nova configuração de rede sem reiniciar o servidor físico:
```bash
ifreload -a
```
### Fase 2 e 3: Mapeamento e Vinculação Física dos HDDs para a VM do TrueNAS
🔍 1. Listar os IDs Físicos dos Discos no Servidor
Como o ID de hardware altera conforme o modelo e fabricante do disco instalado, mapeie previamente os IDs dos discos no terminal do Proxmox Host:
```bash
ls -l /dev/disk/by-id/
```
Identifique os dois discos destinados ao armazenamento de dados (ignore partições como -part1 ou discos de boot). Copie o caminho completo que começa com ata-... ou sata-....

🔗 2. Atrelar os Discos à VM 100 (Passthrough por ID)
Execute os comandos trocando os caminhos abaixo pelos IDs capturados no passo anterior:

````bash
qm set 100 -scsi1 /dev/disk/by-id/ID_DO_SEU_DISCO_1
qm set 100 -scsi2 /dev/disk/by-id/ID_DO_SEU_DISCO_2
````
### ⚠️ 3. Correção Crítica: Emulação de Número de Série (Evitar erro de topologia no TrueNAS)
O TrueNAS SCALE exige números de série válidos e únicos para construir a topologia ZFS. Como o barramento virtual padrão do Proxmox mascara os seriais físicos (gerando erro de "duplicate serial numbers: None"), é obrigatório injetar um serial manual.

Acesse o arquivo de configuração da VM 100 pelo terminal do Proxmox:
````bash
nano /etc/pve/qemu-server/100.conf
````
### Localize as linhas correspondentes ao scsi1 e scsi2 e adicione o argumento ,serial=... ao final de cada uma delas, conforme o exemplo abaixo:
```text
scsi1: /dev/disk/by-id/ata-WDC_WD20PURZ-85AKKY0_WD-WX22D51LJUYN,size=2000G,serial=WDPURPLE01
scsi2: /dev/disk/by-id/ata-WDC_WD20PURZ-85B4ZY0_WD-WXH2D43DKNTN,size=2000G,serial=WDPURPLE02
````
### Salve o arquivo (Ctrl+O, Enter, Ctrl+X) e realize um Shutdown/Power On completo na VM 100 pelo painel do Proxmox antes de prosseguir.

## 💾 Fase 4: Configuração do TrueNAS SCALE (ZFS Mirror & Rede)
1. Ajuste de Interfaces de Rede no TrueNAS
Certifique-se de adicionar duas placas de rede à VM 100 através da interface gráfica do Proxmox (uma atrelada à vmbr0 e outra à vmbr1). Dentro da interface web do TrueNAS, configure:
* Interface da vmbr0: Deixe em DHCP (ou defina o IP estático da sua rede local para acessá-lo, ex: 192.168.1.250).
* Interface da vmbr1: Defina como IP Estático Fixo: 10.0.0.250 com máscara /24 (255.255.255.0). Não configure Gateway nesta interface.
2. Criação do Pool de Armazenamento
 * No manu lateral esquerdo, acesse Storage e clique em Disks ao lado de Import Pool
 * Expanda da linha (Expand Row) clicando em cima do disco a ser usado, limpe o mesmo clicando em Wipe
 * Faça essa limpeza em todos os discos a serem usados.
 * No menu lateral esquerdo, acesse Storage e clique em Create Pool.
 * Dê um nome para o seu pool (ex: pool-dados).
 * Em Layout, selecione a opção Mirror (equivalente ao RAID-1).
 * Selecione os dois drives de 2TB (WDC_WD20PURZ) identificados pelos seriais WDPURPLE01 e WDPURPLE02 e mova-os para a seção do VDEV de dados (Data VDEVs).
 * Clique em Create e confirme a formatação.
 * Com a VM do TrueNAS inicializada e os dois discos de 2TB conectados com sucesso via hardware passthrough, siga os passos abaixo dentro do painel web do TrueNAS:
3. Criação do Dataset e Compartilhamento NFS Seguro
 * Clique nos três pontos ao lado do pool criado e escolha Add Dataset. Nomeie como docker-volumes.
 * Acesse o menu Shares na barra lateral, vá em Unix (NFS) Shares e clique em Add.
 * Aponte para o caminho do seu dataset (ex: /mnt/pool-dados/docker-volumes).
 * Segurança Avançada: Nas configurações do compartilhamento, em Authorized Networks, preencha com 10.0.0.0/24. Isso garante que apenas a rede interna virtual isolada consiga montar o disco, bloqueando qualquer invasão vinda da rede local de computadores comuns.
### 📦 Fase 5: Criação do Container LXC Base, Docker & Portainer
1. Criação do Container e Configuração de Rede Dupla
No canto superior direito do Proxmox, clique em Create CT e configure o container de orquestração:
 * Aba General: Desmarque a caixa Unprivileged container (o container deve ser Privilegiado para permitir montagens NFS).
 * Aba Network:
 * Placa eth0: Atrele à vmbr0 (Rede local) configurada em DHCP ou IP Fixo da sua LAN.
 * Placa eth1: Atrele à vmbr1 (Rede Virtual Isolada) com o IP Estático Fixo 10.0.0.100/24. Sem Gateway.
 * Aba Options (Após a criação): Vá em Features, clique em Edit e marque obrigatoriamente as caixas keyctl e nesting (necessário para o Docker rodar aninhado).
 * baixe um template, o ubuntu-24.04-standard é uma boa opção
2. Preparação do Sistema e Instalação do Docker (Dentro do LXC)
Inicie o Container LXC, abra o Console dele e execute:
```bash
apt update && apt install -y curl cifs-utils nfs-common
curl -fsSL [https://get.docker.com](https://get.docker.com) -o get-docker.sh && sh get-docker.sh
```
3. Montagem do Volume Persistente do TrueNAS via Rede Isolada
Crie o ponto de montagem interno e conecte ao NFS do TrueNAS via rede 10.0.0.x:
```bash
mkdir -p /mnt/storage-truenas
mount -t nfs 10.0.0.250:/mnt/pool-dados/docker-volumes /mnt/storage-truenas
```
(Para garantir que a montagem persista ao reiniciar o container, adicione a linha 10.0.0.250:/mnt/pool-dados/docker-volumes /mnt/storage-truenas nfs defaults 0 0 no arquivo /etc/fstab do container).

4. Inicialização do Portainer
Crie o diretório do Portainer e inicialize o serviço:
```bash
mkdir -p /opt/portainer && cd /opt/portainer
```
Crie o arquivo docker-compose.yml:
```bash
nano docker-compose.yml
```
Cole o conteúdo de deploy básico do Portainer:
```YAML
version: '3.8'
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /mnt/storage-truenas/portainer_data:/data
```
Suba o container do Portainer:
```bash
docker compose up -d
```
Pronto para subir o container LXC limpo do Docker + Portainer nessa estrutura.

2. Configuração de Passthrough de GPU AMD RX 580 para o LXC
Acesse o shell do Proxmox Host e abra o arquivo de configuração do container recém-criado (substitua 101 pelo ID real do seu container):
```bash
nano /etc/pve/lxc/101.conf
```
Adicione as seguintes linhas ao final do arquivo para liberar os nós de renderização de hardware de forma compartilhada:
```text
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
````

   
### 🐳 Fase 6: Configuração do Ambiente Docker & Portainer
1. Atualizar Repositórios e Dependências do Sistema (Dentro do LXC)
Inicie o Container LXC, abra o Console dele e prepare o ambiente:
```bash
apt update && apt install -y curl cifs-utils nfs-common
````
2. Instalação Automatizada do Docker Engine
Execute o script oficial dentro do terminal do container para instalar o Docker e o Docker Compose:
````bash
curl -fsSL [https://get.docker.com](https://get.docker.com) -o get-docker.sh && sh get-docker.sh
````
3. Montagem do Volume Persistente via Rede Isolada
Crie a pasta local dentro do container onde o storage do TrueNAS será mapeado:
````bash
mkdir -p /mnt/storage-truenas
````
Para realizar o mapeamento seguro usando o caminho IP da rede virtual interna (que nunca cai):
```bash
mount -t nfs 10.0.0.250:/mnt/pool-dados/docker-volumes /mnt/storage-truenas
```
(Para tornar essa montagem permanente mesmo após reiniciar o container, adicione a linha correspondente dentro do arquivo /etc/fstab do LXC).

4. Inicialização do Portainer
Crie o diretório do Portainer e inicialize a aplicação apontando seus volumes para a pasta mapeada /mnt/storage-truenas, garantindo a persistência resiliente aos desligamentos de rede física.
````bash
mkdir -p /opt/portainer && cd /opt/portainer
docker compose up -d
````

