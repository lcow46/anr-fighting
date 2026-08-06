# TA 보안 진단 HTML 보고서 정의 및 산출 프로세스 자동화

## 목표

Cloudforet 웹 콘솔에서 수동으로 xlsx export 하던 **AWS TrustedAdvisor 보안(Security) 점검 결과**를
REST API로 직접 조회하여, 이후 HTML 리포트 자동 생성 파이프라인의 데이터 소스로 사용한다.

## 1. API 기본 구조

- 모든 엔드포인트는 `POST` 고정, 경로 규칙: `/<service>/<version>/<resource>/<method>`
- 예: `/inventory/v1/cloud-service/list`
- 인증: `Authorization: Bearer {token}` 헤더
  - 현재는 웹 콘솔 세션 토큰을 그대로 사용 중 (수명이 짧아 주기적 재발급 필요)
  - 서비스 계정/App 단위 장기 토큰 발급은 관리자 권한이 필요해 **보류** 중
- 사내망이 Zscaler로 SSL 인터셉트하는 환경이라, `curl`은 시스템 인증서로 통과되지만 Python
  `requests` 등에서는 별도 CA 번들(`ZscalerRootCertificate-2048-SHA256-Feb2025.crt`) 지정이 필요함

## 2. 대상 리소스 특정 과정

콘솔 화면 경로 `{도메인}/workspace/asset-inventory/cloud-service/aws/TrustedAdvisor/Check`는
프론트엔드 라우팅일 뿐, API 경로가 아니다. 이 경로의 마지막 3개 세그먼트는 실제로는
`CloudServiceType`의 `provider` / `group` / `name` 값이었다.

`/inventory/v1/cloud-service-type/list`로 확인한 결과:

| 필드 | 값 |
|---|---|
| `provider` | `aws` |
| `group` | `TrustedAdvisor` |
| `name` | `Check` |
| `cloud_service_type_key` | `aws.TrustedAdvisor.Check` |
| `cloud_service_type_id` | `cloud-svc-type-da29b4aac66c` |

**주의 — 리소스별로 필터 가능한 필드명이 다름:**

`CloudServiceType`에서 쓰던 필드명(`group`, `cloud_service_type_id`)을 실제 데이터 조회 엔드포인트인
`/inventory/v1/cloud-service/list`에 그대로 쓰면 `ERROR_DB_QUERY: Cannot resolve field` 에러가 난다.
`CloudService`(인스턴스) 리소스에서 유효한 필드명은 다음과 같이 확인됨:

| CloudServiceType 필드명 | ❌ | CloudService 필드명 | ✅ |
|---|---|---|---|
| `group` | 불가 | `cloud_service_group` | 가능 |
| `cloud_service_type_id` | 불가 | `cloud_service_type` (name 문자열) | 가능 |

## 3. 최종 조회 조건

**엔드포인트**: `POST /inventory/v1/cloud-service/list`

| key | operator | value | 의미 |
|---|---|---|---|
| `cloud_service_group` | `eq` | `TrustedAdvisor` | |
| `cloud_service_type` | `eq` | `Check` | |
| `data.category` | `eq` | `security` | 콘솔 상 "Category" 컬럼, `data.` 접두어 필요 |
| `data.status` | `not` | `not_available` | 콘솔 상 "Status" 컬럼. `not_available` 제외 → ok/error/warning만 포함 |

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

**검증**: `total_count = 2758` — 콘솔에서 수동 export한 xlsx 건수와 정확히 일치.

## 4. 페이지네이션

`query.page.start`는 **1부터 시작**. `limit`은 최대 1000 단위로 확인.

| page | start | limit | 반환 건수 |
|---|---|---|---|
| 1 | 1 | 1000 | 1000 |
| 2 | 1001 | 1000 | 1000 |
| 3 | 2001 | 1000 | 758 |

3개 페이지 합산 결과 `cloud_service_id` 기준 **중복 0건**, 총 2758건으로 `total_count`와 일치. 3회 순차 호출로
전체 데이터셋을 빠짐없이 안전하게 수집 가능함을 확인.

## 5. ID → 이름 매핑

`CloudService` 응답에는 `project_id`, `workspace_id`만 있고 사람이 읽을 이름은 없다. Identity 서비스에서
별도 조회해서 매핑해야 한다.

### 5-1. Project (81개)

- 엔드포인트: `POST /identity/v2/project/list`
- 단건 이름 검색: 최상위 `{"name": "harim-FSC_Dev"}` (정확 일치)
- 다건 일괄 조회: `query.filter`에 `operator: "in"`으로 project_id 배열을 한 번에 조회 (81개 → 1회 호출로 전부 매핑 성공)
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

### 5-2. Project Group (31개)

- 엔드포인트: `POST /identity/v2/project-group/list`
- 5-1과 동일하게 `project_group_id`를 `in`으로 일괄 조회
- **project_group = 고객사 단위**로 확인됨 (예: `(주)하림`, `HD현대일렉트릭`, `SK쉴더스` 등). project는 그
  아래 환경 단위(`-Dev`, `-Prd`, `-Network` 등)로 구성.
- 응답 필드: `project_group_id`, `name`, `parent_group_id`, `workspace_id` 등

### 5-3. Workspace (1개)

- 데이터셋 내 `workspace_id`는 `workspace-262a7ab4df64` 단 하나뿐 → 별도 매핑 불필요할 정도로 단순.
- `POST /identity/v2/workspace/get` 호출 시 `ERROR_PERMISSION_DENIED` — 현재 계정 권한으로는 조회 불가.
  **보류** (ID 그대로 사용해도 리포트 작성에 지장 없음).

## 6. 알려진 제약 / 보류 사항

- **토큰 만료**: 웹 콘솔 세션 토큰이라 수명이 짧음. 만료 시 `ERROR_AUTHENTICATE_FAILURE` 발생 → 재발급 후 재시도 필요.
- **관리자 권한 필요 기능**: App/ApiKey 발급(장기 토큰), Workspace 상세 조회 — 현재 계정으로는 불가하여 보류.
- **로컬 API 문서 스키마 불완전**: `resources/*.json`에 파싱된 필드 목록이 실제 API보다 적은 경우가 있음
  (예: `CloudServiceReference`, `ProjectInfo`의 일부 필드 누락). 실제 필터 가능 필드는 응답 예제나
  Swagger 스키마로 교차 검증 필요.

## 7. 구현 산출물

| 파일 | 역할 |
|---|---|
| `cloudforet_client.py` | 재사용 클라이언트 모듈 (`build_filters`, `fetch_all` — 페이지네이션/CA번들/401 처리 포함) |
| `fetch_report_data.py` | CLI 실행 스크립트 (`--group`, `--type`, `--category`, `--project-id`, `--exclude-status` 옵션) |

## 8. 다음 단계

1. `fetch_report_data.py`로 실제 2758건 데이터 수집 → 로컬 JSON 저장
2. Project/ProjectGroup 매핑 테이블을 결합해 `project_id` → 고객사/프로젝트명으로 치환
3. 수집된 데이터를 기반으로 **HTML 리포트** 생성 (표/집계/필터 UI 등 형태 논의 필요)
4. (이후 단계) 리포트 생성 과정을 서비스화하여 웹/WAS 서버로 제공
