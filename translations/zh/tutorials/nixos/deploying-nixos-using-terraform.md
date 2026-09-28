---
myst:
  html_meta:
    "description lang=en": "Continuous Integration with GitHub Actions and Cachix"
    "keywords": "NixOS, deployment, Terraform, AWS"
---

(deploy-nixos-using-terraform)=
# 使用 Terraform 部署 NixOS

```{contributors}
:authors: domenkozar
```
本教程假定你已[熟悉 Terraform 基础知识](https://www.terraform.io/intro/index.html)。
学完后，你将用 Terraform 开通一台 Amazon Web Services (AWS) 实例，并用 Nix 对实例上运行的 NixOS 部署增量变更。

我们将了解如何启动一台 NixOS 机器，以及如何部署增量变更。

## 启动 NixOS 镜像

1. 首先提供 Terraform 可执行文件：

```shell-session
$ nix-shell -p terraform
```

2. 我们使用 [Terraform Cloud](https://app.terraform.io) 作为 [state/locking 后端](https://www.terraform.io/docs/state/purpose.html)：

```shell-session
$ terraform login
```

3. 请确保在 Terraform Cloud 账户中[创建一个组织](https://app.terraform.io/app/organizations/new)，例如 `myorganization`。
4. 在 `myorganization` 中，选择 **CLI-driven workflow** [创建一个工作区](https://app.terraform.io/app/cachix/workspaces/new)，并取一个名称，例如 `myapp`。
5. 在工作区中，进入 `Settings / General`，将 Execution Mode 改为 `Local`。
6. 在一个新目录中，创建包含以下内容的 `main.tf` 文件。这将启动一台使用 NixOS 镜像的 AWS 实例，并配置一个 SSH 密钥对和一个 SSH 安全组：

```terraform
terraform {
    backend "remote" {
        organization = "myorganization"

        workspaces {
            name = "myapp"
        }
    }
}

provider "aws" {
    region = "eu-central-1"
}

module "nixos_image" {
    source  = "git::https://github.com/tweag/terraform-nixos.git//aws_image_nixos?ref=5f5a0408b299874d6a29d1271e9bffeee4c9ca71"
    release = "20.09"
}

resource "aws_security_group" "ssh_and_egress" {
    ingress {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = [ "0.0.0.0/0" ]
    }

    egress {
        from_port       = 0
        to_port         = 0
        protocol        = "-1"
        cidr_blocks     = ["0.0.0.0/0"]
    }
}

resource "tls_private_key" "state_ssh_key" {
    algorithm = "RSA"
}

resource "local_file" "machine_ssh_key" {
    sensitive_content = tls_private_key.state_ssh_key.private_key_pem
    filename          = "${path.module}/id_rsa.pem"
    file_permission   = "0600"
}

resource "aws_key_pair" "generated_key" {
    key_name   = "generated-key-${sha256(tls_private_key.state_ssh_key.public_key_openssh)}"
    public_key = tls_private_key.state_ssh_key.public_key_openssh
}

resource "aws_instance" "machine" {
    ami             = module.nixos_image.ami
    instance_type   = "t3.micro"
    security_groups = [ aws_security_group.ssh_and_egress.name ]
    key_name        = aws_key_pair.generated_key.key_name

    root_block_device {
        volume_size = 50 # GiB
    }
}

output "public_dns" {
    value = aws_instance.machine.public_dns
}
```

其中与 NixOS 相关的片段只有：

```terraform
module "nixos_image" {
  source = "git::https://github.com/tweag/terraform-nixos.git/aws_image_nixos?ref=5f5a0408b299874d6a29d1271e9bffeee4c9ca71"
  release = "20.09"
}
```

:::{note}
给定 [NixOS 发行版本号](https://status.nixos.org)，`aws_image_nixos` 模块会返回对应的 NixOS AMI，
以便 `aws_instance` 资源可以在 [instance_type](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance#instance_type) 参数中引用该 AMI。
:::

5. 请确保[配置 AWS 凭据](https://registry.terraform.io/providers/hashicorp/aws/latest/docs#authentication)。
6. 应用 Terraform 配置后，你应该会得到一台正在运行的 NixOS：

```shell-session
$ terraform init
$ terraform apply
```

## 部署 NixOS 变更

一旦 AWS 实例通过 Terraform 运行了 NixOS 镜像，我们就可以让 Terraform 始终构建最新的 NixOS 配置，并将这些变更应用到你的实例上。

1. 创建包含以下内容的 `configuration.nix`：

```nix
{ config, lib, pkgs, ... }: {
  imports = [ <nixpkgs/nixos/modules/virtualisation/amazon-image.nix> ];

  # Open https://search.nixos.org/options for all options
}
```

2. 将以下片段追加到你的 `main.tf`：

```terraform
module "deploy_nixos" {
    source = "git::https://github.com/tweag/terraform-nixos.git//deploy_nixos?ref=5f5a0408b299874d6a29d1271e9bffeee4c9ca71"
    nixos_config = "${path.module}/configuration.nix"
    target_host = aws_instance.machine.public_ip
    ssh_private_key_file = local_file.machine_ssh_key.filename
    ssh_agent = false
}
```

3. 部署：

```shell-session
$ terraform init
$ terraform apply
```

## 注意事项

- `deploy_nixos` 模块要求目标机器上已安装 NixOS，主机上已安装 Nix。
- 当客户端与目标架构不同时，`deploy_nixos` 模块无法工作（除非你使用[分布式构建](https://nix.dev/manual/nix/stable/advanced-topics/distributed-builds.html)）。
- 如果需要向 Nix 注入值，目前没有优雅的解决方案。
- 每台机器是单独求值的，因此请注意：内存需求会随机器数量线性增长。

## 下一步

- 可以[切换到 Google Compute Engine](https://github.com/tweag/terraform-nixos/tree/master/google_image_nixos#readme)。
- [`deploy_nixos` 模块](https://github.com/tweag/terraform-nixos/tree/master/deploy_nixos#readme) 支持若干参数，例如用于上传密钥。
