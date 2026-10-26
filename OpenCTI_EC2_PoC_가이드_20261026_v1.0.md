# OpenCTI on AWS EC2 — 단독 PoC 설치 가이드

| 항목 | 내용 |
|---|---|
| 문서명 | OpenCTI EC2 PoC 가이드 |
| 버전 | v1.0 |
| 작성일 | 2026-10-26 |
| 목적 | 사내 SOC용 OpenCTI TI 적재소 도입 전, 단일 EC2 + Docker Compose 기반 개인 PoC 수행 |
| 기준 문서 | OpenCTI 공식 문서(latest), OpenCTI-Platform/docker 저장소 |

---

## 0. 구성 개요

- 방식: EC2 1대 + Docker Compose 단일 노드 (OpenCTI Platform, Worker, ElasticSearch, Redis, RabbitMQ, S3 호환 스토리지)
- 접속: 인바운드 포트 미개방, **SSM Session Manager 포트포워딩**으로만 웹 UI 접근
- 범위: 기능 검증 및 적재 경로 검증 (운영/HA 구성 아님)

---

## 1. EC2 준비

| 항목 | 권장값 | 비고 |
|---|---|---|
| AMI | Ubuntu 24.04 LTS (Amazon 제공) | SSM Agent 기본 설치 |
| 인스턴스 | RAM 32GB급 (예: m6i.2xlarge) | 최소 16GB (예: t3.xlarge / m6i.xlarge) |
| 스토리지 | gp3 100GB 이상 | ES 데이터 증가 고려 |
| IAM 역할 | `AmazonSSMManagedInstanceCore` | SSM 접속용 |
| Security Group | 인바운드 없음 | 8080/22 공개 금지 |

**메모리 산정 근거 (공식 문서)**
- Platform(NodeJS) 기본 메모리 한도 8GB
- 공식 `.env` 예시의 `ELASTIC_MEMORY_SIZE=4G`
- Redis 스트림 기본 200만 건 제한 시 약 8GB 사용
- 커뮤니티 툴킷 권장: 16GB RAM / 8 Core / 100GB SSD (8GB/4Core는 동작하나 느림)

---

## 2. 커널 파라미터 (ElasticSearch 필수)

```bash
sudo sysctl -w vm.max_map_count=1048575
echo "vm.max_map_count=1048575" | sudo tee -a /etc/sysctl.conf
```

---

## 3. Docker 설치 (공식 저장소)

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

## 4. 저장소 클론 및 .env 작성

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
- `OPENCTI_ADMIN_TOKEN`은 유효한 UUID가 아니면 컨테이너가 기동되지 않음
- 최근 기본 compose에 **XTM One(AI 어시스턴트) + pgvector DB** 서비스 추가 변경 이력 있음 → `.env.sample`과 `.env` 변수 diff 확인
  ```bash
  diff <(grep -oE '^[A-Z_]+' .env.sample | sort) <(grep -oE '^[A-Z_]+' .env | sort)
  ```
- PoC에 XTM One 불필요 시 `docker-compose.yml`에서 관련 서비스 주석 처리 → 메모리 절감

---

## 5. 실행 및 상태 확인

```bash
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs -f opencti
```

- 최초 기동 시 스키마 생성 및 MITRE ATT&CK 기본 데이터 적재로 수 분 소요
- 기동 확인: 로그에서 platform ready 메시지, 모든 컨테이너 `healthy`/`running`

---

## 6. 웹 UI 접속 (SSM 포트포워딩)

로컬 PC 사전 준비: AWS CLI v2 + Session Manager Plugin

```bash
aws ssm start-session --target i-xxxxxxxxxxxx \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["8080"],"localPortNumber":["8080"]}'
```

브라우저: `http://localhost:8080` → `.env`의 관리자 계정으로 로그인

> ⚠️ Docker가 publish한 포트는 UFW 규칙을 우회함. 호스트 방화벽이 아닌 **Security Group으로만 통제**하고 8080을 0.0.0.0/0으로 열지 말 것.

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

| 증상 | 조치 |
|---|---|
| ES 컨테이너 즉시 종료 | `vm.max_map_count` 설정 확인 |
| Platform OOM | `NODE_OPTIONS=--max-old-space-size=8096` 조정, 인스턴스 상향 |
| 플랫폼 기동 실패 | `.env` 누락 변수, `OPENCTI_ADMIN_TOKEN` UUID 형식 확인 |
| 적재 지연 | RabbitMQ 큐 적체 확인, worker replica 증설 |
| 비용 | 미사용 시 인스턴스 Stop (EBS 비용만 발생) |

---

## 참고

- OpenCTI 공식 설치 가이드: https://docs.opencti.io/latest/deployment/installation/
- OpenCTI 아키텍처/요구사항: https://docs.opencti.io/latest/deployment/overview/
- OpenCTI 설정 파라미터: https://docs.opencti.io/latest/deployment/configuration/
- OpenCTI-Platform/docker 저장소: https://github.com/OpenCTI-Platform/docker
- docker 저장소 XTM One 추가 PR #578: https://github.com/OpenCTI-Platform/docker/pull/578
- OpenCTI Deployment Toolkit (커뮤니티): https://github.com/404SecNotFound/OpenCTI-Deployment-Toolkit
- AWS 블로그 – SSM Session Manager 포트포워딩: https://aws.amazon.com/blogs/aws/new-port-forwarding-using-aws-system-manager-sessions-manager

---

## 변경 이력

| 버전 | 일자 | 내용 |
|---|---|---|
| v1.0 | 2026-10-26 | 최초 작성 |
