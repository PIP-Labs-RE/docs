# Trace 프로바이더 데이터 감사 스테이징 핸드오프

이 문서는 Trace 데이터 감사 수집(ingestion) 및 조회를 위한 현재 스테이징 API의 형태를 설명합니다. 최신 Trace Schema를 반영합니다.

여기에서 설명하는 API와 Trace Schema는 프로바이더에 종속되지 않습니다 — 모든 프로바이더는 동일한 엔드포인트, 헤더, 스키마를 통해 통합됩니다. 아래의 구체적인 예시는 요청과 응답의 형태를 쉽게 이해할 수 있도록 실제 운영 중인 프로바이더인 Kled를 사용합니다. Kled가 등장하는 부분에는 본인의 프로바이더 식별 정보(`X-Provider` 값, source ID, URL, 정책 참조)를 대입하면 됩니다.

## 액세스 받기

쓰기 액세스는 게이트가 적용되어 있습니다. 통합하기 전에 프로바이더는 Story 팀에 연락하여 온보딩을 진행해야 합니다. Story 팀에서 프로바이더 식별 정보를 화이트리스트에 등록하고, 스테이징 API 키를 발급하며, 쓰기 요청에 사용해야 하는 `X-Provider` 값을 할당합니다. 이 절차가 완료되기 전까지는 쓰기 요청이 거부됩니다. 읽기, 검색, 스코프 그룹(scoped-group) 엔드포인트는 공개 감사 뷰이며 키가 필요하지 않습니다.

액세스를 요청하려면 Story 팀에 연락하여 온보딩 절차를 시작하세요.

Base URL:

```text
https://staging-api.storyprotocol.net
```

## API 흐름

```text
Provider client
  -> story-api webhook batch endpoint
  -> asynchronous Story processing
  -> story-api read/search endpoints
```

`records:batch` 응답에는 항상 항목별 상태가 포함됩니다. `202 Accepted`는 적어도 하나의 항목이 비동기 처리를 위해 수락되었으며 충돌이 발생하지 않았음을 의미합니다. `200 OK`는 모든 항목이 중복이었기 때문에 새로운 비동기 작업이 큐에 추가되지 않았음을 의미합니다. `409 Conflict`는 적어도 하나의 항목이 충돌했음을 의미하며, 동일한 응답에 포함된 `accepted` 항목은 그대로 큐에 추가됩니다. 클라이언트는 항목별 상태를 검사해야 하며, 일시적인 요청 실패는 동일한 페이로드와 동일한 `X-Batch-Id`로 재시도해야 합니다.

## 인증 및 프로바이더 스코프

모든 쓰기 요청은 다음을 포함해야 합니다:

```text
X-API-Key: <provider staging key>
X-Provider: kled
X-Batch-Id: <stable batch id>
```

쓰기 테스트에는 제공된 스테이징 API 키를 사용하세요. API 키는 해당 프로바이더와 연결되어 있어야 하며, 쓰기 요청의 `X-Provider`는 그 프로바이더와 일치해야 합니다. 읽기/검색/스코프 그룹 엔드포인트는 공개 감사 뷰입니다.

읽기 API는 글로벌 Story `data_id`를 키로 사용합니다. 프로바이더는 필드로 반환되며, 선택적 필터 또는 검색 가능 필드로 사용할 수 있습니다.

## 현재 프로바이더 / Trace 정렬

프로바이더의 공개 페이로드 초안은 현재 Trace Schema v1.0 작업 형태에 매끄럽게 매핑됩니다. 프로바이더는 아래에 표시된 정규화된 Trace 필드를 전송해야 하며, 프로바이더 고유 세부 정보가 손실되지 않도록 전체 프로바이더 공개 페이로드를 `provider_payload` 아래에 포함해야 합니다.

다음 필드들은 현재 Trace v1.0 작업 형태의 일부입니다:

- `contributor.kyc_country`: 국가 수준의 KYC 관할 신호. 국가만 해당하며, 주소나 GPS에서 파생된 국가는 포함하지 않습니다.
- `contributor.consent.tos_*` 및 `contributor.consent.privacy_policy_*`: 기여자가 해당 레코드에 대해 수락한 정확한 정책 버전과 해시.
- `attestation.signed_at_utc` 및 `attestation.key_url`: 최상위 `attestation` 블록에 대한 검증 메타데이터.
- `file.behavior`: PII가 아닌 캡처/업로드 동작 신호.
- `file_specific.base.motion`: 미디어 유형에 공통적으로 적용되는 모션 신호.
- `app.legal_entity`: 앱/플랫폼 이름과 함께 제공되는 법적 거래 상대방 정보.

프로바이더 수준의 활성 ToS 및 개인정보 처리방침은 각 레코드 페이로드 안이 아니라, 프로바이더 정책 엔드포인트를 통해 설정됩니다. Trace는 활성 정책 버전, 해시, URI를 저장하므로 레코드와 통계가 수락된 정책 참조를 프로바이더의 현재 정책 문서에 연결할 수 있습니다.

다음 필드들은 페이로드에 저장되지만, 현재 스테이징 구현에서는 인덱싱되지 않습니다:

- `app.legal_entity`
- `file.behavior.*`
- `file_specific.base.motion.*`
- `file.hashes.dhash64`
- `file.hashes.ahash64`
- `file.hashes.keyframe_phashes`
- `file.content_md5` / `file.hashes.md5`
- `app.platform_name`

현재의 정확 일치 인덱스는 Trace 프론트엔드와 감사 흐름에 필요한 필드를 포괄합니다. 더 풍부한 프로바이더 필드는 여전히 저장되어 있어서, 프로바이더에게 과거 데이터를 다시 전송하도록 요청하지 않고도 나중에 노출하거나 인덱싱할 수 있습니다.

현재 스테이징 동작:

- 스테이징에서 `data_id`당 메타데이터 업데이트 상한은 이제 `100`입니다. 동일한 프로바이더측 리비전의 일부인 경우, 여러 변경 가능한 필드 변경을 하나의 `seq`를 가진 하나의 메타데이터 업데이트로 통합할 수 있습니다.
- 읽기/검색/스코프 그룹 API는 현재 스테이징 빌드에서 공개 감사 뷰입니다. 공개 Trace 페이로드에 엔터프라이즈 전용 또는 민감한 필드를 넣지 마세요. 향후 프로바이더 전용 또는 파트너 전용 읽기 계층이 필요하다면, 이는 별도의 API/제품 결정이 되어야 합니다.
- 검색 가능한 유일한 해시 유형은 정규(canonical) 콘텐츠 SHA-256입니다. `phash64`, `dhash64`, `ahash64`, `keyframe_phashes`는 유용할 경우 여전히 전송해야 하지만, 현재는 저장 전용입니다.
- 권장되는 `contributor.tax_status` 값은 `submitted`, `not_submitted`, `not_applicable`, `unknown`입니다. 프로바이더는 `tax_form_on_file=true`를 `submitted`로, `false`를 `not_submitted`로 매핑할 수 있습니다.
- 권장되는 `contributor.account_verification_status` 값은 `verified`, `pending`, `failed`, `unverified`입니다.
- 스테이징에서 attestation 서명은 선택 사항입니다. 프로덕션 검증을 위해서는 프로바이더가 `payload_hash`, `signature`, `key_id`, `key_url`, `signed_at_utc`를 전송해야 합니다.
- `source_record_id`에는 프로바이더의 안정적인 공개 미디어 ID(예: `kmf_...`)가 포함되어야 합니다. 스테이징은 주변 공백을 제거하고, 대소문자를 보존하며, 제어 문자는 거부하고, 최대 512바이트의 source ID를 허용합니다.

## Trace Schema v1.0

Story는 프로바이더의 원본 페이로드를 최상위 Story 계약으로 취급하는 대신, 프로바이더에 의해 정규화된 trace 메타데이터 객체를 저장합니다. 프로바이더의 전체 원본 공개 페이로드는 `provider_payload` 아래에 보존되어야 합니다.

프로바이더는 표준화된 Trace Schema 필드를 직접 채워야 합니다. Story는 또한 프로바이더의 전체 원본 공개 페이로드를 `provider_payload` 아래에 보존하지만, 정규화된 필드가 다른 프로바이더와 프론트엔드가 사용해야 하는 이식 가능한 Trace 계약입니다.

`initial_metadata_json`과 `metadata_json`에서는 `schema_version: trace-v1.0`을 사용하세요. Story는 내부 이벤트 해시를 계산하기 전에 메타데이터 JSON을 정규화하므로, 객체 키 순서는 멱등성이나 충돌 감지에 영향을 주지 않습니다.

프로바이더는 쓰기 페이로드에 트랜잭션 해시를 보낼 필요가 없습니다. Story가 `tx_hash`를 소유하며, 등록, 메타데이터 업데이트, 검색 결과 행에 대한 읽기 응답에서 반환합니다. 브로드캐스트 후 Story가 채울 때까지 빈 문자열로 반환됩니다.

초기 등록에는 하나의 정규(canonical) 콘텐츠 해시가 포함되어야 합니다. 허용되는 필드는 `asset.hash`, `content_hash`, `file.content_sha256`, `file.hashes.sha256`이며, 모두 `sha256:<64-lowercase-hex>` 형식으로 정규화됩니다. 둘 이상의 별칭이 존재하면 동일한 해시를 나타내야 합니다.

권장되는 최상위 형태:

```json
{
  "schema_version": "trace-v1.0",
  "file": {
    "content_sha256": "sha256:<64-hex>",
    "mime_type": "video/mp4",
    "media_category": "video",
    "size_bytes": 123456,
    "hashes": {
      "phash64": "facebeef01234567",
      "dhash64": "1b9072d44a8be3c1",
      "ahash64": "ff7e3c1a90d5e8c2",
      "keyframe_phashes": [
        "0011223344556677",
        "8899aabbccddeeff"
      ]
    },
    "behavior": {
      "captured_at_utc": "2026-05-13",
      "uploaded_at_utc": "2026-05-13",
      "capture_to_upload_seconds": 421,
      "capture_to_upload_bucket": "5-60min",
      "upload_session_size": 3,
      "upload_session_kind": "gallery_pick",
      "captured_via": "ios_native_camera",
      "uploaded_via": "ios_app",
      "client_version": "ios-1.42.0"
    }
  },
  "file_specific": {
    "base": {
      "motion": {
        "compass_heading": 247.3,
        "compass_heading_reference": "true_north",
        "speed_bucket": "stationary"
      }
    },
    "video": {},
    "image": {},
    "document": {}
  },
  "contributor": {
    "anon_id": "kled-public-user-id",
    "kyc_status": "verified",
    "kyc_country": "US",
    "geo_region": "US",
    "tax_status": "submitted",
    "account_verification_status": "verified",
    "consent": {
      "tos_version": "2026-05-20",
      "tos_hash": "sha256:<64-hex-policy-hash>",
      "tos_uri": "https://kled.ai/terms/2026-05-20",
      "privacy_policy_version": "2026-05-20",
      "privacy_policy_hash": "sha256:<64-hex-policy-hash>",
      "privacy_policy_uri": "https://kled.ai/privacy/2026-05-20"
    }
  },
  "app": {
    "platform_name": "kled.ai",
    "legal_entity": "Nitrility Inc. (Delaware, USA)"
  },
  "timestamps": {
    "occurred_at": "2026-05-13T00:00:00Z",
    "uploaded_at": "2026-05-13T00:00:00Z",
    "captured_at": "2026-05-12T23:59:00Z"
  },
  "attestation": {
    "payload_hash": "sha256:<canonical-trace-schema-v1-json>",
    "signature": "optional-signature",
    "key_id": "optional-key-id",
    "key_url": "https://kled.ai/.well-known/verification-keys.json",
    "signed_at_utc": "2026-05-13T00:00:02Z"
  },
  "provider_payload": {
    "...": "full provider public payload"
  }
}
```

현재 정확 일치로 검색 가능한 필드:

```text
provider
source_record_id
media_id_public              (alias for source_record_id when present in metadata)
file.content_sha256         (also accepts content_hash and file.hashes.sha256)
file.mime_type              (also accepts mime_type / mimetype aliases)
file.media_category         (also accepts media_category)
contributor.anon_id
collection_id
customer_id
task_id
contributor.kyc_status
contributor.kyc_country
contributor.geo_region
tos_hash                    (also accepts contributor.consent.tos_hash)
privacy_policy_hash         (also accepts contributor.consent.privacy_policy_hash)
```

`app.platform_name`, `app.legal_entity`, `file.behavior`, `file_specific.base.motion`, `file.hashes.phash64`, `dhash64`, `ahash64`, `md5`, `keyframe_phashes`와 같은 필드는 저장된 페이로드에 보존될 수 있지만, 현재 스테이징 구현에서는 정확 일치 인덱스가 없습니다. 분포 정보는 `/stats`를 사용하고, Trace 프론트엔드/감사 뷰의 선택적 쿼리 스코프로 `provider`를 사용하세요.

### 필드 가이드

- `source_record_id`: 프로바이더 소유의 안정적인 공개 미디어 ID. 프로바이더의 `kmf_...` 값이 여기에 해당합니다. 스테이징은 이 값을 최대 512바이트까지 허용합니다.
- `contributor.anon_id`: 프로바이더 소유의 공개 익명화 기여자 ID.
- `contributor.kyc_status`: 권장 값은 `verified`, `pending`, `failed`, `unverified`입니다.
- `contributor.kyc_country`: 사용 가능한 경우 KYC의 ISO 3166-1 alpha-2 국가 코드. 국가만 해당하며, 주소나 GPS에서 파생된 국가는 포함하지 않습니다.
- `contributor.tax_status`: 권장 값은 `submitted`, `not_submitted`, `not_applicable`, `unknown`입니다. 프로바이더의 `tax_form_on_file`의 경우, `true`를 `submitted`로, `false`를 `not_submitted`로 매핑합니다.
- `contributor.account_verification_status`: 권장 값은 `verified`, `pending`, `failed`, `unverified`입니다.
- `contributor.consent.tos_*` 및 `contributor.consent.privacy_policy_*`: 해당 레코드에 대해 수락된 정책 버전, 해시, URI.
- 프로바이더 활성 정책은 `PUT /webhook/v1/data-audit/provider-policy`를 통해 별도로 설정됩니다.
- `attestation.signature`: 스테이징에서 선택 사항입니다. 프로덕션 검증을 위해서는 프로바이더가 `payload_hash`, `signature`, `key_id`, `key_url`, `signed_at_utc`를 전송해야 합니다.
- `file.behavior`: PII가 아닌 업로드/캡처 동작 신호.
- `file_specific.base.motion`: 미디어 종류 전반에 적용될 수 있는 공유 모션 신호.

## 쓰기 API

### 활성 프로바이더 정책 설정

프로바이더가 새로운 현재 이용약관 또는 개인정보 처리방침을 게시할 때 이 엔드포인트를 사용합니다. Trace는 활성 버전, SHA-256 해시, URI를 저장하므로 프론트엔드가 정책 문서로 링크할 수 있고, 통계가 레코드 수준에서 수락된 정책 참조와 프로바이더의 현재 정책을 비교할 수 있습니다.

정책 문서 행은 감사/가시성을 위해서만 저장됩니다. 직접 인덱싱되거나 집계되지는 않습니다. 레코드 페이로드는 여전히 `contributor.consent.tos_*` 및 `contributor.consent.privacy_policy_*`를 포함해야 하며, 이러한 레코드 수준의 수락된 정책 참조가 `/stats`, 스코프 그룹 요약, 정책 해시 검색에서 사용되는 값입니다.

```http
PUT /webhook/v1/data-audit/provider-policy
Content-Type: application/json
X-API-Key: <provider staging key>
X-Provider: kled
```

```json
{
  "tos": {
    "version": "2026-06-01",
    "hash": "sha256:<64-hex-policy-hash>",
    "uri": "https://kled.ai/terms/2026-06-01",
    "effective_at": "2026-06-01T00:00:00Z"
  },
  "privacy_policy": {
    "version": "2026-06-01",
    "hash": "sha256:<64-hex-policy-hash>",
    "uri": "https://kled.ai/privacy/2026-06-01",
    "effective_at": "2026-06-01T00:00:00Z"
  }
}
```

현재 및 과거 정책 문서 조회:

```http
GET /api/v1/data-audit/providers/kled/policy
GET /api/v1/data-audit/providers/kled/policies/tos/sha256:<64-hex-policy-hash>
GET /api/v1/data-audit/providers/kled/policies/privacy_policy/sha256:<64-hex-policy-hash>
```

### 레코드 등록

초기 백로그 및 실시간 등록 배치에 이 엔드포인트를 사용하세요. 프로바이더는 안정적인 `source_record_id`를 보내고, Story가 `data_id`를 생성하여 매핑을 반환합니다.

```http
POST /webhook/v1/data-audit/records:batch
Content-Type: application/json
X-API-Key: <provider staging key>
X-Provider: kled
X-Batch-Id: kled-records-000001
```

요청 본문은 JSON 배열입니다:

```json
[
  {
    "source_record_id": "kmf_8a9c2e7d4b1f0e23",
    "initial_metadata_root": "sha256:<canonical-trace-schema-v1-json>",
    "initial_metadata_json": {
      "schema_version": "trace-v1.0",
      "file": {
        "content_sha256": "sha256:<64-hex>",
        "mime_type": "video/mp4",
        "media_category": "video",
        "size_bytes": 123456,
        "hashes": {
          "phash64": "facebeef01234567",
          "dhash64": "1b9072d44a8be3c1",
          "ahash64": "ff7e3c1a90d5e8c2",
          "keyframe_phashes": [
            "0011223344556677",
            "8899aabbccddeeff"
          ]
        },
        "behavior": {
          "captured_at_utc": "2026-05-13",
          "uploaded_at_utc": "2026-05-13",
          "capture_to_upload_seconds": 421,
          "capture_to_upload_bucket": "5-60min",
          "upload_session_size": 3,
          "upload_session_kind": "gallery_pick",
          "captured_via": "ios_native_camera",
          "uploaded_via": "ios_app",
          "client_version": "ios-1.42.0"
        }
      },
      "file_specific": {
        "base": {
          "motion": {
            "compass_heading": 247.3,
            "compass_heading_reference": "true_north",
            "speed_bucket": "stationary"
          }
        },
        "video": {
          "duration_ms": 120000,
          "width": 1920,
          "height": 1080
        }
      },
      "contributor": {
        "anon_id": "kup_123",
        "kyc_status": "verified",
        "kyc_country": "US",
        "geo_region": "US",
        "tax_status": "submitted",
        "account_verification_status": "verified",
        "consent": {
          "tos_version": "2026-05-20",
          "tos_hash": "sha256:<64-hex-policy-hash>",
          "tos_uri": "https://kled.ai/terms/2026-05-20",
          "privacy_policy_version": "2026-05-20",
          "privacy_policy_hash": "sha256:<64-hex-policy-hash>",
          "privacy_policy_uri": "https://kled.ai/privacy/2026-05-20"
        }
      },
      "app": {
        "platform_name": "kled.ai",
        "legal_entity": "Nitrility Inc. (Delaware, USA)"
      },
      "timestamps": {
        "occurred_at": "2026-05-13T00:00:00Z",
        "uploaded_at": "2026-05-13T00:00:00Z"
      },
      "attestation": {
        "payload_hash": "sha256:<canonical-trace-schema-v1-json>",
        "signature": "optional-on-staging",
        "key_id": "kled-verify-2026-q1",
        "key_url": "https://kled.ai/.well-known/verification-keys.json",
        "signed_at_utc": "2026-05-13T00:00:02Z"
      },
      "provider_payload": {
        "media_id_public": "kmf_8a9c2e7d4b1f0e23"
      }
    },
    "occurred_at": "2026-05-13T00:00:00Z"
  }
]
```

`initial_metadata_root`는 정규화된 Trace Schema v1.0 메타데이터 JSON에 대한 프로바이더의 결정론적 해시여야 합니다. 현재 스테이징 API에서는 Story가 이 값을 제출된 그대로 저장하며, `initial_metadata_json`에 대한 해시 검증은 아직 강제되지 않습니다.

성공 응답:

- `202 Accepted`: 하나 이상의 레코드가 새로 수락되어 큐에 추가되었습니다.
- `200 OK`: 요청은 처리되었지만 모든 항목이 `duplicate`였기 때문에 큐에 추가된 레코드가 없습니다.
- `409 Conflict`: 하나 이상의 항목이 `conflict`였습니다. 동일한 응답에 포함된 `accepted` 항목은 그대로 큐에 추가됩니다.

```json
{
  "request_id": "story-request-uuid",
  "provider": "kled",
  "batch_id": "kled-records-000001",
  "format": "json",
  "kind": "records",
  "records": 1,
  "accepted": 1,
  "duplicates": 0,
  "conflicts": 0,
  "messages": 1,
  "items": [
    {
      "source_record_id": "kmf_8a9c2e7d4b1f0e23",
      "data_id": "story-generated-uuid",
      "status": "accepted"
    }
  ]
}
```

반환되는 `data_id`는 향후 메타데이터 업데이트 및 읽기에 사용되는 Story ID입니다. 동일한 `X-Provider` + `source_record_id`를 다시 보내면 동일한 `data_id`가 생성됩니다.

레코드가 이미 정확히 동일한 초기 등록 페이로드로 영구 저장되어 있는 경우, 항목은 `status: "duplicate"`를 반환하며 다시 큐에 추가되지 않습니다. 동일한 `source_record_id`가 다른 초기 메타데이터로 이미 영구 저장되어 있는 경우, 항목은 `status: "conflict"`를 반환하며 큐에 추가되지 않습니다. 첫 번째 요청이 아직 큐에 있는 동안 중첩된 재시도가 발생하면 `accepted`를 반환할 수 있으며, 다운스트림 수집은 여전히 멱등합니다. 동일한 배치 내의 다른 유효한 항목은 여전히 `accepted`를 반환할 수 있습니다.

호출자가 이미 Story가 할당한 UUID를 가지고 있다면, 하위 레벨 엔드포인트는 다음과 같습니다:

```http
POST /webhook/v1/data-audit/data-ids:batch
```

이 엔드포인트는 모든 레코드에 `data_id`가 필요하며 권장되는 벤더 경로는 아닙니다.

### 메타데이터 업데이트 제출

이후 수정 또는 변경 가능한 메타데이터 변경에 이 엔드포인트를 사용하세요. 동일한 `data_id`에 대해 `seq`는 `1`에서 `100` 사이여야 합니다.

`metadata_json`은 변경 후 전체 최신 Trace 메타데이터 상태여야 하며, 부분 diff나 JSON patch가 아니어야 합니다. 예를 들어, KYC만 변경된 경우라도 현재 메타데이터 상태의 일부로 남아있는 변경되지 않은 file, asset, app, consent, provider 페이로드 필드를 포함해야 합니다. 이렇게 하면 각 메타데이터 이벤트가 `metadata_root`에 대해 독립적으로 검증 가능하며, Story가 프로바이더별 병합 규칙 없이 최신 상태를 재구성할 수 있습니다.

```http
POST /webhook/v1/data-audit/metadata-updates:batch
Content-Type: application/json
X-API-Key: <provider staging key>
X-Provider: kled
X-Batch-Id: kled-metadata-000001
```

요청 본문은 JSON 배열입니다:

```json
[
  {
    "data_id": "11111111-1111-4111-8111-111111111111",
    "seq": 1,
    "prev_metadata_root": "sha256:<previous-canonical-trace-schema-v1-json>",
    "metadata_root": "sha256:<new-canonical-trace-schema-v1-json>",
    "metadata_json": {
      "schema_version": "trace-v1.0",
      "contributor": {
        "anon_id": "kup_123",
        "kyc_status": "unverified",
        "consent": {
          "tos_version": "2026-06-01",
          "tos_hash": "sha256:<64-hex-policy-hash>",
          "tos_uri": "https://kled.ai/terms/2026-06-01",
          "privacy_policy_version": "2026-05-20",
          "privacy_policy_hash": "sha256:<64-hex-policy-hash>",
          "privacy_policy_uri": "https://kled.ai/privacy/2026-05-20"
        }
      },
      "app": {
        "platform_name": "kled.ai"
      },
      "provider_payload": {
        "media_id_public": "kmf_8a9c2e7d4b1f0e23",
        "reason": "kyc_status_changed"
      }
    },
    "occurred_at": "2026-05-13T00:00:01Z"
  }
]
```

`metadata_root`는 업데이트된 전체 Trace Schema v1.0 메타데이터 JSON의 정규화된 형태에 대한 프로바이더의 결정론적 해시여야 합니다. 현재 스테이징 API에서는 Story가 이 값을 제출된 그대로 저장하며, `metadata_json`에 대한 해시 검증은 아직 강제되지 않습니다.

### 선택적 백로그 파일 엔드포인트

일반적인 통합 경로는 `/records:batch`에서 `application/json`을 사용하는 것입니다. 백로그 도구를 위해 이러한 하위 레벨 라우트 변형도 존재하지만, 데이터 ID 파일 라우트는 모든 레코드에 `data_id`가 필요합니다.

```text
POST /webhook/v1/data-audit/data-ids:batch-ndjson
POST /webhook/v1/data-audit/metadata-updates:batch-ndjson
POST /webhook/v1/data-audit/data-ids:batch-csv
POST /webhook/v1/data-audit/metadata-updates:batch-csv
POST /webhook/v1/data-audit/data-ids:batch-txt
POST /webhook/v1/data-audit/metadata-updates:batch-txt
```

NDJSON은 다음을 요구합니다:

```text
Content-Type: application/x-ndjson
Content-Encoding: gzip
```

압축 해제된 각 줄은 위에 표시된 동일한 필드를 가진 하나의 JSON 레코드입니다.

CSV는 `Content-Type: text/csv`를 요구합니다.

데이터 ID CSV 컬럼:

```text
data_id,source_record_id,initial_metadata_root,initial_metadata_json,occurred_at
```

메타데이터 업데이트 CSV 컬럼:

```text
data_id,seq,prev_metadata_root,metadata_root,metadata_json,occurred_at
```

`initial_metadata_json`과 `metadata_json`은 존재할 경우 따옴표로 묶인 CSV 필드 안의 유효한 JSON이어야 합니다.

TXT는 `Content-Type: text/plain`을 요구하며, 줄 단위 JSON, JSON 배열, 또는 동일한 CSV 컬럼을 사용하는 헤더 구분 콤마/탭 텍스트를 허용합니다.

## 읽기 API

읽기 엔드포인트는 공개 감사 뷰입니다. `provider`는 선택 사항이며 좁히기 필터로 작동합니다.

읽기 모델:

- `GET /data-ids/{data_id}`는 등록 프로파일과 최신 raw 메타데이터 이벤트를 반환합니다. 최신 메타데이터 이벤트가 diff임은 보장되지 않습니다. 현재 저장된 최상위 시퀀스에 대해 제출된 정확한 업데이트 페이로드입니다.
- `GET /data-ids/{data_id}/metadatas`는 `seq: 0`의 등록 및 이후 메타데이터 업데이트를 포함한 전체 append-only 메타데이터 이력을 반환합니다.
- 검색, 자산 영수증 조회, 스코프 그룹 요약은 해당 이벤트에서 파생된 Story의 정규화된 최신 상태 프로젝션을 사용합니다. 이 프로젝션은 MIME 유형, 미디어 카테고리, KYC 상태, TOS/개인정보 처리방침 버전, 라이프사이클 상태, `tx_hash` 같은 필드를 구동합니다.

### Story 데이터 ID로 trace 가져오기

```http
GET /api/v1/data-audit/data-ids/11111111-1111-4111-8111-111111111111
```

응답 발췌:

```json
{
  "data_id": "11111111-1111-4111-8111-111111111111",
  "provider": "kled",
  "profile": {
    "data_id": "11111111-1111-4111-8111-111111111111",
    "provider": "kled",
    "tx_hash": ""
  },
  "latest_metadata": {
    "data_id": "11111111-1111-4111-8111-111111111111",
    "seq": 0,
    "event_type": "DataRegistered",
    "tx_hash": ""
  }
}
```

### 메타데이터 이력 조회

```http
GET /api/v1/data-audit/data-ids/11111111-1111-4111-8111-111111111111/metadatas
```

각 메타데이터 행에는 `tx_hash`가 포함되며, 초기에는 빈 문자열입니다:

```json
{
  "data_id": "11111111-1111-4111-8111-111111111111",
  "metadatas": [
    {
      "seq": 0,
      "event_type": "DataRegistered",
      "tx_hash": ""
    },
    {
      "seq": 1,
      "event_type": "MetadataUpdated",
      "tx_hash": ""
    }
  ]
}
```

### 인덱싱된 필드로 검색

```http
GET /api/v1/data-audit/search?field=source_record_id&value=kmf_8a9c2e7d4b1f0e23
```

각 검색 매치에는 `tx_hash`가 포함되며, 초기에는 빈 문자열입니다. 검색 응답에는 `next_cursor`가 포함됩니다. 정확한 필드 값으로 레코드를 찾으려면 `/search`를 사용한 다음, 감사 검증에 사용되는 정규(canonical) 이벤트 페이로드를 얻으려면 `/api/v1/data-audit/data-ids/{data_id}` 또는 `/api/v1/data-audit/data-ids/{data_id}/metadatas`를 사용하세요. MIME 유형, 미디어 카테고리, KYC 상태, 국가/지역, TOS, 개인정보 처리방침 버전 같은 광범위한 분포 카운트에는 `/stats`를 사용하세요. UI에 최신 레코드가 필요하면 `/recent` 또는 `/feed`를 사용하세요.
`source_record_id` 검색은 정확 일치이며 대소문자를 구분합니다.

기타 예시:

```text
GET /api/v1/data-audit/search?field=source_record_id&value=kmf_8a9c2e7d4b1f0e23
GET /api/v1/data-audit/search?field=asset_hash&value=sha256:<64-hex>
GET /api/v1/data-audit/search?field=file.content_sha256&value=<64-hex-or-sha256-prefixed-hex>
GET /api/v1/data-audit/search?field=customer_id&value=<customer-id>
GET /api/v1/data-audit/search?field=task_id&value=<task-id>
GET /api/v1/data-audit/search?field=collection_id&value=<collection-id>
GET /api/v1/data-audit/search?field=contributor.consent.tos_hash&value=sha256:<64-hex-policy-hash>
GET /api/v1/data-audit/search?field=contributor.consent.privacy_policy_hash&value=sha256:<64-hex-policy-hash>
GET /api/v1/data-audit/search?field=collection_id&value=<collection-id>&provider=kled&limit=100&cursor=<next_cursor>
```

### 프로바이더 합계

```http
GET /api/v1/data-audit/stats
GET /api/v1/data-audit/stats?provider=kled
```

응답 발췌:

```json
{
  "total_records": 1000324,
  "total_contributors": 902111,
  "provider": "kled",
  "provider_records": 100234,
  "provider_contributors": 87234,
  "kyc_status": { "verified": 91234, "pending": 9000 },
  "account_verification": { "verified": 90000, "pending": 10234 },
  "tax_status": { "complete": 80000, "submitted": 20234 },
  "tos_versions": { "2026-05": 100234 },
  "privacy_policy_versions": { "2026-05": 100234 },
  "media_category_coverage": { "video": 80000, "image": 20234 },
  "mime_distribution": { "video/mp4": 80000, "image/jpeg": 20234 },
  "geo_distribution": { "us": 70234, "ca": 30000 },
  "total_size_bytes": 1234567890,
  "size_record_count": 100234,
  "average_size_bytes": 12316,
  "active_tos_version": "2026-05",
  "active_tos_hash": "sha256:<64-hex-policy-hash>",
  "active_tos_uri": "https://kled.ai/terms/2026-05",
  "active_privacy_policy_version": "2026-05",
  "active_privacy_policy_hash": "sha256:<64-hex-policy-hash>",
  "active_privacy_policy_uri": "https://kled.ai/privacy/2026-05"
}
```

레코드 합계는 등록된 레코드를 한 번씩 카운트합니다. 기여자 합계는 기여자 ID가 존재하는 경우 고유한 `provider + contributor_anon_id` 값을 카운트합니다. 메타데이터 업데이트는 이 합계를 증가시키지 않습니다. 분포 맵과 크기 합계는 최신 자산 프로젝션을 따르므로, 전체 상태 메타데이터 업데이트는 레코드를 한 버킷에서 다른 버킷으로 이동시킬 수 있습니다. `average_size_bytes`는 `total_size_bytes / size_record_count`이며, 여기서 `size_record_count`는 양의 크기를 가진 최신 프로젝션을 카운트합니다. 앱/플랫폼 필드는 레코드에 저장될 수 있지만 통계 스코프는 아닙니다.

### 최근 수집 피드

```http
GET /api/v1/data-audit/feed?limit=50
GET /api/v1/data-audit/feed?provider=kled&limit=50
GET /api/v1/data-audit/feed?provider=kled&limit=50&cursor=<next_cursor>
```

피드는 등록과 메타데이터 업데이트를 모두 포함한 최근 감사 이벤트를 반환합니다. 행은 수집 시간 기준 최신순으로 정렬되며 `event_type`, `seq`, `data_id`, `source_record_id`, `asset_hash`, `occurred_at`, `ingested_at`이 포함됩니다. 이전 행을 가져오려면 `next_cursor`를 사용하세요.

### 최근 등록 레코드

```http
GET /api/v1/data-audit/recent?limit=50
GET /api/v1/data-audit/recent?provider=kled&limit=50
GET /api/v1/data-audit/recent?provider=kled&limit=50&cursor=<next_cursor>
```

이것은 등록된 영수증 행만 반환하며, 수집 시간 기준 최신순입니다. 이전 행을 가져오려면 `next_cursor`를 사용하세요.

### 콘텐츠 해시별 자산 영수증

```http
GET /api/v1/data-audit/assets/sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

허용되는 해시 형식은 `sha256:<64-hex>`, 일반 `<64-hex>`, `0x<64-hex>`, `0x1220<64-hex>`입니다. 응답은 정규화된 `sha256:<64-lowercase-hex>` 자산 해시와 일치하는 모든 영수증 행을 반환합니다.

### 스코프 그룹

스코프 그룹은 Trace 프론트엔드 또는 파트너 리뷰어가 리뷰 세트에 대한 공개 스냅샷을 생성할 수 있게 해줍니다. 프로바이더 리뷰 워크플로의 경우, 랩(lab)은 Story가 생성한 `data_id`나 콘텐츠 해시가 아닌, 프로바이더의 `source_record_id` 값을 사용해야 합니다. Trace는 결정론적 `data_id`를 계산하고, 제출된 모든 레코드가 존재하는지 검증하며, 전체 세트가 유효한 경우에만 그룹을 생성합니다.

source record ID로 생성:

```http
POST /api/v1/data-audit/scoped-groups
Content-Type: application/json
Idempotency-Key: kled-review-000001
```

```json
{
  "title": "Kled review set",
  "description": "Kled source record IDs supplied to a lab",
  "provider": "kled",
  "source_record_ids": ["kmf_1", "kmf_2"]
}
```

붙여넣은 source record ID로 생성:

```json
{
  "title": "Kled pasted review set",
  "provider": "kled",
  "source_record_ids_text": "kmf_1\nkmf_2\nkmf_3"
}
```

source-record 그룹은 전부 또는 전무 방식(all-or-nothing)입니다. 중복된 `provider + source_record_id` 값은 `400`을 반환하고, 누락된 레코드는 `404`를 반환하며, 프로파일/소스 불일치는 `409`를 반환하고, 자산 프로젝션 없이 등록된 레코드는 `422`를 반환합니다.

더 큰 source-record CSV/TXT 입력의 경우, 먼저 presigned 업로드 URL을 요청하세요:

```http
POST /api/v1/data-audit/scoped-groups/uploads
Content-Type: application/json
```

```json
{ "format": "csv" }
```

지원되는 업로드 형식은 `csv`와 `txt`입니다.

1컬럼 CSV는 `source_record_id`를 사용하며 생성 요청에 `provider`가 필요합니다:

```csv
source_record_id
kmf_1
kmf_2
```

혼합 프로바이더 CSV는 두 컬럼을 모두 포함하며 최상위 `provider`가 필요하지 않습니다:

```csv
provider,source_record_id
kled,kmf_1
oto,oto_1
```

TXT 업로드는 줄바꿈으로 구분된 source record ID이며 생성 요청에 `provider`가 필요합니다:

```text
kmf_1
kmf_2
```

반환된 `Content-Type` 헤더와 함께 반환된 `upload_url`에 파일 바이트를 업로드한 다음, 그룹을 생성하세요:

```json
{
  "title": "Kled uploaded review set",
  "description": "CSV uploaded through presigned S3",
  "provider": "kled",
  "upload_id": "up_<uuid>.csv"
}
```

업로드된 CSV에 `provider` 컬럼이 있는 경우에만 `provider`를 생략하세요.

스코프 그룹 읽기:

```http
GET /api/v1/data-audit/scoped-groups/{group_id}
GET /api/v1/data-audit/scoped-groups/{group_id}/items?limit=100
GET /api/v1/data-audit/scoped-groups/{group_id}/items?limit=100&cursor=<next_cursor>
GET /api/v1/data-audit/scoped-groups/{group_id}/export.csv
```

`GET /scoped-groups/{group_id}`는 그룹 상태를 반환하며, 완료되면 `summary`에 집계 지표를 반환합니다:

```json
{
  "profile": {
    "group_id": "sg_...",
    "title": "Kled review set",
    "description": "Kled source record IDs supplied to a lab",
    "manifest_kind": "source_record",
    "status": "complete",
    "submitted_items": 2,
    "unique_items": 2,
    "computed_at": "2026-06-04T15:30:00Z"
  },
  "summary": {
    "records_in_set": 2,
    "submitted_items": 2,
    "unique_items": 2,
    "matched_items": 2,
    "missing_items": 0,
    "matched_receipts": 2,
    "kyc_verified_percent": 50,
    "distinct_media_categories": ["image", "video"],
    "distinct_tos_versions": ["2026-05"],
    "total_size_bytes": 7340032,
    "average_size_bytes": 3670016,
    "source_distribution": { "kled": 2 },
    "media_category_coverage": { "image": 1, "video": 1 },
    "mime_distribution": { "image/jpeg": 1, "video/mp4": 1 },
    "tos_versions": { "2026-05": 2 },
    "privacy_policy_versions": { "2026-05": 2 },
    "kyc_status": { "verified": 1, "unverified": 1 },
    "geo_distribution": { "us": 2 },
    "lifecycle_status": { "registered": 2 },
    "metadata_presence": { "custom.camera": 1, "tos_acknowledgment": 2 }
  }
}
```

그룹이 여전히 큐에 있거나 처리 중인 경우, `profile.status`는 `pending` 또는 `processing`이며 `summary`는 생략됩니다.

`/items`는 제출된 `source_record_id`별로 한 행을 반환하며, 해결된 `data_id`, `asset_hash`, 영수증 요약을 포함합니다. 커서 페이지네이션 방식이며, 더 많은 행이 있을 경우 `next_cursor`를 반환합니다. CSV 내보내기는 `input_type,provider,source_record_id,data_id,asset_hash,status,...`로 시작합니다.

## 한도 및 재시도 동작

현재 스테이징 한도:

```text
Max request body: 25 MiB
Max serialized record: 350 KiB
Max metadata updates per data_id: 100
Max inline scoped-group body: 5 MiB
Max inline scoped-group hashes: 10,000
Max inline scoped-group source records: 10,000
Max source_record_id length: 512 bytes
```

재시도 가이드:

- `502`, `503`, `504`, 네트워크 타임아웃, `429`는 지수 백오프 및 지터와 함께 재시도하세요.
- 검증/인증 `4xx`는 요청이 수정될 때까지 재시도하지 마세요.
- 재시도 전반에 걸쳐 `data_id`, 요청 본문, `X-Batch-Id`를 안정적으로 유지하세요.
- 쓰기 경로는 동일한 `data_id`, 이벤트 키, 이벤트 해시에 대해 멱등합니다.
- 동일한 `data_id`와 이벤트 키가 서로 다른 메타데이터로 재시도되면, 해당 레코드는 충돌로 처리되어 거부됩니다.
