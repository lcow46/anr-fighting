# 🛡️ AWS Trusted Advisor AI Report Automation

SpaceONE(Cloudforet) API를 통해 다수의 AWS 계정에 대한 Trusted Advisor(보안) 항목을 수집하고, Amazon Bedrock(AI)을 활용해 계정별 맞춤형 보안 분석 HTML 보고서를 자동 생성하는 서버리스 파이프라인입니다.

## 📐 Architecture

이 프로젝트는 완전 관리형 AWS 서버리스 아키텍처로 구성되어 있으며, 다수의 계정을 병렬로 처리하여 빠르고 효율적으로 동작합니다.

![Architecture](cloudforet-archi.png)

1. **Trigger**: `EventBridge Scheduler`를 통해 매월 1회(또는 지정된 Cron 주기) 파이프라인이 자동 시작됩니다.
2. **Orchestration**: `AWS Step Functions`가 전체 워크플로우를 제어합니다.
2-1. **Data Fetching (1차 Lambda)**:
 * 고정된 API Key로 `SpaceONE API`를 호출하여 전체 계정의 Trusted Advisor 데이터를 수집합니다.
 * 페이로드 용량 제한(256KB)을 우회하기 위해 전체 원본 데이터는 `S3 Bucket`에 JSON 형태로 임시 저장하고, 경량화된 계정 배열만 다음 단계로 넘깁니다.
2-2. **Parallel Processing (Map State)**:
 * 수집된 계정(Account) 수만큼 2차 Lambda를 병렬(Map)로 실행합니다.
 * Map State를 활용해 수십~수백 개의 계정을 동시에 분석합니다.
2-3. **AI Analysis & Reporting (2차 Lambda)**:
 * `S3 Bucket`에서 각 계정에 해당하는 원본 데이터를 읽어옵니다.
 * 취약점(Error/Warning) 데이터를 추출하여 `Amazon Bedrock (Nova 모델)`에 전송하고, 위험도 분석 및 조치 가이드 요약을 받아옵니다.
 * AI 요약본과 원본 상세 설명(Description)이 포함된 최종 아코디언 UI 형태의 HTML 보고서를 생성하여 `S3 Bucket`에 저장합니다.

## 📝 Environment Variables (환경 변수)

각 Lambda 함수에 다음 환경 변수를 설정해야 합니다.

### 1차 수집 Lambda

| Key | Description |
| --- | --- |
| `API_BASE_URL` | SpaceONE API 엔드포인트 URL | 
| `CLOUDFORET_API_KEY` | SpaceONE 인증용 API Secret Key | 
| `S3_BUCKET_NAME` | 원본 JSON 데이터를 저장할 S3 버킷명 |

### 2차 분석 및 리포트 생성 Lambda

| Key | Description | 
| --- | --- | 
| `S3_BUCKET_NAME` | 최종 HTML 리포트를 저장할 S3 버킷명 | 
| `BEDROCK_REGION` | Bedrock을 호출할 AWS 리전 |
| `BEDROCK_MODEL_ID` | 사용할 AI 모델의 Inference Profile ID |


## 📂 S3 Storage Structure (Output)

결과물은 지정한 S3 버킷 내에 날짜별로 정리되어 저장됩니다.

```text
s3 bucket
 ├── raw/
 │    └── 2026-08-07/
 │         └── ta_raw_data.json (1차 람다가 수집한 전체 JSON 데이터)
 └── reports/
      └── 2026-08-07/
           ├── trusted_advisor_report_AccountA_20260807.html (어카운트별 보고서)
           ├── trusted_advisor_report_AccountB_20260807.html
           └── ...

```
