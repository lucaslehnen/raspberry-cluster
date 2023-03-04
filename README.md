# Cluster Kubernetes na Raspberry

![Ansible](https://img.shields.io/badge/Ansible-%3E%3D6.1.0-red?logo=ansible&logoColor=white)

Esse repositório tem a automatização do meu cluster de Raspberrys, com todo o processo necessário para reinstala-lo do zero.

**Objetivo**

Disponibilizar um cluster de Kubernetes para colocar algumas bases de dados e rodar serviços em tutoriais e testes, de forma temporária.
Ainda posso utilizar o cluster para deixar alguns serviços em pé.

**LEMBRETE PARA MIM MESMO**

Siga o **KISS:** Keep It Simple Stupid!

## Preparando o Ubuntu das Raspberrys

Os passos abaixo foram utilizados no Ubuntu 22.04, na imagem específica para raspberry de 64 bits no site Oficial.

Baixei a imagem e usei o Raspeberry Pi Imager para gravar a imagem baixada no MicroSD.

<details>
<summary>Configurações após a gravação</summary>

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

</details>

<details>
<summary>Configurações de SSH nas Raspberrys</summary>

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

Acessar a Raspberry para trocar a senha do usuário padrão `ubuntu`.

```shell
ssh ubuntu@192.168.0.21
```

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