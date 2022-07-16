# Cluster Kubernetes na Raspberry

![Ansible](https://img.shields.io/badge/Ansible-%3E%3D2.13.1-red?logo=ansible&logoColor=white)

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

<details>
<summary>Configurando dispositivos USB</summary>

### Configurando dispositivos USB

São dois: um HD externo de 1 TB e um SSD de 128 GB.
Rodei os comandos abaixo para prepará-los, limpando as partições e criando uma nova. 

Para isso, usei o fdisk mesmo:
```shell
fdisk /dev/sda
```

Guia rápido:
`p` Mostra a tabela de partições atual;
`d` Para deletar a partição 
`g` Modifica a tabela de partições para GPT
`n` Para criar nova partição
`w` Para gravar as alterações

Alterei o formato da tabela de partições para GPT.
Criei as três partições como `ext4`. No HDD, ao definir o último setor, usei `+465G` para aproximar as metades.

Para formatar, fiz o processo fora das raspberrys, pois notei que o HD exigia mais poder da USB do que a raspberry podia aguentar.

Formatei elas com o ext4:
```shell
sudo mkfs -t ext4 /dev/{{dev}}
```

</details>

## Overview

Visão geral dos recursos: 

- 3 Raspberrys
  - k3s-01: 4GB RAM, 32 GB MicroSD
    - HDD 1 TB via USB
    - SSD 128 via USB
    - NFS Server
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

- Para rodar o playbook Ansible:

  ```bash
  cd setup
  ansible-playbook -i hosts.yml main.yml --extra-vars "@./variables.yml"
  ```

- Com isso o ambiente já deve ter sido configurado;

## Configurando métricas e IDE

Em um primeiro momento, eu já coloco a stack de monitoramento via Lens mesmo.Posteriormente, se trouxer alguma vantagem, vou atrás de um Helm chart para isso.

Primeiramente, adicionar o cluster na IDE, colocando o conteúdo de `~/.kube/config` no Lens. 

Após, habilito as stacks de monitoramento do Prometheus e do Node Exporter em `Settings > Lens Metrics`. O kube-state-metrics o K3s já tem.