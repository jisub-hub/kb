---
tags:
  - infra
  - terraform
  - iac
  - devops
created: 2026-06-15
---

# Terraform

> [!summary] 한 줄 요약
> 클라우드 인프라(VPC, EC2, RDS, EKS 등)를 **HCL 코드로 선언**하고 버전 관리하는 IaC 도구. 어떤 상태로 만들지만 정의하면 Terraform이 현재 상태와 비교해 최소 변경을 적용한다.

---

## 1. 핵심 개념

| 개념 | 설명 |
|------|------|
| **Provider** | AWS·GCP·Azure 등 API 드라이버 |
| **Resource** | 실제로 만들 인프라 리소스 |
| **Data Source** | 기존 리소스 조회 (읽기 전용) |
| **Variable** | 입력값 파라미터화 |
| **Output** | 다른 모듈/사용자에게 노출할 값 |
| **State** | 실제 인프라와 코드의 매핑 기록 (`terraform.tfstate`) |
| **Module** | 재사용 가능한 리소스 묶음 |

---

## 2. HCL 기본 문법

```hcl
# provider.tf
terraform {
  required_version = ">= 1.7"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  # 원격 상태 (팀 협업 필수)
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-northeast-2"
    dynamodb_table = "tf-lock"   # 동시 apply 방지
  }
}

provider "aws" {
  region = var.aws_region
}
```

```hcl
# variables.tf
variable "aws_region" {
  type        = string
  default     = "ap-northeast-2"
  description = "배포 리전"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}
```

```hcl
# main.tf
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "main-vpc" }
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id    # 다른 리소스 속성 참조
  cidr_block        = "10.0.1.0/24"
  availability_zone = "ap-northeast-2a"
}

# 기존 리소스 조회 (Data Source)
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public.id
  tags          = { Name = "web-server" }
}
```

```hcl
# outputs.tf
output "web_public_ip" {
  value       = aws_instance.web.public_ip
  description = "웹 서버 퍼블릭 IP"
}
```

---

## 3. 워크플로우

```bash
terraform init          # 프로바이더 다운로드, 백엔드 초기화
terraform fmt           # 코드 포맷팅
terraform validate      # 문법 검증

terraform plan          # 변경사항 미리보기 (실제 적용 ❌)
terraform plan -out=tfplan   # 플랜 파일 저장

terraform apply         # 적용 (확인 프롬프트)
terraform apply tfplan  # 저장된 플랜 그대로 적용
terraform apply -auto-approve  # CI/CD 환경

terraform destroy       # 모든 리소스 삭제 ⚠️
terraform state list    # 관리 중인 리소스 목록
terraform state show aws_instance.web  # 특정 리소스 상태
```

---

## 4. 모듈화 — 재사용 패턴

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
└── modules/
    └── eks-cluster/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

```hcl
# 모듈 호출
module "eks" {
  source          = "./modules/eks-cluster"
  cluster_name    = "prod-cluster"
  node_group_size = 3
}

output "cluster_endpoint" {
  value = module.eks.endpoint
}
```

---

## 5. 반복·조건 표현

```hcl
# count — 동일 리소스 N개 생성
resource "aws_subnet" "private" {
  count      = 3
  cidr_block = "10.0.${count.index + 10}.0/24"
}

# for_each — 맵/셋으로 반복
variable "buckets" {
  default = { logs = "ap-northeast-2", backup = "us-east-1" }
}

resource "aws_s3_bucket" "store" {
  for_each = var.buckets
  bucket   = "myorg-${each.key}"

  provider = aws.${each.value}   # 리전별 프로바이더
}

# 조건식 (삼항)
resource "aws_instance" "bastion" {
  instance_type = var.env == "prod" ? "t3.medium" : "t3.micro"
}
```

---

## 6. 실전 AWS 패턴 — EKS 클러스터

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "my-cluster"
  cluster_version = "1.31"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  eks_managed_node_groups = {
    default = {
      min_size       = 2
      max_size       = 5
      desired_size   = 3
      instance_types = ["t3.medium"]
    }
  }
}
```

---

## 7. 모듈 패턴 — 환경별 재사용

```
infra/
├── modules/
│   ├── vpc/          ← 재사용 모듈
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── eks/
│   └── rds/
└── envs/
    ├── dev/          ← 환경별 설정
    │   ├── main.tf
    │   └── terraform.tfvars
    ├── staging/
    └── prod/
```

```hcl
# envs/prod/main.tf — 모듈 호출
module "vpc" {
  source = "../../modules/vpc"
  # 또는 Terraform Registry: source = "terraform-aws-modules/vpc/aws"

  name             = "prod-vpc"
  cidr             = "10.0.0.0/16"
  azs              = ["ap-northeast-2a", "ap-northeast-2c"]
  private_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets   = ["10.0.101.0/24", "10.0.102.0/24"]
  enable_nat_gateway = true
}

module "rds" {
  source = "../../modules/rds"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  db_name    = var.db_name
  db_password = var.db_password   # tfvars 또는 env로 주입
}

# 모듈 출력 참조
output "vpc_id" {
  value = module.vpc.vpc_id
}
```

```hcl
# envs/prod/terraform.tfvars
environment  = "prod"
instance_type = "t3.medium"
min_capacity  = 3
```

---

## 8. Workspace — 환경 분리 (소규모)

```bash
# workspace로 같은 코드로 dev/prod 분리 (소규모 팀)
terraform workspace new dev
terraform workspace new prod
terraform workspace list
terraform workspace select prod

# workspace별 다른 변수 적용
terraform apply -var-file="prod.tfvars"
```

```hcl
# workspace 이름으로 동적 분기
locals {
  env     = terraform.workspace              # "dev" / "prod"
  is_prod = terraform.workspace == "prod"
}

resource "aws_instance" "app" {
  instance_type = local.is_prod ? "t3.large" : "t3.micro"
  count         = local.is_prod ? 3 : 1
}
```

> 복잡한 프로젝트는 `workspace` 대신 **디렉토리 분리(`envs/`)** 권장 (상태 파일 혼재 방지).

---

## 9. 베스트 프랙티스

- **State는 원격 저장소** (S3 + DynamoDB 락) — 로컬 tfstate 팀 작업 금지
- **plan 결과 반드시 검토** — apply 전 diff 확인
- **모듈화** — 환경별(dev/staging/prod) 값만 variables로 분리
- **민감값은 변수로** — 하드코딩 금지, AWS Secrets Manager 연동
- **태그 표준화** — `Environment`, `Owner`, `Project` 태그 일관 적용
- **`terraform fmt` + `validate`** CI에 포함
- **`.terraform.lock.hcl` 커밋** — provider 버전 고정

---

## 관련
- [[Ansible]] · [[Kubernetes]] · [[infra/_index]]
