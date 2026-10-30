# W6 고정 Clock TDD와 W7 두 사례 재현 안내

## 현재 준비 상태

2026-10-29 요청에 따라 **주문 결제 시각에 Clock을 연결하고, 정상 발행·소비 1건과 발행 누락 1건을 고정 시각으로 다시 검증했다.** 재현 코드는 기존 [전체 흐름 테스트](../../apps/commerce-api/src/test/java/com/loopers/application/order/PgOrderPurchaseActivityEndToEndTest.java)를 사용한다. 새 Outbox·랭킹·재현 전용 제품은 만들지 않았다.

| 항목 | 상태 |
| --- | --- |
| 고정 Clock·품목·두 사례 준비/검증 코드 | 구현·실행 완료 |
| 실행 방법·실제 사건 사실·자원 정리 | 이 문서에 기록 |
| 이번 실행의 기반 HEAD | `6e686d824d979ad16a7a2baa70afbeee2c3ce460`. **현재 W6 변경이 포함되지 않은 기반 commit** |
| 검증한 완성 W6 commit | **아직 없음.** 사용자 요청으로 커밋하지 않았으며 미커밋 작업 트리에서 검증 |
| C09 남은 항목 | 코드가 담긴 실제 W6 commit이 생기면 그 해시와 검증 결과를 연결해야 함 |

[과제 원문](assignment-original.md)의 고정 clock·재현 코드 요구에 해당한다. 이 문서를 작성했다는 이유로 미정인 실행 revision까지 완료 처리하지 않는다. 이전 [정상 전체 흐름](normal-pg-purchase-activity-e2e-tdd-log.md)과 [발행 누락](pre-send-failure-e2e-tdd-log.md) 기록의 당시 시각·결과는 역사로 보존한다.

**2026-10-30 공통 발행 경로 변경:** 현재 재현 테스트의 발행 포트·실패 hook은 공통 `DomainEventPublisher`·`KafkaDomainEventPublisher`를 사용하고 실제 commit 뒤 `DomainEventListener`를 거친다. 기존 `OrderConfirmedEvent`의 식별·key·JSON·저장 시각/품목, 고정 Clock과 정상/누락 기대값을 유지한 채 두 사례를 재검증했다. 아래 2026-10-29 사건 값·247개 검사 기록은 당시 증거이며, 구조 변경 뒤의 재실행·사건 값·검사 범위는 [공통 발행 TDD 기록](common-domain-event-publication-tdd-log.md)을 따른다. 실행 명령과 미커밋 revision의 한계는 그대로다.

## 무엇을 바꿨는가

| 파일 | 변경 |
| --- | --- |
| [PaymentConfirmationConfiguration](../../apps/commerce-api/src/main/java/com/loopers/application/order/PaymentConfirmationConfiguration.java) | `orderPaymentClock`이라는 전용 `Clock.systemUTC()` 빈 제공 |
| [OrderPaymentStockService](../../apps/commerce-api/src/main/java/com/loopers/application/order/OrderPaymentStockService.java) | 생성자에서 같은 이름의 Clock을 받아 `startDispatch`와 `confirmPgPayment`에 전달할 현재 시각 생성 |
| [PgOrderPurchaseActivityEndToEndTest](../../apps/commerce-api/src/test/java/com/loopers/application/order/PgOrderPurchaseActivityEndToEndTest.java) | 해당 빈만 `Clock.fixed`로 교체. 두 주문의 저장 확정 시각과 발송 시작 시각, 정상 이벤트·구매 활동 시각 일치 assertion 추가 |

운영 기본 동작은 UTC 현재 시각이다. 테스트 고정값은 **`2026-10-29T00:00:00.123456Z`**로, 기존 모델·DB의 마이크로초 정밀도에 맞췄다. 두 사례의 확정 시각이 같아도 각각 별도의 주문·사건 ID로 구분한다.

고정한 것은 **결제 발송 시작과 최초 주문 확정의 업무 시각**이다. JPA의 생성·수정 감사 시각, 로그·broker 시각은 고정하지 않는다. PG의 대기 제한에 쓰는 `PaymentConfirmationTiming`/`System.nanoTime()`, 조회 회로의 시계와 Kafka timeout도 변경하지 않았다. DB 스키마·트랜잭션 전파/격리·기존 잠금 순서·예약/상태 전이·after-commit 발행 조건은 유지한다.

## 다시 실행하는 방법

JDK 21, 실행 중인 Docker, Gradle 의존성과 테스트 이미지가 준비된 환경에서 저장소 루트의 다음 명령을 실행한다. `--offline`은 의존성 캐시가 준비된 환경을 전제로 한다.

```shell
./gradlew :apps:commerce-api:test \
  --tests 'com.loopers.application.order.PgOrderPurchaseActivityEndToEndTest' \
  --rerun-tasks --offline --console=plain
```

`--rerun-tasks`는 같은 소스에서도 새 격리 사례를 실제로 실행하려는 옵션이다. 아래 기록된 최초/최종 검증에서는 테스트 입력 또는 선택 범위가 달라 실행됐으므로 이 옵션 없이 실행했다.

실행하는 메서드는 `normalDeliveryAndPreSendFailureKeepIndependentCommittedFacts`다. Gradle 테스트 설정이 streamer의 main classpath·DDL 경로를 전달하므로 수동으로 사용자 DB를 연결하거나 API/streamer 운영 앱을 먼저 켤 필요가 없다.

| 단계 | 코드가 실제로 하는 일 |
| --- | --- |
| 준비 | API 테스트 DB, 별도 streamer MySQL, 전용 Kafka와 UUID topic/group, 실제 로컬 PG HTTP 생성. 별도 JVM의 실제 streamer listener를 시작하고 배정 확인 |
| 정상 주문 | `draftThroughHttp`로 주문 생성 → confirm → PG SUCCESS → 확정 commit → 실제 Kafka 발행 → 구매 활동 공동 저장 → ack |
| 정상 비교 기준 | 상품별 수량·고정 시각·민감정보 제외, 완료 1건/활동 2건, broker 위치와 committed offset 확인. 동일 confirm 추가 효과 없음 |
| 누락 주문 | 다른 주문 생성/확정 → 실제 AFTER_COMMIT의 공통 publisher hook에서 별도 DB 연결로 확정 사실 확인 → Kafka adapter 호출 직전 예외 |
| 누락 비교 | 주문/예약/재고 보존, 해당 record·완료·활동 0. 정상 주문의 구매 활동 전체 snapshot·offset 불변 |
| 재요청 | 누락 주문의 confirm·callback·reconcile 반복. PG 추가 생성·재발행·재고 효과·최초 확정 시각 변경 없음 |
| 정리 | child streamer → 관찰 consumer/AdminClient → PG·테스트 DB/컨테이너 순서로 소유 자원 회수 |

정상 주문을 먼저 소비하는 것은 **실제 consumer가 동작하는데도 두 번째 주문의 활동은 없다는 비교 기준**을 마련하기 위해서다. 누락은 통제한 pre-send 예외이며 OS 강제 종료나 모든 네트워크 장애를 재현한 것으로 표현하지 않는다.

각 주문의 고정 품목 조건은 단가 100원인 A 3개/B 2개, 합계 500원이다. HTTP 입력에서 A 1개와 A 2개를 따로 보내 정규화도 검증한다. 초기 재고는 A 10/B 5이며 확정 후 7/3, 포인트는 1,000원으로 유지된다. 사용자 DB·기존 topic/group·volume은 사용하거나 변경하지 않는다.

## W7에서 이어받는 방법

1. W6 코드가 들어 있는 실제 commit을 보존하고, 위 실행 명령과 아래 두 사례의 역할·관찰값을 함께 전달한다. 현재는 커밋 금지 요청에 따라 **commit 해시 연결만 남긴 상태**다.
2. W7의 새 격리 환경에서 그 W6 직접 발행 경로와 테스트 준비 함수를 실행한다. 기존 W6 revision의 실행물을 활용하는 방식도 과제에서 허용한다. W7 Outbox를 먼저 실행하고 행을 지워 누락을 꾸미지 않는다.
3. 새 실행에서 실제 생성된 주문의 ID들을 얻고, **그 실행 안의 준비→복구 사이에 같은 ID와 최초 사실을 유지**한다. 아래 숫자를 그대로 사용한다고 원래 DB 행이 복원된 것은 아니다.
4. 이번 테스트는 관찰 뒤 자원을 정리한다. W7에서 같은 실행 내 복구까지 검증하려면 소유 fixture의 정리 전에 복구 검증을 이어 붙인다. 임시 DB·컨테이너를 일주일 유지하지 않는다.

이 문서는 W7의 복구 구현을 대신하지 않는다. 현재 누락 주문의 재요청은 기존 확정 상태를 반환하며 이벤트를 복구하지 않는다.

## 이번 최종 실행의 실제 사건 사실

실제 실행 시각은 **2026-10-29 16:14 KST**이며, 아래 `confirmedAt`은 주입한 업무 시각이다.

- topic: `order-events-e2e-a04ca32a-991f-494d-a2e9-6a16e080b159`
- streamer group: `purchase-activity-e2e-969b7d05-9270-496f-a6af-447818b7e4b8`

| 항목 | 정상 발행·소비 사례 | 발행 누락 사례 |
| --- | --- | --- |
| orderId | `1` | `2` |
| providerOrderId | `LP-ORD-0000000001` | `LP-ORD-0000000002` |
| transactionKey | `TEST-TX-1` | `TEST-TX-2` |
| eventId | `order-confirmed:v1:1` | `order-confirmed:v1:2` |
| confirmedAt / sendStartedAt | `2026-10-29T00:00:00.123456Z` | `2026-10-29T00:00:00.123456Z` |
| items | 상품 1×3, 상품 2×2 | 상품 3×3, 상품 4×2 |
| 주문·결제 / 예약 | CONFIRMED·SUCCESS / CONSUMED | CONFIRMED·SUCCESS / CONSUMED |
| PG 생성 / 거래 | 1회 / 1건 | 1회 / 1건 |
| Kafka | partition 1·offset 0 | 해당 record 없음 |
| 완료 / 구매 활동 | 1건 / 2건 | 0건 / 0건 |
| 재요청 결과 | 동일 confirm 효과 없음 | confirm·callback·reconcile 효과 없음 |

누락 hook에서 원래 connection `40`과 독립 읽기 connection `41`을 구분하고 commit된 사실을 확인했다. 정상 record의 committed offset은 `1`이며 누락·재요청 이후에도 group offset은 그대로였다. 두 partition의 endOffsets와 정상 주문의 구매 활동 PK/필드도 불변이었다. broker record가 없는 누락 사례에 가짜 partition/offset을 붙이지 않는다.

첫 Green인 16:12 KST 실행은 topic `order-events-e2e-d5b5bdcb-7686-44d1-8e26-f20389b02edd`, group `purchase-activity-e2e-184683ad-66e7-434e-b795-596bcba50762`에서 수행했고, 두 주문 모두 같은 고정 업무 시각으로 통과했다. 두 실행의 실제 사건은 서로 다른 격리 환경이며 동일 역사적 주문의 재전달로 합산하지 않는다.

## Red → Green과 관련 검사

| 단계 | 실제 결과 |
| --- | --- |
| Red | 고정 Clock 빈과 기대값만 테스트에 추가한 뒤 실행. 16:11 KST, 기대 `2026-10-29T00:00:00.123456Z` / 실제 `2026-10-29T07:11:18.765863Z`로 실패. 기존 서비스가 주입한 시계를 사용하지 않는 것을 확인. BUILD FAILED 30초·종료 1 |
| Green | 서비스 생성자·운영 빈·두 시간 생성 호출 연결 후 16:12 KST E2E 1개 통과. 실패/오류/skip 0, BUILD SUCCESSFUL 31초·종료 0 |
| 관련 검사 | 16:14 KST 아래 명령으로 실제 10개 스위트·247개 통과. 실패/오류/skip 0, BUILD SUCCESSFUL 58초·종료 0 |

```shell
./gradlew :apps:commerce-api:test \
  --tests 'com.loopers.application.order.PgOrderPurchaseActivityEndToEndTest' \
  --tests 'com.loopers.application.order.PgOrderEventKafkaIntegrationTest' \
  --tests 'com.loopers.application.order.OrderConfirmedPublicationIntegrationTest' \
  --tests 'com.loopers.application.order.OrderPaymentStockIntegrationTest' \
  --tests 'com.loopers.application.order.OrderPaymentStockRollbackTest' \
  --tests 'com.loopers.application.order.OrderPaymentStockConcurrencyTest' \
  --tests 'com.loopers.domain.order.OrderPaymentStateTest' \
  --tests 'com.loopers.domain.payment.PaymentAttemptTest' \
  --tests 'com.loopers.architecture.ArchitectureTest' \
  :apps:commerce-streamer:test \
  --tests 'com.loopers.architecture.ArchitectureTest' \
  :apps:commerce-api:checkstyleMain :apps:commerce-api:checkstyleTest \
  :apps:commerce-streamer:checkstyleMain :apps:commerce-streamer:checkstyleTest \
  --offline --console=plain
```

| 검사 범위 | 통과 수 |
| --- | ---: |
| 전체 흐름 / PG Kafka / commit 경계 발행 | 1 / 6 / 12 |
| 재고 통합 / 실제 SQL 뒤 롤백 / 재고 경합·실제 잠금 순서 | 47 / 2 / 7 |
| 주문 결제 상태 / 결제 시도 | 106 / 64 |
| API / streamer ArchUnit | 1 / 1 |
| 합계 | **247** |

API는 9개 스위트·246개, streamer는 ArchUnit 1개 스위트·1개를 실제 실행했다. streamer의 업무 처리는 E2E의 별도 production JVM으로도 실행했다. 양 앱 Checkstyle XML 4개는 위반 0이며 API main/test는 실제 실행, streamer main/test는 미변경으로 UP-TO-DATE였다.

고정 시각만 반복하면 시각 덮어쓰기를 놓칠 수 있어, 기존 [OrderPaymentStateTest](../../apps/commerce-api/src/test/java/com/loopers/domain/order/OrderPaymentStateTest.java)의 하루 뒤 재확정·최초 시각 보존과 [PaymentAttemptTest](../../apps/commerce-api/src/test/java/com/loopers/domain/payment/PaymentAttemptTest.java)의 다른 시각 재발송·최초 발송 시각 보존도 함께 실행했다. 이전 기대값을 삭제하거나 완화하지 않았다.

테스트 child의 정상 listener/DB 종료와 실행 중 Testcontainers 컨테이너 0개를 확인했다. [해당 child 로그](../../apps/commerce-api/build/reports/w6-e2e/order-events-e2e-a04ca32a-991f-494d-a2e9-6a16e080b159/streamer.log)·JUnit 보고서는 후속 실행으로 바뀔 수 있으므로 실제 사건 사실은 이 문서에 남긴다. 카드·비밀은 재현 기록에 넣지 않는다.

이번 완료는 고정 Clock 연결·두 사례 재검증·W7 재현 안내다. 실제 W6 commit 연결, W6 전체 11항목의 최종 대조, PR·기술 글·제출은 후속이며 커밋·푸시하지 않았다.
