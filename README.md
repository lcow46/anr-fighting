# TA 보안 진단 HTML 보고서 정의 및 산출 프로세스 자동화

## 목표

Cloudforet 웹 콘솔에서 수동으로 xlsx export 하던 **AWS TrustedAdvisor 보안(Security) 점검 결과**를
REST API로 EST API로 자동 수집하고, AWS Account(Project) 단위의 HTML 보안 보고서를 자동 생성하여 S3에 일별/월별로 보관하는 Serverless 파이프라인을 구축한다.

## 🏗️ 전체 시스템 아키텍처

```text
[EventBridge Scheduler] (월간 1회 Cron 실행)
       │
       ▼ (Trigger)
[Lambda Function] (Memory: 512MB~1GB)
       │ ─── (API Key Header) ───► [SpaceONE API]
       │ ◄── (TA Security Data) ──────┘
       │
       ▼ (Account별 HTML 보고서 분할 저장)
[S3 Bucket] (경로: YYYY-MM-DD/trusted_advisor_report_{AccountName}_{YYYYMMDD}.html)
```

## 1. SpaceOne API 기본 구조

- HTTP Method: `POST` 고정
- 경로 규칙: `/<service>/<version>/<resource>/<method>`
- 인증방식: `Authorization: Bearer {CLOUDFORET_API_KEY}` 헤더
  - 관리자 권한으로 발급된 고정 API Key를 AWS Lambda 환경변수(CLOUDFORET_API_KEY)에 보관하여 호출 시 인증 처리

## 2. 데이터 수집 대상 및 조건

**엔드포인트**: `POST /inventory/cloud-service/list`

| key | operator | value | 의미 |
|---|---|---|---|
| `cloud_service_group` | `eq` | `TrustedAdvisor` | 대상 서비스 지정 |
| `cloud_service_type` | `eq` | `Check` | 대상 리소스 타입 지정 |
| `data.category` | `eq` | `security` | 콘솔 상 "Category" 컬럼, `data.` 접두어 필요 |
| `data.status` | `not` | `not_available` | 콘솔 상 "Status" 컬럼. `not_available` 제외 → ok/error/warning만 수집 |

```json
{
  "query": {
    "filter": [
      { "key": "cloud_service_group", "value": "TrustedAdvisor", "operator": "eq" },
      { "key": "cloud_service_type", "value": "Check", "operator": "eq" },
      { "key": "data.category", "value": "security", "operator": "eq" },
      { "key": "data.status", "value": "not_available", "operator": "not" }
    ]
  }
}
```

## 3. 페이지네이션

`query.page.start`는 **1부터 시작**. `limit`은 최대 1000 단위로 확인.

| page | start | limit | 반환 건수 |
|---|---|---|---|
| 1 | 1 | 1000 | 1000 |
| 2 | 1001 | 1000 | 1000 |
| 3 | 2001 | 1000 | 758 |

3개 페이지 합산 결과 `cloud_service_id` 기준 **중복 0건**, 총 2758건으로 `total_count`와 일치. 3회 순차 호출로
전체 데이터셋을 빠짐없이 안전하게 수집 가능함을 확인.

## 4. 고객사 ID → 이름 매핑 (Identity 연동)

`CloudService` 응답에는 `project_id`, `workspace_id`만 있고 사람이 읽을 이름은 없다. Identity 서비스에서 별도 조회해서 매핑해야 한다.

### 4-1. Project (82개)

**엔드포인트**: `POST /identity/project/list`
- 조회 방식: 수집된 TA Check 데이터의 `project_id` 목록을 추출한 후 in 연산자로 일괄 조회
- `project_id` = 어카운트 단위로 확인됨 (예: `롯데웰푸드-Dev`, `이랜드-Prd`, `한국타이어-Network` 등)로 구성
- 응답 필드: `project_id`, `name`, `project_type`, `project_group_id`, `workspace_id`, `created_at` 등

```json
{
  "query": {
    "filter": [
      { "key": "project_id", "value": ["project-...", "..."], "operator": "in" }
    ],
    "page": { "start": 1, "limit": 100 }
  }
}
```

### 4-2. Project Group (31개)

**엔드포인트**: `POST /identity/project-group/list`
- 조회 방식: 5-1과 동일하게 `project_group_id`를 `in`으로 일괄 조회
- `project_group_id` = 고객사 단위로 확인됨 (예: `(주)하림`, `HD현대일렉트릭`, `SK쉴더스` 등)
- 응답 필드: `project_group_id`, `name`, `parent_group_id`, `workspace_id` 등

### 4-3. Workspace (1개)

- 수집된 데이터셋 내 workspace_id는 단일 객체(workspace-262a7ab4df64)로 구성
- `project_id` 기반 매핑으로 보고서 생성이 가능하므로 참고용으로, 추가 권한 요구 조회를 제외하고 처리


## 5. 주요 구현 현황 및 자동화 산출물

**진행 현황**
1. Serverless 기반 무인 자동화 파이프라인 구축 완료
2. AWS Account(Project) 단위 HTML 리포트 분할 생성
3. S3 날짜별 디렉토리 보관 (YYYY-MM-DD/)

**이후 계획**
1. AWS Step Functions (MAP 분산 병렬 처리) 도입
2. Amazon Bedrock (Generative AI) 연동을 통한 자연어 요약

```text
[S3 원본 저장]
        │
        ▼
[AWS Step Functions (Map 병렬 처리)]
        │
        ▼
[Lambda (Account별 분석)]
        │
        ├──────────────► [Amazon Bedrock]
        │                    │
        │                    ├─ 위험도 평가
        │                    ├─ 보안 분석
        │                    └─ 조치 가이드 생성
        │
        ▼
[S3 최종 결과 저장 (HTML Report)]
```

