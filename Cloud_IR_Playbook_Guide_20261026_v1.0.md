# 클라우드 침해사고대응(IR) 실무 가이드

> 소규모 CERT가 클라우드 중심 환경에서 침해사고 대응 체계를 처음부터 세울 때 참고하기 위한 문서다.
> 공개 표준(NIST, FIRST)과 글로벌 IR 기업의 공개 자료를 근거로 정리했다.

| 항목 | 내용 |
|---|---|
| 버전 | v1.0 |
| 작성일 | 2026-10-26 |
| 대상 | SOC 내 소규모(1~2인) CERT, MSSP/MDR 환경의 IR 담당자 |
| 근거 | NIST SP 800-61r3, FIRST CSIRT Services Framework v2.1, AWS·Mandiant·Unit 42·CrowdStrike·Microsoft 공개 자료 |
| 주의 | 각 기업의 내부 SOP 전문은 공개되어 있지 않다. 이 문서는 공개 자료에 나온 범위만 다룬다 |

## 목차

1. CERT 역할 정의
2. 온프레미스 IR과 클라우드 IR의 차이
3. 사고 리딩 가이드
4. 글로벌 IR 기업 업무절차 벤치마킹
5. 표준 절차와 결과물
6. 우선 플레이북
7. MSSP 환경 고려사항
8. 실행 로드맵
9. 참고 자료

---

## 1. CERT 역할 정의

CERT, CSIRT, CIRT는 업계에서 사실상 같은 뜻으로 쓰인다. 모두 침해대응에 집중하는 팀을 가리키며, SOC는 보안 운영 전반을 다루는 더 넓은 개념이다.

| 구분 | SOC | CERT(CSIRT) |
|---|---|---|
| 성격 | 24시간 상시 모니터링 | 사고 발생 시 가동 (incident-driven) |
| 핵심 업무 | 탐지, 1차 분류(Triage), 에스컬레이션 | 사고 분석, 아티팩트·포렌식 분석, 봉쇄·복구 조정, 사후활동 |
| 산출물 | 이벤트 티켓, 관제 리포트 | 침해사고 분석 보고서, 플레이북, 재발방지 권고 |

- 정식 CSIRT가 없으면 SOC가 IR까지 맡는다. CSIRT가 있으면 SOC는 대응에 필요한 정보 수집을 지원한다.
- FIRST CSIRT Services Framework v2.1에서 CERT의 핵심 서비스는 다음과 같다.
  - 6.2 Information Security Incident Analysis
  - 6.3 Artifact and Forensic Evidence Analysis (표면 분석, 리버스 엔지니어링, 동적 분석, 비교 분석)

## 2. 온프레미스 IR과 클라우드 IR의 차이

| 항목 | 온프레미스 | 클라우드 |
|---|---|---|
| 현장 대응 | 직접 방문, 디스크·메모리 수집 | 방문 없음. API와 로그로 수집 |
| 주요 증거 | 악성코드, 이벤트로그, 아티팩트 | CloudTrail / Audit Log, IAM, VPC Flow Logs, GuardDuty |
| 초기 침투 | 취약점, 피싱, 악성코드 | **탈취되었거나 권한이 과도한 자격증명(Identity)** |
| 조사 초점 | 호스트 타임라인 | 신원, 설정 오류, 서비스 간 상호작용 |
| 봉쇄 | 네트워크 격리, 호스트 차단 | 키 비활성화, 권한 회수, SG/SCP 적용, 스냅샷 보존 |
| 증거 휘발성 | 상대적으로 낮음 | 높음 (리소스 삭제, 로그 보존기간 만료) |

Unit 42에 따르면 클라우드 침해의 대부분은 탈취되었거나 권한이 과도한 계정에서 시작한다. 그래서 조사는 신원, 설정 오류, 서비스 간 상호작용을 중심으로 진행된다.

---

## 3. 사고 리딩 가이드

### 3.1 전제

- 사고 리딩은 **기술이 아니라 조정(Coordination)** 의 문제다. PagerDuty는 비엔지니어도 Incident Commander(IC)로 훈련시킨다.
- 체계 없이 대응하는 방식에서 벗어나는 방법은 경험을 더 쌓는 것이 아니라 **절차, 기록, 역할을 고정하는 것**이다.

### 3.2 리딩 3원칙 (3Cs)

| 원칙 | 의미 | 실무 행동 |
|---|---|---|
| Coordinate (조정) | 누가 무엇을 하는지 정한다 | 이름 지정, 기한 지정, 확인 받기 |
| Communicate (소통) | 같은 정보를 같은 시점에 공유한다 | 정해진 주기로 상황 공유 |
| Control (통제) | 독단적인 조치를 막는다 | 모든 변경은 IC 승인 후 실행 |

Google SRE의 기본 원칙도 같다. 명확한 지휘 체계, 명확한 역할, 진행 중 작업 기록, 이른 사고 선언.

### 3.3 소규모 CERT 역할 구조

혼자서 모든 역할을 맡지 않는다. **IC는 CERT가 맡고, 나머지 역할은 빌려 온다.**

| 역할 | 담당 예시 | 하는 일 |
|---|---|---|
| IC (Incident Commander) | CERT | 판단, 지시, 우선순위 결정, 종료 선언 |
| Ops Lead (분석·조치) | SOC 분석가, 고객사 엔지니어 | 로그 분석, 차단, 격리 실행 |
| Comms Lead (소통) | SOC 팀장, 고객 담당 매니저 | 고객·경영진 보고 창구 |
| Scribe (기록) | SOC 인원 1명 | 타임라인, 결정사항 기록 |

- 소규모 사고에서는 Comms 역할을 IC가 겸한다. Google SRE도 사고 규모가 작으면 CL 역할을 IC에 흡수할 수 있다고 본다.
- PagerDuty는 IC가 직접 조치하거나 로그를 보지 말라고 권고한다. 소규모 CERT는 현실적으로 분석도 해야 하므로, 분석에 들어갈 때 "지금부터 10분간 분석, 그동안 IC는 OO님 대행"처럼 **명시적으로 역할을 전환**한다.

### 3.4 심각도 기준 (예시)

| 등급 | 기준 | 예시 | 목표 착수 | 공유 주기 |
|---|---|---|---|---|
| SEV1 | 침해 확정, 데이터 유출 또는 서비스 영향 진행 중 | 랜섬웨어, 루트 계정 탈취, 대량 유출 | 즉시 | 30분 |
| SEV2 | 침해 확정, 영향 범위 미확정 | 액세스 키 탈취, 웹셸 업로드 | 1시간 이내 | 2시간 |
| SEV3 | 침해 의심, 추가 확인 필요 | 비정상 로그인, 이상 API 호출 | 당일 | 일 1회 |
| SEV4 | 정책 위반 또는 경미 | 스캐닝, 차단된 공격 시도 | SOC 처리 | 주간 보고 |

> 등급은 언제든 올리고 내릴 수 있다. 확실하지 않으면 **높은 쪽으로 선언**한다. 조직 SLA에 맞춰 조정한다.

### 3.5 사고 선언 기준

다음 중 하나라도 해당하면 SOC 이벤트를 **CERT 사고로 선언**한다.

- 공격이 차단되지 않고 성공한 정황이 있다 (응답코드 200, 세션 생성, 프로세스 실행 등)
- 계정, 자격증명, 키가 비정상적으로 사용되었다
- 고객 또는 현업이 직접 침해를 신고했다
- 동일 IoC가 2개 이상 조직(고객사)에서 동시에 탐지되었다
- SOC 분석가가 30분 안에 오탐 여부를 판단하지 못한다

### 3.6 첫 60분 체크리스트

**0~10분: 선언**
- [ ] 사고 ID 발번 (예: `IR-YYYYMMDD-NNN`)
- [ ] 심각도 1차 판정
- [ ] **"이 사고는 제가 IC 맡겠습니다"** 를 명시적으로 선언
- [ ] 소통 채널 개설 (전용 메신저방 또는 콜)
- [ ] Scribe 지정, 타임라인 기록 시작

**10~30분: 파악**
- [ ] 무엇이, 언제, 어디서 탐지되었는가 (탐지 소스, 원본 알림)
- [ ] 영향받은 조직, 자산, 계정 식별
- [ ] **증거 보존 지시**: 로그 export, 스냅샷, EDR 수집 (클라우드는 휘발성이 높으므로 최우선)
- [ ] 자산 소유자 1차 연락

**30~60분: 결정**
- [ ] 봉쇄 여부와 방법 결정 (실행 주체 명시)
- [ ] 1차 상황공유 발송 (3.8 템플릿)
- [ ] 다음 공유 시각 확정
- [ ] 추가 인력이 필요한지 판단 (외부 IR 업체, 벤더 지원 등)

### 3.7 리딩 화법

| 상황 | 이렇게 말한다 |
|---|---|
| 시작 | "IR-001 선언합니다. IC는 저, 기록은 OO님입니다." |
| 지시 | "OO님, CloudTrail에서 해당 키의 최근 24시간 호출 내역을 15분 안에 뽑아 주세요." |
| 확인 | "방금 지시 내용 다시 한 번 말씀해 주시겠어요?" |
| 정리 | "현재 확인된 것은 A, B입니다. 아직 모르는 것은 C입니다. 다음 확인은 D입니다." |
| 독단 조치 차단 | "조치하기 전에 저한테 먼저 말씀해 주세요. 증거가 날아갈 수 있습니다." |
| 모를 때 | "그 부분은 아직 판단 근거가 없습니다. OO까지 확인한 뒤 결정하겠습니다." |

> **"모른다"고 말하는 것은 리딩 실패가 아니다.** 모르는 것을 목록으로 관리하는 것이 리딩이다.

### 3.8 상황공유 템플릿

```
[IR-YYYYMMDD-NNN] 상황공유 #N  (YYYY-MM-DD HH:MM KST)
- 심각도: SEV2
- 현재 상태: 분석 중 / 봉쇄 완료 / 복구 중 / 종료
- 확인된 사실:
  1.
  2.
- 영향 범위: (조직 / 자산 / 계정 / 데이터)
- 조치 내역: (실행 주체 표기 — CERT 권고 / 자산 소유자 실행)
- 미확인 사항:
- 요청사항:
- 다음 공유: HH:MM
- IC:
```

### 3.9 의사결정 로그 템플릿

나중에 "왜 그때 그렇게 했냐"는 질문에 답할 수 있는 유일한 근거다.

| 시각(KST) | 결정 내용 | 근거 | 결정자 | 실행자 | 비고 |
|---|---|---|---|---|---|
| 14:20 | 액세스 키 비활성화 권고 | 비정상 리전 호출 확인 | IC | 자산 소유자 | 승인 메일 수신 |

### 3.10 인계 (Handoff)

- [ ] 현재 상황공유 최신본 전달
- [ ] 진행 중인 작업과 담당자 목록 전달
- [ ] 미해결 결정사항 전달
- [ ] **"지금부터 IC는 OO님입니다"** 를 채널에 명시하고 상대의 확인을 받는다

### 3.11 체계 없는 대응 패턴과 바꿀 행동

| 패턴 | 바꿀 행동 |
|---|---|
| 확실해질 때까지 선언을 미룬다 | 의심되면 먼저 선언하고 나중에 내린다 |
| 모두가 동시에 로그를 본다 | IC가 범위를 나눠 배정한다 |
| 기록은 끝나고 쓴다 | Scribe가 실시간으로 기록한다 |
| 조치부터 하고 증거는 나중에 | 증거 보존을 먼저 지시하고 조치는 그 다음 |
| 보고서는 기억으로 쓴다 | 타임라인과 결정 로그로 쓴다 |
| 끝나면 잊는다 | 사후검토를 하고 플레이북에 반영한다 |

---

## 4. 글로벌 IR 기업 업무절차 벤치마킹

### 4.1 한눈에 비교

| 기업 | 절차 프레임 | 핵심 도구 | 종결·복구 방식 | 결과물 |
|---|---|---|---|---|
| Mandiant (Google) | 기술 배치 → 초기 단서 조사 → … → 복구(Remediation) | 자사 EDR·네트워크 센서, FLARE 리버싱 | Posturing → Remediation Event → Strategic | 경영진 보고서, 조사 보고서, 복구 보고서 |
| Unit 42 (Palo Alto) | Scope → Investigate → Secure | Cortex XDR (+ MDR 전환) | 봉쇄 → 하드닝 → 24/7 모니터링 전환 | 사고 보고, IoC, 하드닝 권고 |
| CrowdStrike Services | 인텔리전스 기반 신속 복구 | Falcon EDR, RTR, Falcon Forensics, Next-Gen SIEM, Identity | RTR로 원격 제거·복구 | 조사 결과, 맞춤 복구 계획 |
| Microsoft IR (구 DART) | Investigation → Containment → Recovery → Modernization | Defender XDR 계열, Sentinel, Defender for Identity | 컨트롤 플레인 침해 시 Compromise Recovery(CR) | 조사 결과를 CR 입력값으로 제공 |
| AWS | Detection → Analysis → Containment → Eradication → Recovery | GuardDuty, Security Hub, Step Functions, 포렌식 AMI | 조치 후 분석 단계로 되돌아가 효과 검증 | 자동 포렌식 리포트, SNS 통보 |

### 4.2 Mandiant (Google Cloud)

**절차**
- 1단계는 "기술 배치와 초기 단서 조사"다. 가장 적합한 도구를 먼저 배치한 뒤 조사를 시작한다.
- 호스트, 네트워크, 이벤트 기반 분석을 병행해 환경 전체를 평가한다.
- 마지막 단계에서 공격자 행위와 비즈니스 요구에 맞춘 봉쇄·복구 전략을 수립한다.

**도구·조직**
- FLARE 리버스 엔지니어가 악성코드를 분석하고, 맞춤 디코더와 파서를 작성해 공격자 역량과 TTP를 파악한다.
- 조사·복구 기간 중 Managed Defense가 24시간 공격자 활동을 분석한다.

**종결 방식: Remediation Event 모델** (Mandiant 발표자료, Black Hat 2012)
1. **Posturing**: 공격자가 눈치채지 않을 조치만 먼저 적용하고, 환경을 "조사 가능한 상태"로 계측한다
2. **Remediation Event**: 정해진 짧은 시간 안에 한꺼번에 공격자를 퇴출하고 공격 기법을 무력화한다
3. **Strategic**: 당장 필요하지는 않지만 장기적으로 필요한 보안 개선

**결과물 3종**: 제3자 검증에도 견딜 수준을 지향한다.

| 보고서 | 내용 |
|---|---|
| Executive | 경영진용 요약 |
| Investigative | 영향받은 컴퓨터, 위치, 계정, 유출되었거나 유출 위험이 있는 정보 목록 |
| Remediation | 수행한 봉쇄 조치, 전략적 권고 |

### 4.3 Unit 42 (Palo Alto Networks)

- **절차**: Scope(범위·심각도·성격 평가) → Investigate(증거 수집·분석으로 IoC·TTP 식별) → Secure(봉쇄·복구·제거)
- **도구**: Cortex XDR 중심. 엔드포인트, 네트워크, 클라우드, ID 텔레메트리를 통합한다.
- **사례**: 랜섬웨어 대응에서 Cortex XDR을 하루 안에 3.5만 대 이상에 배포했고, 배포 후 수 시간 안에 최초 감염 단말(P0)과 침투 경로를 추적했다.
- **종결**: 진입 경로 봉쇄 → 시스템 하드닝 → IoC 차단 → 24/7 MDR 모니터링 전환

### 4.4 CrowdStrike Services

- **절차**: 인텔리전스 기반 신속 복구. 위협 행위자가 한 행위를 파악한 뒤 그것을 정밀하게 되돌리고 제거한다. Falcon 외에 기존 보안도구의 과거 텔레메트리도 활용한다(기술 중립적 접근).

| 도구 | 용도 |
|---|---|
| Falcon EDR | 실시간 가시성 |
| Real Time Response (RTR) | 원격 프로세스 종료, 파일 삭제, 복구 스크립트 실행 |
| Falcon Forensics | 원격·대규모 포렌식 트리아지 데이터 수집 |
| Falcon Next-Gen SIEM | 로그 수집·분석, 근본원인 분석 |
| Falcon Identity Threat Protection | 거의 모든 IR 조사에 활용 |

### 4.5 Microsoft Incident Response (구 DART)

- **절차**: Investigation → Containment → Recovery → Modernization. 랜섬웨어 대응은 현재 상황 평가로 범위를 파악하는 데서 시작하고, 조사와 복구를 병행하면서 근본 원인을 끝까지 규명한다.
- **도구**: 도착하면 가장 먼저 탐지 역량을 강화한다. 대응한 거의 모든 사고에 자격증명 탈취가 있었기 때문에 ID 위협 탐지 도구(현 Defender for Identity)를 가장 먼저 배치한다.
- **종결**: 컨트롤 플레인(주로 Active Directory)이 침해되었다면 별도의 Compromise Recovery(CR) 단계로 넘어간다. 조사 결과물은 CR에 바로 투입될 수 있도록 작성한다.

### 4.6 AWS

- **절차**: Detection → Analysis → Containment → Eradication. 봉쇄와 제거 후에는 다시 분석으로 돌아가 조치 효과를 확인한다.
- **도구: Automated Forensics Orchestrator for EC2/EKS**
  1. GuardDuty·Config 탐지 → Security Hub 집계 → EventBridge 트리거
  2. Step Functions가 메모리·디스크를 수집한 뒤 Security Group을 바꿔 격리한다
  3. 포렌식 AMI로 분석하고 결과를 S3·DynamoDB에 저장한 뒤 SNS로 통보한다
  - 증거 수집·보관을 자동화해 Chain of Custody를 유지한다.

### 4.7 공통 패턴

| # | 패턴 | 근거 |
|---|---|---|
| 1 | **가시성을 먼저 확보한다** | Mandiant 1단계가 기술 배치, Unit 42는 XDR 즉시 배포, Microsoft는 탐지 역량 강화가 최우선 |
| 2 | **범위를 확정한 뒤 제거한다** | Mandiant Posturing: 공격자가 눈치채지 않게 준비한 뒤 한 번에 퇴출 |
| 3 | **ID 중심으로 조사한다** | Microsoft: 거의 모든 사고에 자격증명 탈취. CrowdStrike: ID 모듈을 거의 모든 조사에 사용 |
| 4 | **조치 후 효과를 검증한다** | AWS: 분석 단계로 회귀. Unit 42: MDR 모니터링 전환 |
| 5 | **결과물을 독자별로 분리한다** | Mandiant: 경영진, 조사, 복구 보고서 3종 |

---

## 5. 표준 절차와 결과물

### 5.1 표준 절차 (NIST 800-61r3 정렬, 글로벌 패턴 반영)

| 단계 | 내용 | 활용 예시 |
|---|---|---|
| 1. 가시성 확보 | 조사에 필요한 데이터 접근을 먼저 확보 | SIEM 검색, EDR 콘솔, Velociraptor 등 수집 도구 배포 |
| 2. 범위 확정 | 조직·자산·계정·데이터 범위 확정, ATT&CK 매핑 | 침투 경로 가설 수립 |
| 3. 조사 | 호스트·네트워크·클라우드 로그 교차 분석 | WAF/IPS 로그, CloudTrail, EDR 텔레메트리 |
| 4. 복구 이벤트 | 조치 목록을 확정하고 정해진 시간에 일괄 실행 | Posturing → Remediation Event |
| 5. 검증 | 동일 IoC·TTP 재헌팅, 탐지룰 추가 | 조치 효과 확인 |
| 6. 종결 | 결과물 전달, 사후검토 | 5.3, 5.4 |

### 5.2 IR 보고서 템플릿 골격

1. **개요**: 사고 ID, 대상 조직, 탐지 일시, 심각도, 담당자, 현재 상태
2. **요약 (Executive Summary)**: 무슨 일이 있었고, 영향은 무엇이며, 지금 상태가 어떤지
3. **탐지 경위**: 탐지 소스(SIEM 룰, WAF/IPS, EDR, 신고), 최초 알림 내용
4. **타임라인**: UTC/KST를 병기한 시간순 이벤트
5. **분석**: 초기 침투 경로, 침해 범위(계정, 리소스, 리전, 데이터), 공격자 행위(ATT&CK 매핑), IoC 목록
6. **대응 조치**: 봉쇄, 제거, 복구 각각의 **실행 주체** 표기
7. **근본 원인 및 재발방지 권고**
8. **부록**: 쿼리, 원본 로그 발췌, 증거 보존 위치(해시 포함)

### 5.3 결과물 표준 (Mandiant 3종 차용)

| 결과물 | 독자 | 분량 | 핵심 내용 |
|---|---|---|---|
| 경영진 요약 | 경영진 | 1~2쪽 | 무슨 일이 있었고, 영향은 무엇이며, 조치 상태와 남은 위험은 무엇인가 |
| 조사 보고서 | 보안담당자 | 제한 없음 | 타임라인, 침투 경로, 영향 자산·계정 목록, IoC, ATT&CK 매핑 |
| 복구 보고서 | IT·인프라 담당 | 체크리스트 형식 | 수행한 봉쇄 조치(실행 주체 표기), 단기·장기 권고 |
| (내부) 탐지 개선 | SOC | 룰 단위 | 신규·수정 탐지룰, 헌팅 쿼리 |

### 5.4 종결 기준

- [ ] 침투 경로가 식별되었고 차단되었다
- [ ] 영향 범위(자산, 계정, 데이터)가 확정되었다
- [ ] 공격자의 지속성(백도어, 신규 계정, 키)이 제거되었다
- [ ] 조치 후 재헌팅에서 동일 IoC·TTP가 발견되지 않았다
- [ ] 자산 소유자가 복구 완료를 확인했다
- [ ] 결과물을 전달했고 탐지룰 개선을 반영했다

### 5.5 사후검토 (종결 후 5영업일 이내)

- **비난하지 않는 원칙(Blameless)**: 사람이 아니라 프로세스의 빈틈을 찾는다
- 확인 질문
  1. 탐지는 적시에 되었는가?
  2. 선언이 늦어진 이유가 있는가?
  3. 증거 보존에 실패한 것이 있는가?
  4. 소통에 혼선이 있었는가?
- 산출물: 플레이북 수정사항, 탐지룰 개선사항

---

## 6. 우선 플레이북

| 플레이북 | 주요 트리거 | 핵심 증거 | 핵심 봉쇄 조치 |
|---|---|---|---|
| 자격증명 탈취 | 비정상 지역 로그인, 액세스 키 이상 사용 | CloudTrail, IAM Access Advisor | 키 비활성화, 세션 무효화, 권한 회수 |
| 크립토마이닝 | 비정상 인스턴스 대량 생성, GPU 사용 급증 | CloudTrail(RunInstances), 과금, GuardDuty | 인스턴스 격리, 스냅샷 보존 후 종료 |
| 데이터 유출 | S3 대량 GetObject, 퍼블릭 정책 변경 | S3 Data Events, VPC Flow Logs | 버킷 정책 원복, 관련 자격증명 차단 |

- 각 플레이북은 NIST 단계(준비, 탐지·분석, 봉쇄·제거·복구, 사후활동)로 작성한다.
- AWS 공식 샘플 플레이북(aws-incident-response-playbooks)을 번역·수정해 시작하면 빠르다.

---

## 7. MSSP 환경 고려사항

- **실행 권한 분리**: MSSP에서 봉쇄·복구 권한은 고객사에 있다. 플레이북과 보고서에 "CERT 권고"와 "고객 실행" 칸을 분리한다.
- **사전 합의**: 비상연락망, 긴급 조치 승인 절차, 로그 접근 범위를 계약이나 SLA에 명시한다.
- **장비 이질성**: 고객사마다 연동 장비가 다르므로, 플레이북의 증거 수집 단계에 "고객사별 가용 소스" 체크리스트를 둔다.
- **엔드포인트 가시성**: EDR이 SIEM에 연동되어 있지 않다면 고객 EDR 콘솔에 직접 접근하거나 Velociraptor 같은 수집 도구를 쓴다. 오픈소스 도구는 상업적 서비스 제공 시 라이선스 조건을 확인한다.

---

## 8. 실행 로드맵

| 단계 | 작업 | 산출물 | 참고 |
|---|---|---|---|
| 1 | IR 보고서 템플릿 작성 | 5.2 기반 템플릿 | NIST SP 800-61r3 |
| 2 | 우선 플레이북 3종 작성 | 6장 | aws-incident-response-playbooks |
| 3 | 레퍼런스 보고서 0호 생성 | Stratus Red Team 모의 침해를 템플릿으로 보고서화 | Stratus Red Team |
| 4 | 실사고 사례 벤치마킹 | 보고서 구성·분석 방식 차용 | AWS 실사고 기반 IR 가이드(2026.09), Unit 42 |
| 5 | 테이블탑 훈련 | 역할 확정, 플레이북 보완 | AWS Game Day 권고 |

**리딩 역량 쌓기**

| 주기 | 활동 | 목적 |
|---|---|---|
| 월 1회 | SOC와 테이블탑 (30분, 시나리오 1개) | IC 역할 반복 연습 |
| 분기 1회 | 모의 침해 → 보고서 작성 | 레퍼런스 보고서 축적 |
| 사고마다 | 사후검토 → 본 문서 개정 | 내부 기준 고도화 |
| 상시 | 공개 IR 보고서 주 1편 정독 | 보고서 작성 방식 벤치마킹 |

---

## 9. 참고 자료

**표준·프레임워크**
- NIST SP 800-61r3: https://csrc.nist.gov/pubs/sp/800/61/r3/final
- FIRST CSIRT Services Framework v2.1: https://www.first.org/standards/frameworks/csirts/csirt_services_framework_v2.1
- CISA Incident & Vulnerability Response Playbooks: https://www.cisa.gov/news-events/news/incident-and-vulnerability-response-playbooks

**SOC vs CSIRT**
- TechTarget – CERT vs CSIRT vs SOC: https://www.techtarget.com/searchsecurity/tip/CERT-vs-CSIRT-vs-SOC-Whats-the-difference
- Rapid7 – SOC vs CSIRT: https://www.rapid7.com/blog/post/2017/04/19/difference-between-a-soc-and-a-csirt/
- Exabeam – SOC vs CSIRT: https://www.exabeam.com/explainers/information-security/soc-vs-csirt-6-key-differences-and-which-organizations-need-both/

**사고 리딩**
- PagerDuty Incident Response Docs: https://response.pagerduty.com
- PagerDuty – Different Roles: https://response.pagerduty.com/before/different_roles/
- Google SRE Book – Managing Incidents: https://sre.google/sre-book/managing-incidents/
- Google SRE Workbook – Incident Response: https://sre.google/workbook/incident-response/
- Google Incident Management Guide: https://sre.google/static/pdf/IncidentManagementGuide.pdf

**클라우드 IR**
- AWS Security Incident Response Technical Guide: https://docs.aws.amazon.com/security-ir/latest/userguide/security-incident-response-guide.html
- AWS Security IR – Operations: https://docs.aws.amazon.com/security-ir/latest/userguide/operations.html
- aws-samples/aws-incident-response-playbooks: https://github.com/aws-samples/aws-incident-response-playbooks
- aws-samples/aws-customer-playbook-framework: https://github.com/aws-samples/aws-customer-playbook-framework
- AWS Automated Forensics Orchestrator: https://docs.aws.amazon.com/solutions/latest/automated-forensics-orchestrator-for-amazon-ec2/architecture-overview.html
- Unit 42 – Responding to Cloud Incidents: https://unit42.paloaltonetworks.com/responding-to-cloud-incidents/
- Invictus-AWS: https://invictus-aws.readthedocs.io

**글로벌 IR 기업**
- Mandiant IR Service 데이터시트: https://services.google.com/fh/files/misc/mandiant_incident_response_service_january_2024.pdf
- Mandiant – Remediating Targeted Attacks (Black Hat 2012): https://media.blackhat.com/ad-12/Aldridge/bh-ad-12-remediating-Aldridge-Slides.pdf
- Unit 42 Incident Response: https://www.paloaltonetworks.com/unit42/incident-response
- Unit 42 랜섬웨어 대응 사례: https://www.paloaltonetworks.com/customers/global-healthcare-company-swiftly-recovers-from-ransomware
- Unit 42 Global IR Report 2026: https://www.paloaltonetworks.com/resources/research/unit-42-incident-response-report
- CrowdStrike – Redefining Incident Response: https://www.crowdstrike.com/en-us/blog/redefining-incident-response-in-the-modern-era/
- Microsoft DART TTP Part 1: https://www.microsoft.com/security/blog/2019/11/14/security-incident-response-utilizing-cloud-dart-tools-techniques-procedures-part-1/
- Microsoft DART Ransomware Approach: https://learn.microsoft.com/en-us/security/ransomware/incident-response-playbook-dart-ransomware-approach

---

## 변경 이력

| 버전 | 일자 | 내용 |
|---|---|---|
| v1.0 | 2026-10-26 | 착수 가이드, 리딩 가이드, 글로벌 벤치마킹 문서 통합. 공개용으로 조직·개인 식별 정보 제거 |
