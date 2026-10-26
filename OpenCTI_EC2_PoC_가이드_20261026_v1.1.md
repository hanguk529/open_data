# OpenCTI on AWS EC2 — 단독 PoC 설치 가이드

| 항목 | 내용 |
|---|---|
| 문서명 | OpenCTI EC2 PoC 가이드 |
| 버전 | v1.1 |
| 작성일 | 2026-10-26 |
| 목적 | 사내 SOC용 OpenCTI TI 적재소 도입 전, 단일 EC2 + Docker Compose 기반 개인 PoC 수행 |
| 기준 문서 | OpenCTI 공식 문서(latest), OpenCTI-Platform/docker 저장소, AWS 공식 문서 |

---

## 0. 전체 그림

```
[내 PC 브라우저]
   │  http://localhost:8080
   ▼
[내 PC 터널] ── SSH 터널(22) 또는 SSM 포트포워딩(443 아웃바운드)
   │
   ▼  (인터넷)
[AWS 기본 VPC / 퍼블릭 서브넷]
   └─ EC2 (Ubuntu 24.04, 퍼블릭 IP 자동 할당)
        └─ Docker Compose: OpenCTI(8080) + ES + Redis + RabbitMQ + S3
```

- 핵심 원칙: **8080 포트는 인터넷에 직접 열지 않는다.** 터널로만 접근.
- 접속 방식 3가지 중 하나 선택 (자세한 비교는 2장)

| 방식 | 난이도 | 보안 | 준비물 | 권장 |
|---|---|---|---|---|
| A. SSH 터널 | 쉬움 | 양호 (22 포트를 내 IP만 허용) | 키페어(.pem), Windows 기본 ssh | **PoC 1순위** |
| B. SSM 포트포워딩 | 중간 | 최상 (인바운드 0개) | IAM 역할, AWS CLI + Session Manager 플러그인, Access Key | 운영 전환 시 |
| C. 8080 직접 개방 | 매우 쉬움 | 낮음 (HTTP 평문) | 없음 | 비권장 (잠깐 확인용만) |

---

## 1. EC2 인스턴스 생성 (콘솔 단계별)

### 1-0. 사전: 리전 선택
- 콘솔 우측 상단 리전 → **아시아 태평양(서울) ap-northeast-2**

### 1-1. (방식 B를 쓸 경우만) IAM 역할 먼저 생성
1. 콘솔 검색창 `IAM` → **역할(Roles)** → **역할 생성**
2. 신뢰할 수 있는 엔터티: **AWS 서비스** → 사용 사례: **EC2** → 다음
3. 권한 정책 검색: `AmazonSSMManagedInstanceCore` 체크 → 다음
4. 역할 이름: `opencti-poc-ssm-role` → 생성

> 방식 A만 쓸 거면 생략 가능. 단, 붙여두면 SSH가 막혀도 콘솔에서 **브라우저 쉘(Session Manager)** 로 접속할 수 있어 비상구 역할을 하므로 **붙이는 것을 권장**.

### 1-2. 인스턴스 시작
콘솔 검색창 `EC2` → **인스턴스 시작(Launch instance)**

| 단계 | 설정값 | 설명 |
|---|---|---|
| ① 이름 및 태그 | `opencti-poc` | |
| ② AMI | **Ubuntu Server 24.04 LTS (HVM), SSD Volume Type** / 아키텍처 **64비트(x86)** | 이름에 "Pro" 없는 것. 설명에 `noble` 표기 = 24.04 |
| ③ 인스턴스 유형 | `m6i.2xlarge`(8vCPU/32GB) 권장, 최소 `t3.xlarge`/`m6i.xlarge`(16GB) | 프리티어(micro)로는 구동 불가 |
| ④ 키 페어 | **새 키 페어 생성** → 이름 `opencti-poc-key`, 유형 **RSA**, 형식 **.pem** → 자동 다운로드됨 | 방식 A 필수. 재다운로드 불가하니 안전한 곳에 보관 |
| ⑤ 네트워크 설정 | **편집(Edit)** 클릭 후 아래 1-3 참조 | 가장 중요 |
| ⑥ 스토리지 | **30 GiB, gp3** (프리티어 범위) | 부족 시 무중단 확장 (8장) |
| ⑦ 고급 세부 정보 | **IAM 인스턴스 프로파일** → `opencti-poc-ssm-role` | 1-1 생성한 경우 |

→ **인스턴스 시작**

### 1-3. 네트워크 설정 상세 (⑤ 편집 화면)

| 항목 | 설정값 | 이유 |
|---|---|---|
| VPC | **기본 VPC (default)** | 별도 구성 불필요 |
| 서브넷 | **기본 설정 없음** 또는 아무 퍼블릭 서브넷 (기본 VPC의 서브넷은 모두 퍼블릭) | |
| 퍼블릭 IP 자동 할당 | **활성화(Enable)** | docker pull, git, 피드 수집, SSM 통신 등 아웃바운드 인터넷에 필요 |
| 방화벽(보안 그룹) | **보안 그룹 생성**, 이름 `opencti-poc-sg` | |
| 인바운드 규칙 | 방식별 아래 표 참조 | |

**인바운드 규칙 (방식별)**

| 방식 | 유형 | 포트 | 소스 |
|---|---|---|---|
| A. SSH 터널 | SSH | 22 | **내 IP (My IP)** ← 드롭다운에서 선택 |
| B. SSM | (규칙 없음, 기본 SSH 규칙도 삭제) | - | - |
| C. 직접 개방 | 사용자 지정 TCP | 8080 | **내 IP (My IP)** — 절대 0.0.0.0/0 금지 |

- **아웃바운드 규칙**: 기본값(모두 허용) 유지. SSM은 아웃바운드 443이 필요함.
- 마법사 기본값은 SSH 22를 `0.0.0.0/0`(위치 무관)으로 채워두는 경우가 있음 → **반드시 "내 IP"로 변경**.

> ⚠️ "내 IP"는 현재 접속한 공인 IP 기준. 회사/집/VPN 등 장소가 바뀌면 접속 안 됨 → 보안 그룹 인바운드 규칙에서 소스를 다시 "내 IP"로 갱신.

### 1-4. 생성 후 확인
- EC2 → 인스턴스 → `opencti-poc` 선택
  - 인스턴스 상태: **실행 중**, 상태 검사: **2/2 통과**
  - **퍼블릭 IPv4 주소** 메모 (예: `3.35.x.x`)
- ⚠️ 퍼블릭 IP는 **Stop → Start 시 바뀜**. 고정이 필요하면 탄력적 IP(Elastic IP) 할당 (IP당 시간 과금 동일).

---

## 2. 내 PC에서 접속하기

### 방식 A. SSH 터널 (권장) — Windows 기준

**A-1. 키 파일 권한 정리 (최초 1회)**
Windows OpenSSH는 키 파일 권한이 넓으면 `UNPROTECTED PRIVATE KEY FILE` 오류로 거부함.
PowerShell에서:
```powershell
mkdir $env:USERPROFILE\.ssh -Force
move $env:USERPROFILE\Downloads\opencti-poc-key.pem $env:USERPROFILE\.ssh\
icacls $env:USERPROFILE\.ssh\opencti-poc-key.pem /inheritance:r
icacls $env:USERPROFILE\.ssh\opencti-poc-key.pem /grant:r "$($env:USERNAME):(R)"
```

**A-2. 서버 접속 (설치 작업용)**
```powershell
ssh -i $env:USERPROFILE\.ssh\opencti-poc-key.pem ubuntu@<퍼블릭IP>
```
- 기본 사용자명: **ubuntu**
- 첫 접속 시 fingerprint 질문 → `yes`

**A-3. 웹 접속용 터널 (설치 완료 후)**
```powershell
ssh -i $env:USERPROFILE\.ssh\opencti-poc-key.pem -N -L 8080:localhost:8080 ubuntu@<퍼블릭IP>
```
- `-L 8080:localhost:8080` : 내 PC 8080 → 서버 내부 8080으로 전달
- `-N` : 쉘 없이 터널만 유지 (창을 열어둔 동안 유지, 종료는 Ctrl+C)
- 브라우저: **http://localhost:8080**

**편의: `~/.ssh/config` 등록**
`C:\Users\<사용자>\.ssh\config` 파일 생성:
```
Host opencti
    HostName <퍼블릭IP>
    User ubuntu
    IdentityFile ~/.ssh/opencti-poc-key.pem
    LocalForward 8080 localhost:8080
```
이후 `ssh opencti` 한 줄로 접속 + 터널 동시 수립. IP 바뀌면 HostName만 수정.

### 방식 B. SSM 포트포워딩 (인바운드 0개)

**B-1. 내 PC 준비**
1. AWS CLI v2 설치: https://awscli.amazonaws.com/AWSCLIV2.msi
2. Session Manager 플러그인 설치 (관리자 권한 필요):
   https://s3.amazonaws.com/session-manager-downloads/plugin/latest/windows/SessionManagerPluginSetup.exe
3. 자격증명 설정: IAM → 사용자 → 보안 자격 증명 → **액세스 키 생성** 후
   ```powershell
   aws configure
   # Access Key / Secret Key / region: ap-northeast-2 / output: json
   ```
4. 설치 확인
   ```powershell
   session-manager-plugin
   ```
   → "The Session Manager plugin was installed successfully" 출력되면 정상

**B-2. 서버 쉘 접속 (설치 작업용)**
- 콘솔: EC2 → 인스턴스 선택 → **연결(Connect)** → **Session Manager** 탭 → 연결 (브라우저 쉘)
- 또는 CLI: `aws ssm start-session --target i-xxxxxxxxxxxx`
- 브라우저 쉘은 `ssm-user`로 접속됨 → `sudo su - ubuntu`로 전환 권장

**B-3. 웹 접속용 포트포워딩**
```powershell
aws ssm start-session --target i-xxxxxxxxxxxx `
  --document-name AWS-StartPortForwardingSession `
  --parameters '{\"portNumber\":[\"8080\"],\"localPortNumber\":[\"8080\"]}'
```
- PowerShell은 JSON 따옴표 이스케이프 주의 (위 형식). CMD에서는:
  `--parameters "portNumber"=["8080"],"localPortNumber"=["8080"]`
- 브라우저: **http://localhost:8080**

> SSM 연결 실패 시 확인: ① IAM 역할 부착 여부 ② 퍼블릭 IP/아웃바운드 443 ③ 역할 부착 후 수 분 대기 (Fleet Manager에 인스턴스 표시되는지 확인)

### 방식 C. 8080 직접 개방 (비권장)
- 보안 그룹 인바운드 TCP 8080, 소스 **내 IP**
- 브라우저: `http://<퍼블릭IP>:8080`
- HTTP 평문이라 로그인 정보가 암호화되지 않음 → 잠깐 확인용으로만, 확인 후 규칙 삭제

---

## 3. 커널 파라미터 (ElasticSearch 필수)

이하 서버 쉘에서 실행.
```bash
sudo sysctl -w vm.max_map_count=1048575
echo "vm.max_map_count=1048575" | sudo tee -a /etc/sysctl.conf
```

---

## 4. Docker 설치 (공식 저장소)

```bash
sudo apt update
sudo apt install -y ca-certificates curl git jq
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update   # 저장소 추가 후 반드시 먼저 실행
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

> ⚠️ 더블체크: 공식 문서 예시는 `apt install` 후 `apt update` 순서로 되어 있어 그대로 실행 시 패키지를 찾지 못할 수 있음. 위와 같이 `apt update`를 먼저 수행.

---

## 5. 저장소 클론 및 .env 작성

```bash
cd ~
git clone https://github.com/OpenCTI-Platform/docker.git
cd docker
```

공식 문서 예시 기반 `.env` 생성 (**`OPENCTI_ADMIN_PASSWORD` 반드시 변경**):

```bash
(cat << EOF
###########################
# DEPENDENCIES            #
###########################
MINIO_ROOT_USER=$(cat /proc/sys/kernel/random/uuid)
MINIO_ROOT_PASSWORD=$(cat /proc/sys/kernel/random/uuid)
RABBITMQ_DEFAULT_USER=opencti
RABBITMQ_DEFAULT_PASS=$(openssl rand -base64 32)
SMTP_HOSTNAME=localhost
OPENSEARCH_ADMIN_PASSWORD=changeme
ELASTIC_MEMORY_SIZE=4G

###########################
# COMMON                  #
###########################
XTM_COMPOSER_ID=8215614c-7139-422e-b825-b20fd2a13a23
COMPOSE_PROJECT_NAME=xtm

###########################
# OPENCTI                 #
###########################
OPENCTI_HOST=localhost
OPENCTI_PORT=8080
OPENCTI_EXTERNAL_SCHEME=http
OPENCTI_ADMIN_EMAIL=admin@opencti.io
OPENCTI_ADMIN_PASSWORD=ChangeMePlease
OPENCTI_ADMIN_TOKEN=$(cat /proc/sys/kernel/random/uuid)
OPENCTI_HEALTHCHECK_ACCESS_KEY=$(cat /proc/sys/kernel/random/uuid)
OPENCTI_ENCRYPTION_KEY=$(openssl rand -base64 32)

###########################
# OPENCTI CONNECTORS      #
###########################
CONNECTOR_EXPORT_FILE_STIX_ID=dd817c8b-abae-460a-9ebc-97b1551e70e6
CONNECTOR_EXPORT_FILE_CSV_ID=7ba187fb-fde8-4063-92b5-c3da34060dd7
CONNECTOR_EXPORT_FILE_TXT_ID=ca715d9c-bd64-4351-91db-33a8d728a58b
CONNECTOR_IMPORT_FILE_STIX_ID=72327164-0b35-482b-b5d6-a5a3f76b845f
CONNECTOR_IMPORT_DOCUMENT_ID=c3970f8a-ce4b-4497-a381-20b7256f56f0
CONNECTOR_IMPORT_FILE_YARA_ID=7eb45b60-069b-4f7f-83a2-df4d6891d5ec
CONNECTOR_IMPORT_EXTERNAL_REFERENCE_ID=d52dcbc8-fa06-42c7-bbc2-044948c87024
CONNECTOR_ANALYSIS_ID=4dffd77c-ec11-4abe-bca7-fd997f79fa36

###########################
# OPENCTI DEFAULT DATA    #
###########################
CONNECTOR_OPENCTI_ID=dd010812-9027-4726-bf7b-4936979955ae
CONNECTOR_MITRE_ID=8307ea1e-9356-408c-a510-2d7f8b28a0e2
EOF
) > .env
```

**체크포인트**
- `OPENCTI_HOST=localhost`, `OPENCTI_PORT=8080` 유지 → 터널로 `http://localhost:8080` 접속과 일치
  (방식 C 사용 시 `OPENCTI_HOST`를 퍼블릭 IP로 변경)
- `OPENCTI_ADMIN_TOKEN`은 유효한 UUID가 아니면 컨테이너가 기동되지 않음
- 최근 기본 compose에 **XTM One(AI 어시스턴트) + pgvector DB** 서비스 추가 변경 이력 있음 → `.env.sample`과 `.env` 변수 diff 확인
  ```bash
  diff <(grep -oE '^[A-Z_]+' .env.sample | sort) <(grep -oE '^[A-Z_]+' .env | sort)
  ```
- PoC에 XTM One 불필요 시 `docker-compose.yml`에서 관련 서비스 주석 처리 → 메모리 절감

---

## 6. 실행 및 상태 확인

```bash
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs -f opencti
```

- 최초 기동 시 스키마 생성 및 MITRE ATT&CK 기본 데이터 적재로 수 분 소요
- 서버 내부에서 먼저 확인: `curl -I http://localhost:8080` → 200/302 응답이면 정상
- 이후 내 PC에서 2장 터널 연결 → 브라우저 `http://localhost:8080` → `.env`의 관리자 계정 로그인

> ⚠️ Docker가 publish한 포트는 UFW 규칙을 우회함. 호스트 방화벽이 아닌 **Security Group으로만 통제**.

---

## 7. PoC 검증 항목

| # | 항목 | 검증 방법 | 판단 기준 |
|---|---|---|---|
| 1 | 적재 경로 | Google SecOps 탐지 IOC → STIX 2.1 번들 → `pycti` 또는 UI 파일 임포트 | 일방향 적재 구조 성립 여부 |
| 2 | 외부 피드 | 무료 피드 1~2개 연동 (예: AlienVault OTX, CISA KEV) | 워커 큐 처리량, ES 용량 증가 추이 |
| 3 | 데이터 영속성 | `docker compose down` → `up -d` 후 데이터 유지 확인 | 볼륨(esdata, s3data, redisdata, amqpdata) 정상 |
| 4 | 백업/복구 | EBS 스냅샷 생성 → 신규 볼륨 복원 테스트 | 복원 후 정상 기동 |
| 5 | 리소스 | `docker stats`로 ES/Platform/Redis 메모리 관찰 | 운영 사이징 산정 근거 확보 |

---

## 8. 운영 팁 / 트러블슈팅

### 8-1. 접속 문제

| 증상 | 원인 / 조치 |
|---|---|
| `ssh: connect ... timed out` | 보안 그룹 22 소스가 현재 내 IP와 다름 → "내 IP"로 갱신 / 회사망이 22 아웃바운드 차단 시 방식 B 사용 |
| `UNPROTECTED PRIVATE KEY FILE` | 2장 A-1 icacls 권한 정리 |
| `Permission denied (publickey)` | 사용자명 `ubuntu` 확인, 키 파일 경로 확인 |
| Stop/Start 후 접속 불가 | 퍼블릭 IP 변경됨 → 새 IP로 접속 (또는 Elastic IP 할당) |
| 터널은 됐는데 브라우저 무응답 | 서버에서 `curl -I localhost:8080` 확인, 컨테이너 기동 대기 |
| 로컬 8080 사용 중 오류 | `-L 18080:localhost:8080` 으로 로컬 포트 변경 → `http://localhost:18080` |
| SSM 대상 인스턴스 안 보임 | IAM 역할 부착 확인, 부착 후 수 분 대기, 아웃바운드 443 확인 |

### 8-2. 플랫폼 문제

| 증상 | 조치 |
|---|---|
| ES 컨테이너 즉시 종료 | `vm.max_map_count` 설정 확인 |
| ES 쓰기 중단 (read-only) | 디스크 95% 도달(flood stage) → 볼륨 확장 후 재개 |
| Platform OOM | `NODE_OPTIONS=--max-old-space-size=8096` 조정, 인스턴스 상향 |
| 플랫폼 기동 실패 | `.env` 누락 변수, `OPENCTI_ADMIN_TOKEN` UUID 형식 확인 |
| 적재 지연 | RabbitMQ 큐 적체 확인, worker replica 증설 |

### 8-3. 디스크 확장 (무중단)
1. EC2 → 볼륨 → 해당 볼륨 → **작업 → 볼륨 수정** → 크기 변경
2. 서버에서:
   ```bash
   lsblk
   sudo growpart /dev/nvme0n1 1
   sudo resize2fs /dev/nvme0n1p1   # Ubuntu 기본 ext4
   df -h /
   ```
- 같은 볼륨은 6시간에 1회만 수정 가능 → 한 번에 넉넉히

### 8-4. 비용 관리
| 항목 | 과금 기준 | 팁 |
|---|---|---|
| EC2 인스턴스 | 실행 시간 | **미사용 시 Stop** (가장 큰 절감) |
| EBS | 할당 용량(사용량 무관), Stop 중에도 과금 | 30GB 시작 → 필요 시 확장 |
| 퍼블릭 IPv4 | IP당 시간 과금 | Elastic IP도 동일 과금 |
| 예산 알림 | - | Billing → Budgets 월 예산 알림 설정 |

---

## 9. 작업 순서 체크리스트

- [ ] 리전 서울 선택
- [ ] (선택) IAM 역할 `opencti-poc-ssm-role` 생성
- [ ] EC2 생성: Ubuntu 24.04 x86 / 인스턴스 유형 / 키페어 / 기본 VPC / 퍼블릭 IP 활성화 / SG(22 내 IP) / 30GB gp3 / IAM 프로파일
- [ ] 상태 검사 2/2, 퍼블릭 IP 메모
- [ ] 키 권한 정리 → `ssh ubuntu@<IP>` 접속
- [ ] vm.max_map_count → Docker 설치 → clone → .env → `docker compose up -d`
- [ ] 서버 내부 `curl -I localhost:8080` 확인
- [ ] 내 PC SSH 터널 → `http://localhost:8080` 로그인
- [ ] PoC 검증 항목 수행
- [ ] 미사용 시 인스턴스 Stop

---

## 참고

- OpenCTI 공식 설치 가이드: https://docs.opencti.io/latest/deployment/installation/
- OpenCTI 아키텍처/요구사항: https://docs.opencti.io/latest/deployment/overview/
- OpenCTI 설정 파라미터: https://docs.opencti.io/latest/deployment/configuration/
- OpenCTI-Platform/docker 저장소: https://github.com/OpenCTI-Platform/docker
- docker 저장소 XTM One 추가 PR #578: https://github.com/OpenCTI-Platform/docker/pull/578
- OpenCTI Deployment Toolkit (커뮤니티): https://github.com/404SecNotFound/OpenCTI-Deployment-Toolkit
- AWS – Session Manager 플러그인 설치(Windows): https://docs.aws.amazon.com/systems-manager/latest/userguide/install-plugin-windows.html
- AWS – Session Manager 플러그인 개요: https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html
- AWS 블로그 – SSM 포트포워딩: https://aws.amazon.com/blogs/aws/new-port-forwarding-using-aws-system-manager-sessions-manager
- AWS Knowledge Center – EBS 확장 후 파일시스템 확장: https://www.repost.aws/knowledge-center/extend-linux-file-system
- AWS – 퍼블릭 IPv4 요금: https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights
- AWS – EBS 요금: https://aws.amazon.com/ebs/pricing

---

## 변경 이력

| 버전 | 일자 | 내용 |
|---|---|---|
| v1.0 | 2026-10-26 | 최초 작성 |
| v1.1 | 2026-10-26 | EC2 생성 콘솔 단계별 절차, 네트워크/보안그룹 설정, 내 PC 접속 방식 3종(SSH 터널·SSM·직접) 상세, 접속 트러블슈팅, 디스크 확장·비용 관리, 작업 체크리스트 추가 |
