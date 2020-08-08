`terraform init` に成功するまでに少し詰まったので、構文理解のためにも成功前後のファイル差分と解決までの道程を残しておく。

# やること

`terraform init` まずはこれだけ。

# init 成功ファイル

## vpc.tf

```t
provider "aws" {
  version = "~> 3.1"
}

variable "project_prefix" {}
variable "vpc_cidr" {}

resource "aws_vpc" "vpc" {
  name             = "${var.project_prefix}-vpc"
  cidr_block       = var.vpc_cidr
  instance_tenancy = "default"

  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = var.project_prefix
  }
}
```

# エラー解決の道程

## 元ファイル

```t
provider "aws" {}

variable "project_prefix" {}
variable "vpc_cidr" {}

resource "aws_vpc" "${var.project_prefix}-vpc" {
  cidr_block       = "${var.vpc_cidr}"
  instance_tenancy = "default"

  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.project_prefix}"
  }
}
```

## The following providers do not have any version

### 該当箇所

```t
provider "aws" {}
```

### warning

```yml
The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.aws: version = "~> 3.1"
```

### 修正後

バージョンが上がった時に再現性がないから、指定しとけよってことらしい。

```tf
provider "aws" {
  version = "~> 3.1"
}
```

## Interpolation-only expressions are deprecated

### 該当箇所

```t
  cidr_block       = "${var.vpc_cidr}"
```

### warning

```yml
Warning: Interpolation-only expressions are deprecated

  on vpc.tf line 9, in resource "aws_vpc" "${ ... }-vpc":
   9:   cidr_block       = "${var.vpc_cidr}"

Terraform 0.11 and earlier required all non-constant expressions to be
provided via interpolation syntax, but this pattern is now deprecated. To
silence this warning, remove the "${ sequence from the start and the }"
sequence from the end of this expression, leaving just the inner expression.

Template interpolation syntax is still used to construct strings from
expressions when the template includes multiple interpolation sequences or a
mixture of literal strings and interpolations. This deprecation applies only
to templates that consist entirely of a single interpolation sequence.
```

### 修正後

普通に変数使うときは、わざわざ `""` で囲わんでもええで、ってことらしい。

```t
  cidr_block       = var.vpc_cidr
```

## Invalid resource name

### 該当箇所

```t
resource "aws_vpc" "${var.project_prefix}-vpc" {
```

### Error

```yml
Error: Invalid resource name

  on vpc.tf line 8, in resource "aws_vpc" "${ ... }-vpc":
   8: resource "aws_vpc" "${var.project_prefix}-vpc" {

A name must start with a letter or underscore and may contain only letters,
digits, underscores, and dashes.

Error: Invalid string literal

  on vpc.tf line 8, in resource "aws_vpc" "${ ... }-vpc":
   8: resource "aws_vpc" "${var.project_prefix}-vpc" {

Template sequences are not allowed in this string. To include a literal "$",
double it (as "$$") to escape it.
```

### 修正後

ここは AWS 上の名前ではなく、 Terraform 都合の名前を置く場所らしい。
名前は tags のプロパティを用いて指定する。

```t
resource "aws_vpc" "vpc" {

  tags = {
    Name = "${var.project_prefix}-vpc"
  }
```

## successed!

```bash
$ terraform init
```

```yml
Initializing the backend...

Initializing provider plugins...

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

# 次やりたいこと

- plan, apply

- Docker 化
- リソース間連携
  - VPC に IGW 生やして Attach したり
- module 分割
- workspace の活用
