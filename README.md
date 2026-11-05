# aap-ee-legacy-py26

Execution Environment (EE) para o **Ansible Automation Platform (AAP)** moderno
com suporte a managed nodes que ainda executam **Python 2.6** (ex.: RHEL 5/6).

O objetivo é substituir uma instância AWX legada dedicada a esses hosts por uma
única EE registrada no AAP, sem necessidade de manter infraestrutura paralela.

> [!WARNING]
> Esta EE usa `ansible-core 2.12` (pacote `ansible` 5.x), a última versão com
> suporte a Python 2.6 nos managed nodes. Essa versão está **EOL desde 2022** e
> não recebe correções de segurança. O uso deve ficar restrito aos hosts que
> realmente dependem dela. Ver [Escopo de uso](#escopo-de-uso).

## Sumário

- [Características](#características)
- [Pré-requisitos](#pré-requisitos)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Build](#build)
- [Validação](#validação)
- [Publicação no AAP](#publicação-no-aap)
- [Compatibilidade SSH com hosts legados](#compatibilidade-ssh-com-hosts-legados)
- [Imagem base e licenciamento](#imagem-base-e-licenciamento)
- [Escopo de uso](#escopo-de-uso)
- [Plano de saída](#plano-de-saída)
- [Troubleshooting](#troubleshooting)

## Características

| Componente        | Versão / detalhe                                              |
| ----------------- | ------------------------------------------------------------- |
| Imagem base       | `registry.access.redhat.com/ubi9/ubi` (Python 3.9)            |
| Ansible           | `ansible>=5.0.0,<6.0.0` → `ansible-core 2.12`                 |
| ansible-runner    | `2.4.2` (versões ≥ 2.4.3 são incompatíveis com core 2.12)     |
| Collections       | Conjunto community embutido no pacote `ansible` 5.x           |
| SSH legado        | KEX/host key SHA-1 e `ssh-rsa` habilitados por padrão         |
| OpenSSL           | Provider `legacy` + `rh-allow-sha1-signatures` habilitados    |
| Bindings Python   | `selinux`, `rpm`, `gpg`, `psycopg2`, `cryptography`, `jmespath` |

## Pré-requisitos

- [`ansible-builder`](https://ansible.readthedocs.io/projects/builder/) ≥ 3.0
  (validado com 3.1.1)
- `podman` ou `docker`
- Acesso a um registro de imagens acessível pelo AAP

```bash
pip3 install --user ansible-builder
ansible-builder --version
```

## Estrutura do repositório

```text
legacy-awx-ee/
├── execution-environment.yml   # Definição da EE (schema v3)
├── requirements.txt            # Dependências Python (pip)
├── requirements.yml            # Collections Ansible (vazio por padrão)
└── bindep.txt                  # Dependências de sistema (dnf)
```

Os arquivos de configuração contêm comentários detalhados sobre cada decisão
de versão e cada dependência.

## Build

```bash
ansible-builder build \
  --tag registry.example.com/aap-ee-legacy-py26:1.0 \
  --container-runtime podman \
  -v 3
```

Para usar Docker, basta trocar para `--container-runtime docker`. O diretório
`context/` é gerado pelo build e não é versionado.

O próprio build executa verificações ao final (versão do Ansible, bindings
`selinux`/`rpm`, provider `legacy` do OpenSSL e `ansible.cfg` padrão); qualquer
falha interrompe a construção da imagem.

## Validação

### Verificação da imagem

```bash
podman run --rm -it registry.example.com/aap-ee-legacy-py26:1.0 bash
```

Dentro do container:

```bash
ansible --version          # esperado: ansible [core 2.12.x]
ansible-runner --version   # deve retornar a versão sem erro
python3 -c "import selinux, rpm, psycopg2, cryptography, gpg, jmespath; print('bindings OK')"
```

O `ansible-runner` é o processo usado pelo AAP para executar playbooks dentro
da EE, portanto sua presença é obrigatória.

### Teste contra um host Python 2.6

Este é o teste decisivo: executar um comando ad-hoc contra um host legado real
a partir do container.

```bash
podman run --rm -it -v ~/.ssh:/root/.ssh:ro \
  registry.example.com/aap-ee-legacy-py26:1.0 \
  ansible legacy-host.example.com -i legacy-host.example.com, -m setup
```

O retorno dos facts do host, sem erro de versão mínima de Python, indica que a
EE é funcionalmente equivalente ao ambiente legado para esse tipo de host.

## Publicação no AAP

```bash
podman push registry.example.com/aap-ee-legacy-py26:1.0
```

No AAP, em **Automation Execution → Infrastructure → Execution Environments → Add**:

| Campo        | Valor                                                    |
| ------------ | -------------------------------------------------------- |
| Name         | `aap-ee-legacy-py26`                                     |
| Image        | `registry.example.com/aap-ee-legacy-py26:1.0`            |
| Organization | Apenas a organização que gerencia hosts legados          |

Restringir a organização evita que esta EE seja usada como padrão para outros
inventários.

## Compatibilidade SSH com hosts legados

Hosts da era RHEL 5/6 oferecem apenas algoritmos SSH que o OpenSSH e o
OpenSSL 3.x modernos desabilitam por padrão. Sem ajustes, a conexão falha antes
de alcançar o Python do host:

```text
no matching key exchange method found. Their offer: diffie-hellman-group14-sha1,...
```

A imagem já inclui os dois ajustes necessários:

1. **OpenSSL**: `/etc/pki/tls/legacy-ssh-openssl.cnf` (via `OPENSSL_CONF`)
   habilita o provider `legacy` e assinaturas SHA-1.
2. **SSH**: `/etc/ansible/ansible.cfg` define o padrão:

   ```ini
   [ssh_connection]
   ssh_common_args = -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa
   ```

Esses ajustes apenas **adicionam** algoritmos aceitos; conexões com hosts
modernos continuam funcionando normalmente.

A configuração é global porque a Red Hat não rebaseia o OpenSSH dentro do ciclo
de vida de uma major release (apenas aplica backports de CVE). Assim, hosts da
mesma versão de RHEL tendem a oferecer o mesmo conjunto de algoritmos, e um
único padrão cobre a grande maioria dos casos sem configuração por host.

### Sobrescrita por host ou grupo

Hosts com `sshd_config` customizado podem exigir valores diferentes. A variável
de inventário `ansible_ssh_common_args` tem precedência sobre o `ansible.cfg` da
imagem.

> [!IMPORTANT]
> `ansible_ssh_common_args` (de qualquer origem: inventário, extra vars ou
> `--ssh-common-args`) **substitui integralmente** o valor do `ansible.cfg`,
> sem mesclagem. Ao sobrescrevê-la, é necessário repetir as três opções
> `-o KexAlgorithms/HostKeyAlgorithms/PubkeyAcceptedAlgorithms`.
>
> Para **acrescentar** opções sem perder o padrão, usar
> `ansible_ssh_extra_args`, que é aditiva.

## Imagem base e licenciamento

A imagem base é a **UBI 9** (`registry.access.redhat.com/ubi9/ubi`), e não
CentOS Stream ou imagens de `registry.redhat.io`:

- O EULA da UBI permite redistribuir livremente a imagem resultante, inclusive
  em registros públicos (Quay.io, Docker Hub etc.).
- Imagens de `registry.redhat.io` (ex.: `ee-minimal-rhel9`) são uma alternativa
  válida com suporte oficial do AAP, mas o EULA de assinatura geralmente proíbe
  a redistribuição. Nesse caso, a imagem deve ser publicada apenas em registros
  privados.
- A escolha da imagem base afeta somente a camada de sistema operacional; o
  status EOL do `ansible-core 2.12` permanece o mesmo.

## Escopo de uso

Associar esta EE apenas aos Job Templates e Projetos que operam sobre
inventários com hosts Python 2.6. Para o restante da infraestrutura, usar a EE
padrão do AAP, com `ansible-core` atual e correções de segurança em dia. Isso
limita a superfície de exposição de uma versão sem suporte ao mínimo
necessário.

## Plano de saída

Esta EE é uma solução de transição. Recomenda-se documentar e monitorar a
quantidade de hosts que dependem dela. Cada host migrado para Python 2.7+
(inclusive por meio da instalação de um interpretador paralelo) pode sair da
lista de exceção e voltar a usar a EE padrão do AAP.

## Troubleshooting

<details>
<summary><strong>Conflito de dependências do pip durante o build</strong></summary>

Ocorre ao reintroduzir pins exatos antigos (ex.: `cryptography==2.3`). O
`ansible` 5.x exige versões mínimas mais recentes. Manter as dependências sem
pin, como em `requirements.txt`, e deixar o pip resolver.

</details>

<details>
<summary><strong><code>ansible --version</code> não mostra <code>core 2.12.x</code></strong></summary>

Verificar se alguma collection em `requirements.yml` está forçando a instalação
de um `ansible-core` mais novo. Conferir com `pip show ansible-core` dentro do
container.

</details>

<details>
<summary><strong>Bindings <code>selinux</code>/<code>rpm</code> ausentes mesmo com o pacote de sistema instalado</strong></summary>

Indica múltiplas versões de Python na imagem base. Comparar `python3 --version`
com o interpretador usado pelo `dnf` para instalar `python3-libselinux`.

</details>

<details>
<summary><strong><code>Additional properties are not allowed ('EE_BASE_IMAGE' was unexpected)</code></strong></summary>

O campo `version` do `execution-environment.yml` não corresponde à sintaxe
usada. A partir do schema v2, a imagem base é definida em
`images.base_image.name` (e não em `build_arg_defaults.EE_BASE_IMAGE`, exclusivo
do v1). As chaves `additional_build_steps.prepend_base`/`append_final` existem
apenas a partir do schema v3, por isso o arquivo usa `version: 3`.

</details>

<details>
<summary><strong><code>No match for argument: gpgme-devel swig</code></strong></summary>

Alguns pacotes `-devel` ficam no repositório CodeReady Builder. Na UBI 9, o
repositório `ubi-9-codeready-builder-rpms` já vem habilitado (verificar com
`dnf repolist --all`). No CentOS Stream 9, o equivalente é `crb` e precisa ser
habilitado com `dnf config-manager --set-enabled crb`. Ao trocar a imagem base,
confirmar o nome do repositório e seu estado.

</details>

<details>
<summary><strong><code>All matches were filtered out by modular filtering for argument: swig</code></strong></summary>

Na UBI 9/RHEL 9, o `swig` é distribuído como módulo dnf (`swig:4.1`) e requer
`dnf module enable -y swig:4.1` antes da instalação. Esse passo já está
configurado em `prepend_base`.

</details>

<details>
<summary><strong><code>epel-release</code> não encontrado na UBI 9</strong></summary>

A UBI não empacota o `epel-release` em seus repositórios. A instalação é feita
diretamente pela URL do projeto Fedora:

```bash
dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```

O EPEL é necessário apenas para o `libsodium-devel` (PyNaCl/bcrypt).

</details>

<details>
<summary><strong><code>ansible-galaxy: command not found</code> no estágio "galaxy"</strong></summary>

No schema v3, a simples presença de `dependencies.galaxy` faz o
`ansible-builder` gerar um estágio que executa `ansible-galaxy` antes da
instalação do Ansible. Como nenhuma collection além das incluídas no
`ansible` 5.x é necessária, essa chave foi removida intencionalmente. Ela só
deve ser reintroduzida se `requirements.yml` passar a listar collections.

</details>

<details>
<summary><strong><code>ansible</code>/<code>ansible-runner: command not found</code> no estágio "final"</strong></summary>

No schema v3, o `check_ansible` roda no início do estágio final, antes da
instalação do `requirements.txt`. Por isso o Ansible e o ansible-runner são
instalados via `dependencies.ansible_core.package_pip` e
`dependencies.ansible_runner.package_pip`, que atuam já no estágio base. Ambos
estão configurados no `execution-environment.yml`.

</details>

<details>
<summary><strong><code>TypeError: unsupported operand type(s) for |: 'NoneType' and 'AnsibleMapping'</code> durante jobs no AAP</strong></summary>

A partir do `ansible-runner` 2.4.3, o callback `awx_display` (usado para
transmitir a saída dos jobs ao AAP) é incompatível com o `ansible-core 2.12`.
O erro não aparece ao executar `ansible` diretamente pela CLI. Por isso o
`ansible-runner` está fixado em `2.4.2` (versões 2.2.0 a 2.4.2 foram testadas
com sucesso).

</details>
