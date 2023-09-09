# Cluster Kubernetes na Raspberry

![Ansible](https://img.shields.io/badge/Ansible-%3E%3D6.1.0-red?logo=ansible&logoColor=white)

Esse repositório tem a automatização do meu cluster de Raspberrys, com todo o processo necessário para reinstala-lo do zero.

**Objetivo**

Disponibilizar um cluster de Kubernetes para colocar algumas bases de dados e rodar serviços em tutoriais e testes, de forma temporária.
Ainda posso utilizar o cluster para deixar alguns serviços em pé.

**LEMBRETE PARA MIM MESMO**

Siga o **KISS:** Keep It Simple Stupid!

## Preparando o sistema operacional das Raspberrys

Playbook testado com:
 - Ubuntu 22.04 LTS 64 bits
 - Raspberry OS 64 bits

Usei o Raspeberry Pi Imager para gravar a imagem baixada no MicroSD.
Quando usei o Raspberry Pi OS, configurei opções avançadas para já definir usuário (`pi`) e senha, bem como o host (`raspberrypi.local`).

<details>
<summary>Configurações após a gravação para o Ubuntu 22.04 LTS</summary>

Como a imagem vem com o Cloud-init, deixei o IP configurado, de modo a facilitar o acesso, editando o arquivo `network-config`.
Coloquei o seguinte conteúdo:

```yaml
version: 2
ethernets:
  eth0:
    dhcp4: false
    addresses:
      - 192.168.0.21/24
    gateway4: 192.168.0.1
    nameservers:
      search: [home.tchecode.com]
      addresses: [192.168.0.1]
```

Após isso, é necessário alterar a senha do usuário padrão `ubuntu`, conectando via SSH.

</details>

<details>
<summary>Configurações após a gravação para o Raspberry Pi OS</summary>

Diferente do Ubuntu, não encontrei uma forma de definir o IP estático antes, portanto primeiro preciso localizar o IP que o DHCP irá atribuir assim que ligar: 

```shell
PING raspberrypi.local (192.168.0.114) 56(84) bytes of data.
64 bytes from 192.168.0.114 (192.168.0.114): icmp_seq=1 ttl=63 time=1.33 ms
64 bytes from 192.168.0.114 (192.168.0.114): icmp_seq=2 ttl=63 time=1.00 ms
```

Aqui no ambiente funcionou o ping devido o suporte do roteador e ao fato de eu ter configurado o hostname na instalação do Raspberry OS.
Mas há outras alternativas para localizar o IP, como o nmap:

```shell
nmap -sn 192.168.0.0/24
```	
Ele roda um comando ping em todos os IPs da rede e retorna os que responderam, permitindo identificarmos o host que queremos.

Com o IP em mãos, podemos acessar a máquina via SSH e alterar o ip para estático, editando o arquivo `/etc/dhcpcd.conf`:

```ini
interface eth0
static ip_address=192.168.0.21/24
static routers=192.168.0.1
static domain_name_servers=192.168.0.1
```

Reiniciar a raspberry para garantir. Após isso, podemos acessar via SSH novamente agora já com o IP estático.

</details>

<details>
<summary>Configurações de SSH nas Raspberrys </summary>

### Configurações de SSH nas Raspberrys

Ao criar um par de chaves SSH e registrar a pública nos servidores, podemos conectar a partir da nossa máquina (ou de onde for necessário) sem solicitar senhas.

Os passos para configurá-la estão abaixo:

**Criando as chaves**

Para gerar um novo par de chaves:

```shell
ssh-keygen -C "raspberrys" -f ~/.ssh/raspberrys -t rsa -b 4096 -q -N ""
```

A seguir, vamos configurar para que o ssh encontre estas identidades facilmente, editando o arquivo ~/.ssh/config:

```
IdentityFile ~/.ssh/id_rsa
IdentityFile ~/.ssh/raspberrys
```

E em seguida copiar a chave pública para o servidor:

```shell
ssh-copy-id -i ~/.ssh/raspberrys.pub ubuntu@192.168.0.21
```

</details>

## Overview

Visão geral dos recursos:

- 3 Raspberrys
  - k3s-01: 4GB RAM, 32 GB MicroSD
    - K3s Server
  - k3s-02 e k3s-03: 2GB RAM,32 GB MicroSD
    - K3s Agents
- Ubuntu 22.04

## Subindo o ambiente

- Conferir se os IPS configurados nas raspberrys estão corretos no arquivo de inventário: `hosts.yml`;
- Conferir as variáveis em `variables.yml`:

  ```yaml
  local_network:
  cidr: 192.168.0.0/24    # CIDR da rede local
  k3s:
    version: v1.24.2+k3s1 # Versão do K3S
    server:
      ip: 192.168.0.21    # Endereço da API do Kubernetes
      args: ""            # Argumentos extra do servidor https://rancher.com/docs/k3s/latest/en/installation/install-options/server-config/
    agent:
      args: ""            # Argumentos extra do agente https://rancher.com/docs/k3s/latest/en/installation/install-options/agent-config/
  ```
- Configurar o ambiente virtual e instalar o Ansible:

** Talvez seja necessário instalar o venv caso esteja usando o Debian/Ubuntu diretamente ao invés do WSL:

  ```shell
  apt install python3.10-venv
  ```

  Criar e ativar o ambiente virtual:
  ```shell
  cd setup
  python3 -m venv .venv
  source .venv/bin/activate
  pip install --upgrade pip
  ```

  Instalar o Ansible:
  ```shell
  pip install -r requirements.txt
  ```

- Para rodar o playbook Ansible:

  ```bash
  cd setup
  ansible-playbook -i hosts.yml main.yml --extra-vars "@./variables.yml"
  ```


- Com isso o ambiente já deve ter sido configurado;