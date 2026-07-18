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

### Fase 1: Configuração Base de Rede no Proxmox Host
Para alterar a interface de rede do Proxmox para obter IP dinâmico via DHCP, acesse o shell do host e edite o arquivo de configuração de rede:

```bash
nano /etc/network/interfaces
````
### Deixe o bloco da ponte principal (vmbr0) com a seguinte estrutura:
```text
auto vmbr0
iface vmbr0 inet dhcp
        bridge-ports enp3s0
        bridge-stp off
        bridge-fd 0
```
### Para aplicar as configurações de rede sem reiniciar o host:
```bash
ifreload -a
```
Fase 2 e 3: Vinculação Física dos HDDs WD Purple (Passthrough por ID)
Execute os comandos abaixo no shell do Proxmox Host (root@pvs) para atrelar os discos rígidos diretamente à VM 100 do TrueNAS SCALE:
```bash
qm set 100 -scsi1 /dev/disk/by-id/ata-WDC_WD20PURZ-85AKKY0_WD-WX22D51LJUYN
qm set 100 -scsi2 /dev/disk/by-id/ata-WDC_WD20PURZ-85B4ZY0_WD-WXH2D43DKNTN
````
### ⚠️ Correção Crítica: Emulação de Número de Série (Evitar erro de topologia no TrueNAS)
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

## 💾 Fase 4: Configuração do TrueNAS SCALE (ZFS Mirror / RAID-1)
Com a VM do TrueNAS inicializada e os dois discos de 2TB conectados com sucesso via hardware passthrough, siga os passos abaixo dentro do painel web do TrueNAS:
1. Criação do Pool de Armazenamento
* No manu lateral esquerdo, acesse Storage e clique em Disks ao lado de Import Pool
* Expanda da linha (Expand Row) clicando em cima do disco a ser usado, limpe o mesmo clicando em Wipe
* Faça essa limpeza em todos os discos a serem usados.
* No menu lateral esquerdo, acesse Storage e clique em Create Pool.
* Dê um nome para o seu pool (ex: pool-dados).
* Em Layout, selecione a opção Mirror (equivalente ao RAID-1, garantindo redundância imediata).
* Na lista de discos disponíveis, selecione os dois drives de 2TB (WDC_WD20PURZ) e mova-os para a seção do VDEV de dados (Data VDEVs).
* No restante das opções apenas avance clicando em Next e por fim em Create Pool.

Clique em Create e confirme a formatação dos discos.
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

