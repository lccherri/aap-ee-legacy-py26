# Procedimento: build da EE única (AAP moderno + retrocompatibilidade Python 2.6)

## 1. Instalar o ansible-builder

    pip3 install ansible-builder --user
    ansible-builder --version   # >= 3.0

## 2. Organizar os arquivos

    legacy-awx-ee/
    ├── execution-environment.yml
    ├── requirements.txt
    ├── requirements.yml
    ├── bindep.txt
    └── awx_deps_report_docker.txt   # extração de origem, não entra no build

## 3. Build

    cd legacy-awx-ee
    ansible-builder build \
      --tag meu-registro.exemplo.com/aap-ee-legacy-py26:1.0 \
      --container-runtime docker \
      -v 3

  (`--container-runtime podman` funciona igual, se for o que você tem
  disponível. Este procedimento foi validado de ponta a ponta com
  `ansible-builder 3.1.1` + `podman` antes de ir para o cliente.)

## 4. Validar a imagem

    docker run --rm -it meu-registro.exemplo.com/aap-ee-legacy-py26:1.0 bash

    ansible --version
    # esperado: ansible [core 2.12.x]
    ansible-runner --version
    # esperado: retorna uma versão sem erro (é o processo que o AAP usa
    # pra invocar o playbook dentro da EE)

    python3 -c "import selinux, rpm, psycopg2, cryptography, gpg, jmespath; print('bindings OK')"

## 5. TESTE CRÍTICO — validar contra um host Python 2.6 real

Este é o teste que decide se o esforço valeu a pena. Rode um ad-hoc simples
contra o host antigo, de dentro do container recém-buildado:

    docker run --rm -it -v ~/.ssh:/root/.ssh:ro \
      meu-registro.exemplo.com/aap-ee-legacy-py26:1.0 \
      ansible remote.hostname -i remote.hostname, -m setup

Se retornar os facts do host sem o erro de versão mínima de Python, a EE
está funcionalmente equivalente ao ambiente antigo pra esse tipo de host.

## 5.1. Nota sobre a imagem base e licenciamento

A base é `registry.access.redhat.com/ubi9/ubi` (UBI - Universal Base Image),
não CentOS Stream nem uma imagem de `registry.redhat.io`. Isso é proposital:

- UBI tem EULA próprio que permite redistribuir livremente a imagem
  resultante, inclusive em registro público (Quay.io público, Docker Hub,
  etc.) - diferente de imagens de `registry.redhat.io` (ex:
  `ansible-automation-platform-2X/ee-minimal-rhelX`), cujo EULA de
  assinatura geralmente proíbe redistribuir pra quem não tem entitlement.
- Se pretende publicar essa EE num registro privado da organização e
  prefere a EE oficialmente suportada do AAP, `ee-minimal-rhelX` de
  `registry.redhat.io` é uma opção válida - só não publique o resultado
  num registro público.
- De qualquer forma: nenhuma imagem base muda o status de suporte do
  `ansible-core 2.12` em si, que já está EOL - a escolha de base só afeta
  a camada de SO por baixo.

## 6. Publicar e registrar no AAP

    docker push meu-registro.exemplo.com/aap-ee-legacy-py26:1.0

No AAP: Automation Execution Environments > Add
  Name: aap-ee-legacy-py26
  Image: meu-registro.exemplo.com/aap-ee-legacy-py26:1.0
  Organização: restrinja à organização/projeto que efetivamente usa
  hosts antigos, para não virar a EE padrão de tudo.

## 6.1. SSH pra hosts legados (RHEL5/6-era) - já vem embutido por padrão

Hosts realmente antigos (RHEL5/6-era) só falam algoritmos de SSH que o
OpenSSH/OpenSSL modernos desativaram por padrão - descoberto testando de
verdade contra um host desses. Sem isso, a conexão SSH falha antes mesmo
de chegar no Python do host, com erro do tipo:

    no matching key exchange method found. Their offer: diffie-hellman-group14-sha1,...

Tanto o lado do OpenSSL (provider "legacy" + `rh-allow-sha1-signatures`)
quanto o lado do SSH (`ssh_common_args`) já vêm embutidos na imagem, como
**padrão** via `/etc/ansible/ansible.cfg`:

```ini
[ssh_connection]
ssh_common_args = -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa
```

Decisão consciente: essa EE substitui um AWX inteiro dedicado só a hosts
legados (majoritariamente RHEL5), e a Red Hat não reboisa o OpenSSH dentro
do ciclo de vida de uma major release (só faz backport de CVE) - ou seja,
praticamente todo RHEL5 do parque oferece o mesmíssimo conjunto de
algoritmos. Um padrão único cobre a esmagadora maioria dos hosts sem
precisar configurar nada por host/grupo no inventário.

**Exceção pontual**: se algum host tiver `sshd_config` customizado
(cifras/algoritmos restritos manualmente) e precisar de um valor
diferente, a variável de inventário `ansible_ssh_common_args` no
host/grupo tem precedência sobre o `ansible.cfg` da imagem - configure só
nesse host específico, sem afetar o padrão dos demais.

**Atenção - substitui, não soma**: `ansible_ssh_common_args` (de qualquer
origem - inventário, extra vars, `--ssh-common-args`) **substitui por
inteiro** o valor do `ansible.cfg`, não mescla com ele. Se precisar
sobrescrever num host específico por outro motivo (ex: só pra pular
`known_hosts` num teste), inclua de novo os três `-o` de
Kex/HostKey/PubkeyAccepted nessa variável, senão o host perde a
compatibilidade de criptografia legada ao ganhar o outro ajuste. Pra
adicionar algo *sem* perder o padrão, use `ansible_ssh_extra_args` em vez
de `ansible_ssh_common_args` - esse é aditivo.

## 7. Escopo de uso recomendado

Associe essa EE apenas aos Job Templates/Projetos que tocam inventários
com hosts Python 2.6. Para toda a infraestrutura mais nova, use a EE
padrão/moderna do AAP (ansible-core atual, com patches de segurança em dia).
Isso limita a superfície de exposição do ansible-core 2.12 sem patch a
só o que realmente precisa dele.

## 8. Plano de saída (não deixe isso virar permanente)

Documente e monitore quantos hosts realmente dependem dessa EE. Cada host
migrado para Python 2.7+ (mesmo que via reinstalação de um Python paralelo,
como discutimos lá no início) é um host que pode sair da lista de exceção
e voltar a usar a EE moderna padrão do AAP.

---

## Troubleshooting

- **Conflito de dependência do pip durante o build**: normal se você
  reintroduzir algum pin exato do requirements.txt original (ex:
  `cryptography==2.3`). O ansible 5.x exige uma versão mínima mais nova.
  Deixe sem pin, como está no arquivo, e deixe o pip resolver.

- **`ansible --version` não mostra "core 2.12.x"**: confirme que não há
  outro `ansible-core` sendo puxado por engano via `requirements.yml`
  (ex: alguma collection listada com dependência de core mais nova
  forçando um upgrade). Rode `pip show ansible-core` dentro do container
  pra conferir.

- **selinux/rpm ausentes mesmo com pacote de sistema instalado**: mesma
  causa do build anterior — múltiplas versões de Python na imagem base
  desalinhadas. Confirme com `python3 --version` vs. o Python que o
  `dnf` usou pra instalar `python3-libselinux`.

- **`Additional properties are not allowed ('EE_BASE_IMAGE' was unexpected)`**:
  `execution-environment.yml` está com `version` desalinhada da sintaxe
  usada. A partir do schema v2, a imagem base vai em `images.base_image.name`,
  não em `build_arg_defaults.EE_BASE_IMAGE` (isso só existe no schema v1).
  As chaves granulares `additional_build_steps.prepend_base`/`append_final`
  só existem a partir do schema v3 - por isso este arquivo usa `version: 3`
  com `images.base_image.name`.

- **`No match for argument: gpgme-devel swig`** (ou qualquer outro pacote
  do bindep.txt "sumindo"): alguns pacotes -devel vivem no repositório
  CodeReady Builder. Na UBI9 esse repo (`ubi-9-codeready-builder-rpms`)
  já vem habilitado por padrão (confirme com `dnf repolist --all`); no
  CentOS Stream 9 o equivalente se chama só `crb` e vem desabilitado
  (precisa de `dnf config-manager --set-enabled crb`). Se trocar de
  imagem base de novo, confirme o nome do repo e se já vem habilitado.

- **`All matches were filtered out by modular filtering for argument: swig`**:
  na UBI9/RHEL9, `swig` fica atrás de um módulo dnf (`swig:4.1`), diferente
  do CentOS Stream 9 onde é pacote normal. Precisa de
  `dnf module enable -y swig:4.1` antes de instalar (já configurado no
  `prepend_base` do execution-environment.yml atual).

- **`epel-release` não encontrado via `dnf install -y epel-release`
  na UBI9**: diferente do CentOS Stream, a UBI não empacota um ponteiro
  pro EPEL em nenhum repo próprio. Instale direto da URL do projeto
  Fedora: `dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm`
  (funciona em qualquer sistema ABI-compatível com RHEL9).

- **`ansible-galaxy: command not found` no estágio "galaxy"**: no schema
  v3, a simples presença da chave `dependencies.galaxy` no
  execution-environment.yml já faz o ansible-builder gerar um estágio que
  roda `ansible-galaxy` ANTES de instalar o pacote `ansible` via pip - e
  nesse ponto ainda não existe Ansible na imagem. Como não usamos nenhuma
  collection fora do que o `ansible==5.*` já traz, essa chave foi
  deliberadamente removida do execution-environment.yml. Só volte a
  declará-la se popular `requirements.yml` com uma collection de verdade.

- **`ansible: command not found` / `ansible-runner: command not found`
  no estágio "final"**: no schema v3, o `check_ansible` roda no início do
  estágio final, antes de qualquer coisa do `requirements.txt` ser
  instalada nesse estágio - então listar `ansible` só no requirements.txt
  não basta. É preciso usar `dependencies.ansible_core.package_pip` (e
  `dependencies.ansible_runner.package_pip`) no execution-environment.yml,
  que instalam essas duas coisas já no estágio base. Ambas já estão
  configuradas no arquivo atual.