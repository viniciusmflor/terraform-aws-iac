# Terraform AWS IaC — Tutorial HashiCorp Get Started

Repositório criado como parte da atividade de **Infraestrutura como Código (IaC)** utilizando o [tutorial oficial da HashiCorp](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-create). O objetivo é provisionar uma instância EC2 na AWS de forma declarativa, versionada e reproduzível.

---

## Sumário

1. [Pré-requisitos](#1-pré-requisitos)
2. [Instalação do Terraform e AWS CLI](#2-instalação-do-terraform-e-aws-cli)
3. [Configuração das Credenciais AWS](#3-configuração-das-credenciais-aws)
4. [Estrutura dos Arquivos Terraform](#4-estrutura-dos-arquivos-terraform)
5. [Inicialização — `terraform init`](#5-inicialização--terraform-init)
6. [Formatação e Validação](#6-formatação-e-validação)
7. [Planejamento — `terraform plan`](#7-planejamento--terraform-plan)
8. [Aplicação — `terraform apply`](#8-aplicação--terraform-apply)
9. [Inspeção do Estado](#9-inspeção-do-estado)
10. [Recursos Provisionados na AWS](#10-recursos-provisionados-na-aws)
11. [Destruição da Infraestrutura — `terraform destroy`](#11-destruição-da-infraestrutura--terraform-destroy)
12. [Referências](#12-referências)

---

## 1. Pré-requisitos

| Ferramenta | Versão mínima | Finalidade |
|---|---|---|
| Terraform CLI | 1.2.0+ | Gerenciar infraestrutura como código |
| AWS CLI | 2.x | Autenticar e interagir com a AWS |
| Conta AWS | — | Provisionar recursos na nuvem |
| IAM User | Permissão `AmazonEC2FullAccess` | Acesso programático seguro |

> **Importante:** Nunca utilize as credenciais da conta root da AWS para acesso programático. Crie sempre um IAM User dedicado.

---

## 2. Instalação do Terraform e AWS CLI

O Terraform foi removido do repositório padrão do Homebrew. É necessário adicionar o tap oficial da HashiCorp:

```bash
# Adicionar o tap oficial da HashiCorp
brew tap hashicorp/tap

# Instalar Terraform e AWS CLI
brew install hashicorp/tap/terraform awscli
```

**Verificação das instalações:**

```
$ terraform --version
Terraform v1.15.5
on darwin_arm64

$ aws --version
aws-cli/2.34.64 Python/3.14.5 Darwin/25.3.0 source/arm64
```

> Em casos de erro com Homebrew (Command Line Tools desatualizadas), é possível baixar o binário diretamente em [releases.hashicorp.com/terraform](https://releases.hashicorp.com/terraform/).

---

## 3. Configuração das Credenciais AWS

### 3.1 Criar IAM User no Console AWS

1. Acesse **IAM → Users → Create user**
2. Nome: `terraform-user`
3. Permissão: `AmazonEC2FullAccess` (Attach policies directly)
4. Em **Security credentials → Access keys**, gere uma chave para **CLI**
5. Anote o **Access Key ID** e o **Secret Access Key**

### 3.2 Configurar localmente

```bash
aws configure
```

```
AWS Access Key ID [None]: ****************
AWS Secret Access Key [None]: ****************
Default region name [None]: sa-east-1
Default output format [None]: json
```

### 3.3 Verificar autenticação

```bash
aws sts get-caller-identity
```

```json
{
    "UserId": "AIDAXAQBISFG...",
    "Account": "482112737613",
    "Arn": "arn:aws:iam::482112737613:user/terraform-user"
}
```

---

## 4. Estrutura dos Arquivos Terraform

```
terraform-aws-iac/
├── terraform.tf        # Bloco terraform: versão mínima e provider necessário
├── main.tf             # Provider AWS, data source AMI e resource EC2
├── .gitignore          # Exclui .terraform/, tfstate e arquivos sensíveis
└── README.md           # Documentação do projeto
```

### `terraform.tf`

Define os requisitos de versão do Terraform e do provider AWS.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.92"
    }
  }
  required_version = ">= 1.2"
}
```

### `main.tf`

Define o provider, busca dinamicamente a AMI Ubuntu mais recente e cria a instância EC2.

```hcl
provider "aws" {
  region = "sa-east-1"
}

data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }

  owners = ["099720109477"] # Canonical
}

resource "aws_instance" "app_server" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  tags = {
    Name = "learn-terraform"
  }
}
```

> **Por que `t3.micro`?** Na região `sa-east-1` (São Paulo) o tipo Free Tier elegível é `t3.micro` — diferente do `t2.micro` usado nas demais regiões.

---

## 5. Inicialização — `terraform init`

O `terraform init` baixa o provider AWS e prepara o diretório de trabalho.

```bash
terraform init
```

```
Initializing provider plugins found in the configuration...
- Finding hashicorp/aws versions matching "~> 5.92"...
- Installing hashicorp/aws v5.100.0...
- Installed hashicorp/aws v5.100.0 (signed by HashiCorp)

Initializing the backend...

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.
```

---

## 6. Formatação e Validação

### `terraform fmt`

Formata os arquivos `.tf` seguindo o estilo padrão do Terraform.

```bash
terraform fmt
```

Sem output significa que os arquivos já estavam formatados corretamente.

### `terraform validate`

Verifica se a configuração é sintaticamente válida.

```bash
terraform validate
```

```
Success! The configuration is valid.
```

---

## 7. Planejamento — `terraform plan`

Exibe o que será criado, modificado ou destruído **sem realizar nenhuma alteração real**.

```bash
terraform plan
```

```
data.aws_ami.ubuntu: Reading...
data.aws_ami.ubuntu: Read complete after 0s [id=ami-0b17d8f9ce1b14ec1]

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_instance.app_server will be created
  + resource "aws_instance" "app_server" {
      + ami                         = "ami-0b17d8f9ce1b14ec1"
      + instance_type               = "t3.micro"
      + tags                        = {
          + "Name" = "learn-terraform"
        }
      + (demais atributos conhecidos após apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

O `data source` `aws_ami` buscou automaticamente a AMI mais recente do Ubuntu 24.04 disponível em `sa-east-1`.

---

## 8. Aplicação — `terraform apply`

Cria efetivamente os recursos na AWS conforme o plano.

```bash
terraform apply -auto-approve
```

```
data.aws_ami.ubuntu: Reading...
data.aws_ami.ubuntu: Read complete after 1s [id=ami-0b17d8f9ce1b14ec1]

Terraform will perform the following actions:

  # aws_instance.app_server will be created
  + resource "aws_instance" "app_server" {
      + ami           = "ami-0b17d8f9ce1b14ec1"
      + instance_type = "t3.micro"
      + tags = {
          + "Name" = "learn-terraform"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

aws_instance.app_server: Creating...
aws_instance.app_server: Still creating... [10s elapsed]
aws_instance.app_server: Creation complete after 13s [id=i-037592dd2c8c050f4]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

A instância foi criada em **13 segundos** com o ID `i-037592dd2c8c050f4`.

---

## 9. Inspeção do Estado

### `terraform state list`

Lista todos os recursos gerenciados pelo Terraform.

```bash
terraform state list
```

```
data.aws_ami.ubuntu
aws_instance.app_server
```

### `terraform show`

Exibe o estado completo com todos os atributos dos recursos provisionados.

```bash
terraform show
```

```hcl
# aws_instance.app_server:
resource "aws_instance" "app_server" {
    ami                         = "ami-0b17d8f9ce1b14ec1"
    arn                         = "arn:aws:ec2:sa-east-1:482112737613:instance/i-037592dd2c8c050f4"
    availability_zone           = "sa-east-1a"
    id                          = "i-037592dd2c8c050f4"
    instance_state              = "running"
    instance_type               = "t3.micro"
    private_dns                 = "ip-172-31-10-142.sa-east-1.compute.internal"
    private_ip                  = "172.31.10.142"
    public_dns                  = "ec2-52-67-167-197.sa-east-1.compute.amazonaws.com"
    public_ip                   = "52.67.167.197"
    tags = {
        "Name" = "learn-terraform"
    }
}
```

---

## 10. Recursos Provisionados na AWS

### Instância EC2

| Atributo | Valor |
|---|---|
| **Instance ID** | `i-037592dd2c8c050f4` |
| **Tipo** | `t3.micro` |
| **AMI** | `ami-0b17d8f9ce1b14ec1` (Ubuntu 24.04 Noble) |
| **Região** | `sa-east-1` (São Paulo) |
| **Availability Zone** | `sa-east-1a` |
| **Estado** | `running` |
| **IP Público** | `52.67.167.197` |
| **IP Privado** | `172.31.10.142` |
| **DNS Público** | `ec2-52-67-167-197.sa-east-1.compute.amazonaws.com` |
| **Tag Name** | `learn-terraform` |
| **Volume Root** | `vol-0413d8b9336055860` — 8 GB gp3 |
| **Security Group** | `sg-098058357a9235baf` (default) |
| **ARN** | `arn:aws:ec2:sa-east-1:482112737613:instance/i-037592dd2c8c050f4` |

### AMI Utilizada

| Atributo | Valor |
|---|---|
| **AMI ID** | `ami-0b17d8f9ce1b14ec1` |
| **Nome** | `ubuntu-noble-24.04-amd64-server-20260604` |
| **Sistema Operacional** | Ubuntu 24.04 LTS (Noble Numbat) |
| **Arquitetura** | `x86_64` |
| **Tipo de Virtualização** | `hvm` |
| **Owner** | `099720109477` (Canonical) |
| **Data de criação** | 2026-06-04 |

---

## 11. Destruição da Infraestrutura — `terraform destroy`

Após concluir a atividade, os recursos foram destruídos para evitar cobranças.

```bash
terraform destroy -auto-approve
```

```
aws_instance.app_server: Destroying... [id=i-037592dd2c8c050f4]
aws_instance.app_server: Still destroying... [10s elapsed]
aws_instance.app_server: Still destroying... [20s elapsed]
aws_instance.app_server: Still destroying... [30s elapsed]
aws_instance.app_server: Destruction complete after 32s

Destroy complete! Resources: 1 destroyed.
```

> O `terraform destroy` remove **todos** os recursos gerenciados pelo estado atual. Sempre execute antes de encerrar uma atividade para não gerar custos desnecessários.

---

## 12. Referências

- [HashiCorp Terraform — Get Started AWS](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-create)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Ubuntu AMIs na AWS](https://cloud-images.ubuntu.com/locator/ec2/)
