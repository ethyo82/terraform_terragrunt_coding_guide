# Terraform 코드 컨벤션 가이드

> 이 문서는 팀 내 Terraform 코드 작성 시 일관성을 유지하기 위한 컨벤션 가이드입니다.
> 프로젝트 환경에 맞게 수정하여 사용하세요.

---

## 목차

1. [변수 타입 선택 기준](#1-변수-타입-선택-기준)
2. [함수 사용 규칙](#2-함수-사용-규칙)
3. [네이밍 컨벤션](#3-네이밍-컨벤션)
4. [파일 구조](#4-파일-구조)
5. [모듈 설계 원칙](#5-모듈-설계-원칙)
6. [코드 리뷰 체크리스트](#6-코드-리뷰-체크리스트)

---

## 1. 변수 타입 선택 기준

### 1.1 기본 타입 선택 매트릭스

| 상황 | 권장 타입 | 예시 |
|------|----------|------|
| 단일 값 | `string`, `number`, `bool` | `instance_type = "t3.micro"` |
| 순서가 중요한 목록 | `list(type)` | 서브넷 CIDR 순서 |
| 순서 무관, 중복 불허 | `set(type)` | 보안 그룹 규칙 |
| 키-값 매핑 | `map(type)` | 태그, 환경별 설정 |
| 복합 구조 | `object({...})` | 상세 설정이 필요한 리소스 |
| 유연한 입력 | `any` | 외부 모듈 호환성 |

### 1.2 list vs set vs map 선택 기준

```hcl
# ✅ list 사용: 순서가 의미 있을 때
variable "availability_zones" {
  description = "AZ 목록 (첫 번째가 primary)"
  type        = list(string)
  default     = ["ap-northeast-2a", "ap-northeast-2c"]
}

# ✅ set 사용: 순서 무관, for_each와 함께 사용
variable "allowed_ports" {
  description = "허용할 포트 목록"
  type        = set(number)
  default     = [80, 443, 8080]
}

# ✅ map 사용: 키로 참조가 필요할 때
variable "environment_configs" {
  description = "환경별 설정"
  type = map(object({
    instance_type = string
    min_size      = number
    max_size      = number
  }))
}
```

### 1.3 object vs map 선택 기준

```hcl
# ✅ object: 정해진 스키마가 있을 때
variable "database_config" {
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    allocated_storage = number
  })
}

# ✅ map: 동적 키가 필요할 때
variable "tags" {
  type    = map(string)
  default = {}
}
```

### 1.4 타입 선택 결정 트리

```
단일 값인가?
├─ Yes → string / number / bool
└─ No → 여러 값인가?
         ├─ 키-값 쌍인가?
         │   ├─ Yes → 키가 정해져 있나?
         │   │         ├─ Yes → object({...})
         │   │         └─ No → map(type)
         │   └─ No → 순서가 중요한가?
         │            ├─ Yes → list(type)
         │            └─ No → set(type)
         └─ 복잡한 중첩 구조 → list(object({...})) 또는 map(object({...}))
```

---

## 2. 함수 사용 규칙

### 2.1 함수 선택 가이드

| 용도 | 권장 함수 | 비권장 (레거시) | 비고 |
|------|----------|----------------|------|
| 기본값 처리 | `try()`, `coalesce()` | `lookup()`, `element(concat(...))` | Terraform 0.13+ |
| 조건부 값 | `condition ? true : false` | `count` 기반 조건 | 단순 조건에 사용 |
| 리스트 병합 | `flatten()` | `concat()` 중첩 | 다차원 리스트 |
| null 처리 | `coalesce()` | `var != null ? var : default` | 첫 번째 non-null 반환 |
| 맵 병합 | `merge()` | 수동 병합 | 오른쪽 우선 |

### 2.2 try() vs lookup() 사용 기준

```hcl
# ✅ try() 권장: 복잡한 접근 경로, null 안전
output "instance_ip" {
  value = try(aws_instance.this[0].private_ip, "N/A")
}

# ✅ lookup(): 단순 맵 조회 (여전히 유효)
locals {
  instance_type = lookup(var.instance_types, var.environment, "t3.micro")
}

# ❌ 비권장: element(concat(...)) - 레거시 패턴
output "old_pattern" {
  value = element(concat(aws_instance.this.*.id, [""]), 0)
}
```

### 2.3 조건문 패턴

```hcl
# ✅ 권장: count에서 boolean 사용
resource "aws_instance" "this" {
  count = var.create_instance ? 1 : 0
  # ...
}

# ✅ 권장: for_each에서 조건부 맵
resource "aws_security_group_rule" "ingress" {
  for_each = var.enable_ingress ? var.ingress_rules : {}
  # ...
}

# ❌ 비권장: length() 기반 조건
resource "aws_instance" "this" {
  count = length(var.instances) > 0 ? 1 : 0  # boolean 사용 권장
}
```

### 2.4 문자열 처리 함수

```hcl
# 문자열 조합
locals {
  # ✅ format(): 복잡한 포맷팅
  bucket_name = format("%s-%s-%s", var.project, var.environment, var.region)
  
  # ✅ join(): 리스트를 문자열로
  subnet_list = join(", ", var.subnet_ids)
  
  # ✅ replace(): 문자열 치환
  safe_name = replace(var.name, "-", "_")
}
```

---

## 3. 네이밍 컨벤션

### 3.1 일반 규칙

| 항목 | 규칙 | 예시 |
|------|------|------|
| 리소스/변수/출력 | snake_case (언더스코어) | `aws_instance.web_server` |
| 리소스 이름 | 타입 반복 금지 | ✅ `aws_vpc.main` / ❌ `aws_vpc.main_vpc` |
| 단일 리소스 | `this` 사용 | `aws_nat_gateway.this` |
| 복수 리소스 | 설명적 이름 | `aws_subnet.public`, `aws_subnet.private` |
| 변수명 | 단수/복수 구분 | `list` → 복수형, 단일 값 → 단수형 |

### 3.2 리소스 네이밍

```hcl
# ✅ 올바른 예시
resource "aws_vpc" "main" {}
resource "aws_subnet" "public" {}
resource "aws_subnet" "private" {}
resource "aws_nat_gateway" "this" {}  # 단일 리소스

# ❌ 잘못된 예시
resource "aws_vpc" "main_vpc" {}        # 타입 반복
resource "aws_subnet" "public_subnet" {} # 타입 반복
```

### 3.3 변수 네이밍

```hcl
# ✅ 복수형: list/set/map 타입
variable "subnet_ids" {
  type = list(string)
}

variable "security_group_rules" {
  type = map(object({...}))
}

# ✅ 단수형: 단일 값
variable "instance_type" {
  type = string
}

variable "enable_monitoring" {
  type = bool
}
```

### 3.4 출력 네이밍

```hcl
# 패턴: {name}_{type}_{attribute}
output "web_instance_id" {
  description = "웹 서버 인스턴스 ID"
  value       = aws_instance.web.id
}

output "public_subnet_ids" {
  description = "퍼블릭 서브넷 ID 목록"
  value       = aws_subnet.public[*].id
}
```

---

## 4. 파일 구조

### 4.1 표준 파일 구성

```
module/
├── main.tf          # 주요 리소스 정의
├── variables.tf     # 입력 변수 정의
├── outputs.tf       # 출력 정의
├── versions.tf      # 프로바이더/Terraform 버전
├── locals.tf        # 로컬 변수 (선택)
├── data.tf          # 데이터 소스 (선택)
└── README.md        # 모듈 문서
```

### 4.2 대규모 프로젝트 구조

```
infrastructure/
├── modules/
│   ├── vpc/
│   ├── eks/
│   └── rds/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   └── prod/
└── global/
    └── iam/
```

### 4.3 변수 블록 순서

```hcl
variable "example" {
  description = "변수 설명 (필수)"  # 1. description
  type        = string              # 2. type
  default     = "value"             # 3. default
  nullable    = false               # 4. nullable (선택)
  sensitive   = true                # 5. sensitive (선택)
  
  validation {                      # 6. validation (선택)
    condition     = length(var.example) > 0
    error_message = "값이 비어있을 수 없습니다."
  }
}
```

### 4.4 리소스 블록 순서

```hcl
resource "aws_instance" "web" {
  # 1. count/for_each (첫 번째, 빈 줄로 구분)
  count = var.instance_count

  # 2. 필수 인자
  ami           = var.ami_id
  instance_type = var.instance_type

  # 3. 선택 인자
  subnet_id = var.subnet_id

  # 4. 중첩 블록
  root_block_device {
    volume_size = 20
  }

  # 5. tags (마지막 실제 인자)
  tags = var.tags

  # 6. depends_on (있는 경우)
  depends_on = [aws_iam_role.example]

  # 7. lifecycle (있는 경우)
  lifecycle {
    create_before_destroy = true
  }
}
```

---

## 5. 모듈 설계 원칙

### 5.1 입력 변수 설계

```hcl
# ✅ 간단한 타입 우선 (object보다 개별 변수)
variable "instance_type" {
  description = "EC2 인스턴스 타입"
  type        = string
  default     = "t3.micro"
}

variable "enable_monitoring" {
  description = "상세 모니터링 활성화 여부"
  type        = bool
  default     = false
}

# ✅ 복잡한 설정은 object 사용
variable "autoscaling_config" {
  description = "Auto Scaling 그룹 설정"
  type = object({
    min_size         = number
    max_size         = number
    desired_capacity = number
    health_check_type = optional(string, "EC2")
  })
  default = null
}
```

### 5.2 출력 설계

```hcl
# ✅ 항상 description 포함
output "vpc_id" {
  description = "생성된 VPC의 ID"
  value       = aws_vpc.main.id
}

# ✅ 조건부 리소스 출력
output "nat_gateway_ip" {
  description = "NAT Gateway의 퍼블릭 IP (생성된 경우)"
  value       = try(aws_eip.nat[0].public_ip, null)
}
```

### 5.3 조건부 리소스 생성

```hcl
# ✅ boolean 변수로 제어
variable "create_nat_gateway" {
  description = "NAT Gateway 생성 여부"
  type        = bool
  default     = true
}

resource "aws_nat_gateway" "this" {
  count = var.create_nat_gateway ? 1 : 0
  # ...
}

resource "aws_eip" "nat" {
  count = var.create_nat_gateway ? 1 : 0
  # ...
}
```

---

## 6. 코드 리뷰 체크리스트

### 6.1 필수 검토 항목

- [ ] 모든 변수에 `description` 있음
- [ ] 모든 출력에 `description` 있음
- [ ] `terraform fmt` 적용됨
- [ ] `terraform validate` 통과
- [ ] 민감 정보에 `sensitive = true` 설정
- [ ] 하드코딩된 값 없음 (변수화)

### 6.2 타입 일관성 검토

- [ ] list/set/map 사용이 적절함
- [ ] 복수형 변수명은 컬렉션 타입임
- [ ] object vs map 선택이 적절함
- [ ] `any` 타입 사용이 최소화됨

### 6.3 함수 사용 검토

- [ ] `try()` 사용 (레거시 `element(concat(...))` 대신)
- [ ] 조건문에서 boolean 사용 (`length() > 0` 대신)
- [ ] `coalesce()` 사용 (null 처리 시)

### 6.4 보안 검토

- [ ] 기본값에 민감 정보 없음
- [ ] IAM 정책 최소 권한 원칙 적용
- [ ] 보안 그룹 규칙 명시적 정의

---

## 부록: 프로젝트별 커스터마이징

### A. 프로젝트 정보

| 항목 | 값 |
|------|------|
| 프로젝트명 | (프로젝트명 입력) |
| Terraform 버전 | (예: >= 1.5.0) |
| 주요 프로바이더 | (예: AWS, Azure, GCP) |
| 팀/담당자 | (팀명 또는 담당자) |

### B. 프로젝트 특화 규칙

```
이 섹션에 프로젝트 특화 규칙을 추가하세요:
- 특정 리소스 네이밍 규칙
- 환경별 변수 관리 방법
- 모듈 버전 관리 정책
- 기타 팀 합의 사항
```

### C. 변경 이력

| 날짜 | 버전 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| YYYY-MM-DD | 1.0.0 | 초기 작성 | (이름) |

---

## 참고 자료

- [Terraform 공식 스타일 가이드](https://developer.hashicorp.com/terraform/language/style)
- [Terraform Best Practices](https://www.terraform-best-practices.com)
- [Google Cloud Terraform 가이드](https://cloud.google.com/docs/terraform/best-practices-for-terraform)