# 로그 수집 중단 알림 → Google SecOps 얼럿 연동 절차

- 작성일: 2026-10-23
- 목적: Cloud Monitoring의 "로그 수집 중단" 알림을 관제 화면(Google SecOps)에 얼럿으로 띄움
- 흐름: Cloud Monitoring 알림 정책 → 웹훅 알림 채널 → SecOps 웹훅 피드 → 커스텀 파서(UDM) → YARA-L 룰 → 얼럿(SOAR 사용 시 케이스)
- 백업: 동일 알림 정책에 이메일 채널 병행 (SecOps 수집 경로 장애 시 얼럿 유실 방지)

---

## 0. 사전 준비

| 항목 | 내용 |
|------|------|
| GCP 권한 | API 키 생성 권한, Cloud Monitoring 알림 정책/채널 편집 권한 |
| SecOps 권한 | 피드 생성, 파서 편집, 룰 생성/Alerting 설정 권한 |
| 알림 정책 점검 | 데이터 부재(metric absence) 조건 포함 여부 — 수집이 완전히 멈추면 0이 아니라 데이터 자체가 사라짐 |
| 임계 시간 | 소스별 로그 발생 주기 고려 (야간 저빈도 소스는 짧게 잡으면 오탐) |

---

## 1. SecOps 웹훅 피드 생성

1. SecOps > **Settings > Feeds > Add new**
2. Feed name: `gcp-monitoring-ingestion-alert` (예시)
3. Source type: **Webhook**
4. Log type: 커스텀 로그 타입 지정 (Cloud Monitoring 인시던트용 기본 파서는 없을 가능성이 높음 — 목록에서 먼저 확인)
5. Split delimiter: 비워둠 (요청 1건 = 이벤트 1건)
6. 생성 후 **Generate Secret Key** → 시크릿 키 복사·보관 (재조회 불가, 재생성 시 이전 키 무효)
7. Details 탭에서 **Endpoint URL** 복사

### API 키 발급

1. GCP 콘솔 > **APIs & Services > Credentials > Create credentials > API key**
2. 이름: `secops-webhook-cloud-monitoring` (웹훅 전용 키로 분리)
3. **API restrictions > Restrict key > Chronicle API** 만 허용

---

## 2. Cloud Monitoring 웹훅 알림 채널 등록

Cloud Monitoring 웹훅 채널은 커스텀 헤더를 지원하지 않으므로 쿼리 파라미터 인증 사용.

1. Cloud Monitoring > **Alerting > Edit notification channels > Webhooks > Add new**
2. Endpoint URL:
   ```
   <ENDPOINT_URL>?key=<API_KEY>&secret=<SECRET>
   ```
3. Display name: `SecOps Webhook`
4. **Test connection**으로 전송 확인
5. 대상 알림 정책 편집 > Notifications에 **SecOps Webhook + 이메일 채널** 둘 다 추가

> URL에 키가 포함되므로 채널 설정 화면 접근 권한을 최소화할 것.

---

## 3. 커스텀 파서 (UDM 매핑)

YARA-L 룰은 UDM 이벤트에만 동작하므로 파서 필수. 미적용 시 원본 로그 검색으로만 조회됨.

### 매핑 표

| Cloud Monitoring 필드 | UDM 필드 |
|----------------------|----------|
| (고정) | `metadata.event_type` = `GENERIC_EVENT` |
| (고정) | `metadata.vendor_name` = `Google Cloud` |
| (고정) | `metadata.product_name` = `Cloud Monitoring` |
| `incident.incident_id` | `metadata.product_log_id` |
| `incident.state` (open/closed) | `security_result.action_details` |
| `incident.policy_name` | `security_result.rule_name` |
| `incident.summary` | `security_result.summary` |

> `STATUS_UPDATE`는 principal 장비 정보가 필수라 검증 실패 가능 → `GENERIC_EVENT` 사용.

### 파서 초안

```
filter {
  mutate {
    replace => {
      "incident.incident_id" => ""
      "incident.state" => ""
      "incident.policy_name" => ""
      "incident.summary" => ""
    }
  }
  json {
    source => "message"
    array_function => "split_columns"
    on_error => "not_json"
  }

  mutate {
    replace => {
      "event.idm.read_only_udm.metadata.event_type" => "GENERIC_EVENT"
      "event.idm.read_only_udm.metadata.vendor_name" => "Google Cloud"
      "event.idm.read_only_udm.metadata.product_name" => "Cloud Monitoring"
      "event.idm.read_only_udm.metadata.product_log_id" => "%{incident.incident_id}"
      "sr.action_details" => "%{incident.state}"
      "sr.rule_name" => "%{incident.policy_name}"
      "sr.summary" => "%{incident.summary}"
    }
  }
  mutate {
    merge => { "event.idm.read_only_udm.security_result" => "sr" }
  }
  mutate {
    merge => { "@output" => "event" }
  }
}
```

### 적용 절차

1. 2단계의 Test connection 또는 실제 알림으로 샘플 페이로드 확보 (원본 로그 검색에서 확인)
2. SecOps > **Settings > Parsers** > 해당 로그 타입 > Create custom parser
3. 샘플 로그로 **Preview** → UDM 출력 확인 후 Submit

> 위 파서는 초안. 실제 페이로드 구조(버전별 필드)로 반드시 검증 후 적용.

---

## 4. YARA-L 룰 등록 및 Alerting 활성화

```
rule log_ingestion_stopped {
  meta:
    description = "Cloud Monitoring 로그 수집 중단 알림"
    severity = "HIGH"
  events:
    $e.metadata.log_type = "<커스텀 로그 타입>"
    $e.metadata.product_name = "Cloud Monitoring"
    $e.security_result.action_details = "open"
  condition:
    $e
}
```

1. SecOps > **Detection > Rules Editor** > 신규 룰 작성
2. 저장 후 **Live Rule 활성화 + Alerting 켜기** (Alerting 미설정 시 탐지만 되고 얼럿 화면에 안 뜸)
3. SOAR 사용 시 케이스 생성 연동 확인

- `open` 이벤트만 얼럿 생성, `closed`는 이력으로만 보관

---

## 5. 테스트 및 검증

| 단계 | 확인 방법 | 기대 결과 |
|------|----------|----------|
| 전송 | Cloud Monitoring 채널 Test connection | 오류 없음 |
| 수집 | SecOps 피드 상태, 원본 로그 검색 | 피드 Active, 로그 수신 |
| 파싱 | UDM 검색 `metadata.product_name = "Cloud Monitoring"` | 매핑 필드 정상 |
| 탐지 | 임계 시간 짧은 테스트 알림 정책으로 실제 open 발생 | 얼럿 생성 |
| 해제 | 테스트 정책 해소(closed) | 얼럿 미생성, 이벤트만 수집 |
| 백업 | 이메일 수신 | 동일 알림 수신 |

테스트 종료 후 테스트용 알림 정책 삭제.

---

## 6. 운영 시 주의사항

- **순환 문제**: SecOps 수집 경로 자체 장애 시 이 얼럿도 뜨지 않음 → 이메일 백업 채널 유지 필수
- **키 관리**: 웹훅 전용 API 키 사용, Chronicle API로 제한, 시크릿 재생성 시 Cloud Monitoring 채널 URL 동시 갱신
- **중복**: 인시던트당 open/closed 2건 수집 — 얼럿은 open만
- **커버리지 보강**: 소스 측 지표 외에 SecOps 수집량 지표 기준 알림도 추가하면 중간 구간 장애까지 탐지 가능
