# JustInNeed Backend

JustInNeed 기록 관리 MVP 백엔드입니다. 현재 구현 범위는 세션 기록 관리와 해시태그 그룹 관리입니다.

## 구현 범위

| Method | Path | 설명 |
| --- | --- | --- |
| GET | `/sessions` | 세션 목록 조회 |
| GET | `/sessions/{id}` | 세션 상세 조회. 요약 본문과 출처를 함께 반환 |
| PATCH | `/sessions/{id}` | 세션 편집. `title`, `editedMarkdown`, `isPublic`, `isFavorite`, `tags` 수정 |
| DELETE | `/sessions/{id}` | 세션 삭제. 실제 삭제가 아니라 soft delete 처리 |
| GET | `/tag-groups` | 해시태그 그룹 목록 조회. 그룹별 매칭 세션 포함 |
| POST | `/tag-groups` | 새 해시태그 그룹 생성 |
| PATCH | `/tag-groups/{id}` | 그룹 이름과 해시태그 수정 |
| DELETE | `/tag-groups/{id}` | 그룹 삭제 |
| PATCH | `/tag-groups/order` | 그룹 표시 순서 변경 |

다음 항목은 이번 백엔드 구현에서 제외했습니다.

- 마인드맵 뷰
- 회원 관련 기능
- 어드민 관련 기능
- 즐겨찾기 전용 세션 목록 뷰

`isFavorite` 값 자체는 세션 편집 API에서 수정할 수 있습니다.

## 기술 스택

- Java 21
- Spring Boot 3.3.7
- Spring Web
- Spring Data JPA
- Bean Validation
- H2 in-memory database for local/test
- PostgreSQL driver included for production database integration
- Gradle Wrapper

## 실행 방법

```bash
./gradlew bootRun
```

Windows PowerShell에서는 다음 명령을 사용합니다.

```powershell
.\gradlew.bat bootRun
```

기본 DB는 H2 in-memory입니다. 설정은 `src/main/resources/application.yml`에 있습니다.

## 테스트

```bash
./gradlew test
```

Windows PowerShell:

```powershell
.\gradlew.bat test
```

현재 컨트롤러 테스트는 다음 흐름을 검증합니다.

- 세션 목록 조회
- 세션 상세 조회에서 요약과 출처 반환
- 세션 편집
- 세션 soft delete
- 해시태그 그룹별 매칭 세션 조회
- 해시태그 그룹 생성
- 해시태그 그룹 순서 변경

## 인증 임시 처리

회원 기능은 이번 범위에서 제외되어 있습니다. 그래서 현재 API는 임시로 `X-User-Id` 헤더를 사용합니다.

```http
X-User-Id: 1
```

헤더가 없으면 기본값 `1`로 처리합니다. 이후 회원/인증 기능이 추가되면 로그인 사용자 ID를 인증 컨텍스트에서 가져오도록 교체하면 됩니다.

## 주요 패키지 구조

```text
src/main/java/com/justinneed
  global
    common
      ApiResponse.java
      BaseEntity.java
      HashtagValidator.java
    config
      JpaConfig.java
    exception
      CustomException.java
      ErrorCode.java
      GlobalExceptionHandler.java
  session
    management
      controller
      domain
      dto
      repository
      service
    summary
      domain
  taggroup
    controller
    domain
    dto
    repository
    service
```

## 도메인 설명

### BrowsingSession

`sessions` 테이블에 매핑되는 기록 세션 엔티티입니다.

주요 필드:

- `user_id`: 소유 사용자 ID
- `title`: 기록 제목
- `started_at`, `ended_at`: 탐색 시작/종료 시각
- `page_count`: 수집 페이지 수
- `status`: `ANALYZING`, `COMPLETED`, `FAILED`
- `is_favorite`: 즐겨찾기 여부
- `is_public`: 공개 여부
- `tags`: JSON 컬럼. 세션 해시태그 목록
- `sources`: JSON 컬럼. 출처 목록
- `deleted_at`: soft delete 시각

삭제 API는 row를 물리 삭제하지 않고 `deleted_at`만 채웁니다.

### Summary

`summaries` 테이블에 매핑되는 세션 요약 엔티티입니다. `session_id`를 기본키이자 외래키로 사용합니다.

주요 필드:

- `heading`: 요약 제목
- `markdown`: 줄글 본문
- `insights`: JSON 컬럼. 핵심 인사이트 목록

세션 상세 API는 `Summary`와 `sources`를 함께 반환합니다.

### TagGroup

`tag_groups` 테이블에 매핑되는 해시태그 그룹 엔티티입니다.

주요 필드:

- `user_id`: 소유 사용자 ID
- `name`: 그룹 이름
- `hashtags`: JSON 컬럼. 그룹에 속한 해시태그 목록
- `position`: 그룹 표시 순서

`GET /tag-groups`는 각 그룹의 `hashtags`와 세션 `tags`를 비교해서 매칭되는 세션 목록을 함께 반환합니다.

## 해시태그 검증 규칙

해시태그는 `HashtagValidator`에서 공통 검증합니다.

- 최대 10개
- 한글, 영문, 숫자만 허용
- 각 태그는 최대 10자
- 공백/특수문자 불가
- 대소문자 무시 중복 불가

## API 요청 예시

### 세션 수정

```http
PATCH /sessions/1
Content-Type: application/json
X-User-Id: 1
```

```json
{
  "title": "수정된 기록 제목",
  "editedMarkdown": "수정된 Markdown 본문",
  "isPublic": true,
  "isFavorite": false,
  "tags": ["Spring", "JPA"]
}
```

### 해시태그 그룹 생성

```http
POST /tag-groups
Content-Type: application/json
X-User-Id: 1
```

```json
{
  "name": "Backend",
  "hashtags": ["Spring", "JPA"]
}
```

### 해시태그 그룹 순서 변경

```http
PATCH /tag-groups/order
Content-Type: application/json
X-User-Id: 1
```

```json
{
  "groupIds": [3, 1, 2]
}
```

`groupIds`에는 해당 사용자의 모든 그룹 ID가 새 순서대로 들어와야 합니다.

## 응답 형식

모든 API는 공통 응답 객체를 사용합니다.

```json
{
  "success": true,
  "data": {},
  "message": null
}
```

예외 발생 시:

```json
{
  "success": false,
  "data": null,
  "message": "세션을 찾을 수 없습니다."
}
```

## Git Convention 메모

PR 작업 시 원본 README convention을 따릅니다.

- 기능/이슈 단위로 GitHub Issue 생성
- 이슈 번호 기반 브랜치 생성
- 브랜치 형식: `{issue number}-{prefix}-{issue-related-content}`
- 허용 prefix: `feature`, `fix`, `refactor`, `chore`, `docs`
- 커밋 형식: `{prefix}: {content}`
- 기능 브랜치에서 작업 후 `develop` 브랜치 대상으로 PR 생성
- 검증 완료 후 `main`으로 배포

예시:

```text
1-feature-session-taggroup-api
feature: implement session and tag group APIs
```

다음 PR 대상은 `JustInNeed/JustInNeed-BE`이며, PR 생성 시 위 convention에 맞춰 브랜치와 PR 흐름을 정리합니다.
