# Loop  - 콘서트 동행자 매칭 서비스

> 같은 공연을 함께 관람할 동행자를 찾고, 공연 정보까지 한곳에서 확인하는 서비스

팀 레포 [ManaP0tion/Loop_Backend](https://github.com/ManaP0tion/Loop_Backend)의 커밋 기록을 그대로 옮겨 온 개인 저장소로,<br>
백엔드 개발자 **추동원**의 담당 범위와 기술적 고민을 정리했습니다.

## 1. 프로젝트 개요

- **기간 :**&nbsp;  `2026.06 ~ 진행 중`
- **팀 :**&nbsp;  `PM 1`, `디자인 1`, `FE 1`, `BE 2`
- **상태 :**&nbsp;  `V1 운영 배포`, `V2 개발 서버 배포`
- **수상 :**&nbsp;  `스위프(SWYP) 웹 14기 대상`

## 2. 담당 업무

- **공연 데이터 수집 :**&nbsp; `KOPIS 수집 로직 리팩토링`, `식별·저장 책임 분리`
- **데이터 조회 :**&nbsp; `동행글 목록 N+1 개선`, `채팅방 생성 응답의 지연 로딩 해결`
- **채팅 이력 설계 :**&nbsp; `탈퇴·재가입 시 참여자별 메시지 노출 범위 설계`
- **인프라 :**&nbsp; `개발·운영 서버 분리`, `개발 서버 배포 워크플로우`, `Nginx·SSL 구성`, `빌드 단계 이관`
- **인증 :**&nbsp; `카카오 로그인`, `Refresh Token 저장 구조 개선`
- **협업 :**&nbsp; `API 변경 공지와 Swagger 명세 공유`, `QA 대응`

## 3. 기술 스택

- **Backend :** &nbsp; `Java 17` `Spring Boot 3.5` `Spring Security` `Spring Data JPA` `WebSocket(STOMP)` `JWT`
- **Database :** &nbsp; `MySQL 8` `Redis 7`
- **Infra :** &nbsp; `AWS EC2` `Docker Compose` `Nginx` `Let's Encrypt` `GitHub Actions`
- **External :** &nbsp; `Kakao OAuth` `KOPIS Open API` `iTunes API` `AWS S3` `AWS SES` `Discord Webhook`

## 4. 시스템 구조

### 4-1. 서비스 구조

![서비스 구조](https://github.com/user-attachments/assets/d8534c81-441c-43b6-8e82-ad935aa86f5a)

- 클라이언트 요청은 Nginx를 거쳐 Spring Boot로 들어오고, HTTPS는 Nginx에서 처리합니다.
- 서비스 데이터는 MySQL에 저장합니다.
- Refresh Token, 토큰 블랙리스트, 로그인 시도 제한, 이메일 인증, 관리자 2단계 인증처럼 수명이 짧은 데이터는 Redis에 둡니다.
- 소셜 로그인, 공연 정보 수집, 곡 정보, 이미지 저장, 메일, 운영 알림은 외부 서비스를 호출합니다.

### 4-2. 스케줄 작업

| 시각 (KST) | 작업 |
|---|---|
| 매일 03:05 | 기간이 끝난 이용정지 계정 자동 해제 |
| 매일 09:00 | KOPIS 공연 수집 |
| 매일 09:00 | 미확인 채팅 메일 발송 |
| 매일 10:00 | 다음 날 공연 관람 예정자에게 리마인드 메일 |

### 4-3. KOPIS 공연 데이터 수집

![KOPIS 공연 데이터 수집 흐름](https://github.com/user-attachments/assets/5ab61700-e4ed-4e16-8c54-613a0b77f56d)

- 매일 09시에 스케줄러가 KOPIS에서 공연을 가져옵니다.
- 제목으로 수집 대상을 가려내고, 검토 대기열까지만 저장합니다.
- 어드민이 승인하면 비공개 공연이 만들어지고, 값을 보완해 공개해야 사용자에게 노출됩니다.

외부 데이터가 검증 없이 바로 사용자에게 노출되지 않도록 사람의 확인 단계를 두었습니다.

### 4-4. 실시간 채팅

![실시간 채팅 구조](https://github.com/user-attachments/assets/f53f7425-b9df-4801-aec2-00c1bce63e65)

- 메시지 전송은 STOMP, 채팅방 생성·이력 조회·읽음 처리·나가기는 REST로 처리합니다.
- STOMP 연결 단계에서 인터셉터로 JWT를 검증합니다.
- 메시지는 트랜잭션 안에서 참여 상태·차단·탈퇴·퇴장 여부를 검증한 뒤 저장합니다.
- 저장된 메시지는 Simple Broker로 같은 방 구독자에게 전달됩니다.

채팅 기능은 팀원과 함께 구현했습니다.

### 4-5. 배포

![배포 구조](https://github.com/user-attachments/assets/115c10dc-137b-4cae-b9fd-50e12d7e639f)

| 브랜치 | 배포 대상 |
|---|---|
| `main` | 운영 서버 |
| `dev` | 개발 서버 |

- 빌드는 GitHub Actions, EC2는 실행만 담당합니다. 빌드된 JAR만 EC2로 전달합니다.
- EC2에서는 Docker Compose로 Nginx, Spring Boot, MySQL, Redis를 함께 띄웁니다.
- 민감 정보는 레포에 두지 않고 EC2의 `.env`와 GitHub Secrets로 관리합니다.
- 워크플로우는 원본 레포에서만 실행되도록 제한되어 있어, 이 저장소에서는 배포가 일어나지 않습니다.

## 5. 기술적 고민

| 주제 | 결과 | 블로그 |
|---|---|---|
| KOPIS 수집 로직의 책임 분리 | 식별·저장 분리, 불필요한 출연진 매칭 단계 제거 | [링크](https://chu-studyroom.tistory.com/135) |
| 동행글 목록 N+1 개선 | 쿼리 6회 → 1회 | [링크](https://chu-studyroom.tistory.com/125) |
| 채팅방 생성 응답의 지연 로딩 문제 | 프로젝션 조회로 트랜잭션 밖 지연 로딩 차단 | - |
| 탈퇴·재가입 시 채팅 이력 가시성 | 상대 이력 보존 + 본인 이전 대화 숨김 | [링크](https://chu-studyroom.tistory.com/128) |
| 개발·운영 환경 분리와 빌드 이관 | QA 환경 분리, EC2 메모리 부족 해결 | [링크](https://chu-studyroom.tistory.com/122) |
| 인증 | Refresh Token 저장 구조 개선 | [링크](https://chu-studyroom.tistory.com/120) |

### 5-1. KOPIS 수집 로직의 책임 분리

![KOPIS 수집 로직 분리 전후](https://github.com/user-attachments/assets/abf2c106-1033-49f2-977d-4a342322c28d)

- **문제:** `processPerformance()` 하나가 제목 식별, 상세 조회 판단, 출연진 매칭, 저장을 모두 처리했습니다. 공연 유형을 판단하는 기준도 여러 단계에 흩어져 있어, 규칙 하나를 바꾸려 해도 전체 흐름을 따라가야 했습니다.
- **개선:** 식별은 `PerformanceIdentifier`, 저장은 `ConcertImportWriter`로 분리했습니다. 기존 매칭 규칙은 유지하고, V2에서 쓰지 않는 국내 공연 로직은 별도 책임으로 분리했습니다.
- **검증:** 분리 전에 작성한 테스트를 기준으로 기존 동작이 유지되는지 확인했고, 식별 로직은 DB와 외부 API 없이 테스트할 수 있게 했습니다.
- **추가 개선:** 최근 공연 98건을 분석해 보니 출연진 정보가 있는 68건에서도 출연진 매칭 사례가 없어, 출연진 매칭 단계를 제거했습니다. 책임을 나눠 둔 덕분에 이 단계만 독립적으로 뺄 수 있었습니다.

### 5-2. 동행글 목록 N+1 개선

- **문제:** 지난 공연을 거르는 `isExpired()`가 목록의 글마다 LAZY로 연결된 `Concert`를 추가 조회했습니다.
- **해결:** 해당 목록 조회에만 `@EntityGraph(attributePaths = "concert")`를 적용했습니다. `EAGER`는 엔티티의 모든 조회에 영향을 주어 쓰지 않았고, 동적 검색 조건이나 컬렉션 연관관계가 있는 목록은 `@BatchSize`로 처리했습니다.
- **검증:** 서로 다른 공연을 가진 동행글 5건으로 테스트해 Hibernate Statistics와 SQL 로그를 확인했습니다.
- **결과:** 쿼리 6회(글 1 + 공연 5) → 1회(JOIN)

### 5-3. 채팅방 생성 응답의 지연 로딩 문제

- **배경:** 1:1 채팅방은 사용자 쌍당 하나만 있어야 해서, 사용자 쌍 기반 잠금 안에서 `TransactionTemplate`으로 커밋한 뒤 잠금을 푸는 구조입니다. 이 동시성 처리 구조는 팀원이 설계했습니다.
- **문제:** 트랜잭션이 끝난 뒤 응답 DTO를 만들면서 지연 로딩 연관관계에 접근해 `LazyInitializationException`이 발생했습니다.
- **제약:** 커밋 전에 잠금이 풀리면 중복 생성이 생기므로 트랜잭션 범위는 넓힐 수 없었습니다.
- **해결:** Fetch Join, 트랜잭션 안 직접 변환과 비교한 뒤, 필요한 값만 담는 `ChatRoomSummaryDto` 프로젝션으로 조회해 엔티티가 트랜잭션 밖으로 나가지 않게 했습니다.
- **한계:** 현재 잠금은 단일 서버에서만 유효해, 서버가 늘어나면 DB 유니크 제약이나 분산 락으로 바꿔야 합니다.

### 5-4. 회원 탈퇴·재가입 시 채팅 이력 가시성

- **문제:** 메시지를 지우면 상대방의 대화 이력까지 사라지고, 그대로 두면 재가입한 사용자에게 이전 대화가 다시 보입니다.
- **설계:** 참여자별 메시지 노출 기준 시각(`messageVisibleFrom`)을 두어, 상대방에게는 전체 이력을 보존하면서 재가입한 본인에게만 이전 대화를 숨겼습니다.
- **일관성:** 채팅방 목록의 마지막 메시지 미리보기와 안 읽은 메시지 수도 같은 기준으로 집계합니다.
- **판단:** 탈퇴 전용 참여 상태를 새로 만들지 않고 기존 나가기(`LEFT`) 상태를 재사용해 변경 범위를 줄였습니다. 상대방에게는 "나갔다"가 아니라 "탈퇴했다"는 문구가 보이도록, 참여 상태와 무관하게 상대 사용자를 조회합니다.

### 5-5. 개발·운영 환경 분리와 빌드 이관

![개발·운영 환경 분리](https://github.com/user-attachments/assets/5a848b2e-20fc-4284-a952-06f94734f6a1)

- **환경 분리:** 테스트 환경이 없어 PM의 테스트·QA가 운영 DB에 영향을 주고 있었습니다. 개발용 EC2를 따로 두고 서버·DB·설정을 운영과 분리해, 운영 데이터 걱정 없이 QA할 수 있게 했습니다.
- **담당 범위:** 개발 서버 배포 워크플로우와 Nginx·SSL 설정을 맡아, 환경마다 환경변수와 도메인에 맞춘 인증서를 따로 구성했습니다.
- **빌드 이관:** EC2에서 Gradle과 Docker 빌드를 함께 하다 보니 배포 중 메모리가 부족해 EC2가 응답을 멈췄습니다. 빌드를 GitHub Actions로 옮기고 JAR만 전달해, EC2는 실행만 담당하도록 바꿨습니다.

### 5-6. 인증

- 카카오 로그인과 JWT 발급·재발급 흐름을 구현했습니다.
- 토큰 저장 방식에 따른 보안 위험을 검토해 Refresh Token 저장 구조를 개선했습니다.

## 6. 협업 방식

- **FE 수정 범위를 줄이는 원칙:** 프론트엔드 개발자가 한 명이라, 요구사항이 바뀌면 기존 API 구조를 유지할 수 있는지 먼저 검토했습니다. 변경이 필요하면 FE가 고칠 범위를 정리해 Swagger 명세와 함께 먼저 공유했습니다.
- **API 변경 공지:** 변경 요약, 값의 규칙, 입력 규칙, 주의할 점과 한계를 정리해 FE가 바로 반영할 수 있게 공유했습니다.
- **QA:** 노션 체크리스트로 진행하고, PR과 배포 결과는 디스코드 알림 채널로 공유했습니다.

![API 변경 공지 형식](https://github.com/user-attachments/assets/188d24ad-ba41-4bc9-9c12-718f1f7945bc)

![QA 진행 흐름](https://github.com/user-attachments/assets/fa412619-7db7-48d3-bf59-4e511c389b53)

## 7. 패키지 구조

도메인별로 패키지를 나누고, 각 도메인 안에 `controller / service / repository / domain / dto`를 둡니다.

```
com.loop.loop_backend
├── auth            # 카카오 로그인, JWT, 관리자 2단계 인증
├── User            # 회원, 탈퇴·재가입, 이메일 인증
├── Concert         # 공연 조회, 리마인드
├── ConcertImport   # KOPIS 수집 (kopis / scheduler / service)
├── CompanionPost   # 동행글
├── Chat            # 채팅 (STOMP 인터셉터, 스케줄러 포함)
├── Admin           # 어드민, 이용 정지 관리
├── Report · Block · Inquiry
├── Artist · Lineup · Venue · Song · HashTag · FavoriteArtist · ConcertScrap · CompanionHeart
├── Storage · Mail · Notification
└── common · Config
```
