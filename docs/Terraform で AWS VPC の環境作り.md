# やりたいこと

terraform の大まかな流れが知りたい。
とりあえず AWS VPC の環境を作成して、確認後廃棄。

# 各種ファイルの用意

## VPC 本体

### vpc.tf

```tf
variable "aws_region" {}

provider "aws" {
  version = "~> 3.1"
  region  = var.aws_region
}

variable "project_prefix" {}
variable "vpc_cidr" {}

resource "aws_vpc" "vpc" {
  cidr_block       = var.vpc_cidr
  instance_tenancy = "default"

  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.project_prefix}-vpc"
  }
}
```

## 変数の設定

### test.tfvars

```tfvars
project_prefix = "tftest"
vpc_cidr       = "10.0.0.0/16"
aws_region     = "ap-northeast-1"
```

## Git ignore

ref: https://github.com/github/gitignore/blob/master/Terraform.gitignore

### .gitignore

```
# Local .terraform directories
**/.terraform/*

# .tfstate files
*.tfstate
*.tfstate.*

# Crash log files
crash.log

# Exclude all .tfvars files, which are likely to contain sentitive data, such as
# password, private keys, and other secrets. These should not be part of version
# control as they are data points which are potentially sensitive and subject
# to change depending on the environment.
#
*.tfvars

# Ignore override files as they are usually used to override resources locally and so
# are not checked in
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Include override files you do wish to add to version control using negated pattern
#
# !example_override.tf

# Include tfplan files to ignore the plan output of command: terraform plan -out=tfplan
# example: *tfplan*

# Ignore CLI configuration files
.terraformrc
terraform.rc
```

# 実行

## init

```bash
$ terraform init
```

## 差分の確認

```bash
$ terraform plan -var-file=test.tfvars
```

## 適用

```bash
$ terraform apply -var-file=test.tfvars
```

AWS Console 上からでもリソースが確認できる。

## 実行結果の表示

```bash
$ terraform show
```

## 破棄

```bash
$ terraform destroy -var-file=test.tfvars
```

# おまけ

## Makefile

今回は env という変数で呼び出すファイルを制御したが、workspace を使った方が良さそう(?)

```Makefile
env=test

clean:
	rm -rf ./.terraform

init:
	terraform init

plan:
	terraform plan -var-file=$(env).tfvars

apply:
	terraform apply -var-file=$(env).tfvars

show:
	terraform show

deploy: init plan apply show

destroy:
	terraform destroy -var-file=$(env).tfvars
```

# 次やりたいこと

- Docker 化
- リソース間連携
  - VPC に IGW 生やして Attach みたいな
- module 分割
- workspace の活用
