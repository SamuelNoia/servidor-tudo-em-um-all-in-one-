# 🏢 Manual de Implementação: Servidor All-in-One para Pequenas Empresas

Este repositório contém o guia passo a passo para a montagem, configuração e deploy de um servidor corporativo baseado no hipervisor **Proxmox VE**, utilizando armazenamento virtualizado e containers integrados.

## 📌 Arquitetura do Projeto

O ecossistema é centralizado em um único servidor físico executando o **Proxmox VE 9.2**. A topologia de dados e serviços divide-se em:
1. **Armazenamento de Arquivos:** Uma VM dedicada rodando **TrueNAS SCALE** recebe, via mapeamento direto por ID físico (Passthrough de bloco), dois HDDs mecânicos para gerenciar um array ZFS espelhado.
2. **Serviços e Containers:** O próprio Proxmox hospeda os containers (LXC). A placa de vídeo física (GPU) é compartilhada diretamente com os containers do host para aceleração de hardware e transcodificação de mídia.
3. **Orquestração de Aplicações:** Dentro do container principal, o **Docker** e o **Portainer** gerenciam os microsserviços da empresa, utilizando **volumes persistentes amarrados ao RAID de dados do TrueNAS**.

---

## 🛠️ Especificações de Armazenamento e Hardware

*   **SO Host (Proxmox VE):** 1x SSD SATA de 120GB destinado ao Proxmox VE.
*   **Armazenamento de Alta Velocidade (`nvme250`):** 1x SSD NVMe destinado a armazenar imagens ISO, templates e os discos virtuais rápidos.
*   **Armazenamento de Dados (Mídia/Arquivos):** 2x HDDs de 2TB Western Digital (WD Purple - foco em CFTV/Alta durabilidade), mapeados diretamente para a VM de armazenamento.
*   **Gráficos:** 1x GPU AMD Radeon RX 580 256-bits (Destinada ao pass-through de LXC via Host Kernel).

---

## 🚀 Cronograma de Implementação

- [x] **Fase 1:** Instalação do Proxmox VE 9.2 no SSD SATA & Ajuste de Rede para DHCP.
- [x] **Fase 2:** Criação da VM do TrueNAS SCALE (VM 100) utilizando o Storage NVMe para Boot (32GB).
- [x] **Fase 3:** Passthrough físico por ID dos 2x HDDs WD Purple de 2TB para a VM 100.
- [x] **Fase 4:** Configuração do Armazenamento (ZFS Mirror) no TrueNAS SCALE.
- [x] **Fase 5:** Configuração do Driver AMD no Host Proxmox e Passthrough de GPU para Containers LXC.
- [ ] **Fase 6:** Instalação do Docker + Portainer com Persistência no Storage RAID do TrueNAS.

---

## 📖 Passo a Passo de Infraestrutura

### Transição de IP Estático para IP Dinâmico (DHCP) no Proxmox
Para alterar a interface de rede do Proxmox para obter IP via DHCP, edite o arquivo de configuração de rede:

```bash
nano /etc/network/interfaces
````
## Deixe o bloco da ponte principal (vmbr0) com a seguinte estrutura:
```text
auto vmbr0
iface vmbr0 inet dhcp
        bridge-ports enp3s0
        bridge-stp off
        bridge-fd 0
```
## Para aplicar as configurações de rede sem reiniciar o host:
```bash
ifreload -a
```
## Mapeamento Físico dos HDDs WD Purple (Passthrough por ID)
Execute os comandos no shell do Proxmox para atrelar os discos diretamente à VM 100 do TrueNAS SCALE:
```bash
qm set 100 -scsi1 /dev/disk/by-id/ata-WDC_WD20PURZ-85AKKY0_WD-WX22D51LJUYN
qm set 100 -scsi2 /dev/disk/by-id/ata-WDC_WD20PURZ-85B4ZY0_WD-WXH2D43DKNTN
```
## Configuração de Passthrough de GPU AMD RX 580 para LXC
Para liberar a aceleração de hardware aos containers LXC, adicione as seguintes linhas ao final do arquivo de configuração do seu container no Proxmox (ex: /etc/pve/lxc/101.conf):
```text
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```
### 🐳 Configuração do Ambiente Docker & Portainer
1. Atualizar Repositórios e Dependências do Sistema (Dentro do LXC)
Execute o comando abaixo no terminal do seu container para preparar as ferramentas necessárias:
```bash
apt update && apt install -y curl cifs-utils nfs-common
````
### 2. Instalação Automatizada do Docker Engine
Execute o script oficial para instalar o Docker e o Docker Compose dentro do container LXC:
````bash
curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh
````
### 3. Criação da Pasta da Aplicação
Crie o diretório onde ficará o arquivo de deploy do Portainer:
````bash
mkdir -p /opt/portainer && cd /opt/portainer
````
### 4. Inicialização do Portainer
Após criar o arquivo docker-compose.yml na pasta (conforme modelo no repositório), suba o container em segundo plano com o seguinte comando:
````bash
docker compose up -d
````

