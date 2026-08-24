# Java Map은 어디까지 저장소가 될 수 있을까?

## 학습 흐름

| 순서 | Java에서 읽고 구현할 것 | 다음 단계로 넘어가는 질문 |
| --- | --- | --- |
| 1 | `HashMap`의 조회·삽입·충돌·리사이즈 | 여러 스레드가 동시에 수정하면? |
| 2 | `synchronizedMap`과 `ConcurrentHashMap` | 안전한 메서드를 조합해도 안전할까? |
| 3 | `LinkedHashMap`으로 작은 LRU 캐시 구현 | 읽을 때마다 순서를 바꾸면 동시성은? |
| 4 | Caffeine의 데이터 저장과 정책 관리 | 어떤 정보는 정확해야 하고, 어떤 정보는 늦게 반영해도 될까? |
| 5 | Java로 두 키 갱신과 파일 복구 실험 | 여러 변경을 묶고, 중간에 죽어도 복구하려면? |

특히 중간에 LinkedHashMap을 넣는 게 중요해요. 직접 단순한 캐시를 만들어봐야 Caffeine 코드가 복잡해진 이유를 이해하기 쉬워집니다.

처음에는 다음처럼 진행하면 좋겠어요.

## 1. HashMap에서는 put 한 번을 끝까지 따라가기

`hash → 버킷 선택 → 키 비교 → 삽입/교체 → resize` 흐름을 읽고, 같은 해시를 반환하는 키와 삽입 후 값이 바뀌는 키로 실험해보세요.

여기서 얻을 것은 상수 암기보다 “키를 어떻게 찾아가고, 어떤 조건에서 탐색 비용이 커지는가”예요. 이것이 나중에 인덱스의 접근 경로를 이해하는 출발점이 됩니다. 다만 HashMap의 트리 버킷을 DB의 B-tree와 동일시하면 안 되고요.

참고: [OpenJDK 21 HashMap 소스](https://raw.githubusercontent.com/openjdk/jdk21u/master/src/java.base/share/classes/java/util/HashMap.java)

## 2. ConcurrentHashMap에서는 ‘원자적인 범위’를 확인하기

같은 카운터를 여러 스레드가 갱신하게 해서 아래 둘을 비교해요.

```java
map.put(key, map.get(key) + 1);

map.compute(key, (k, value) -> value + 1);
```

그다음 키를 두 개로 늘려서 계좌 이체처럼 만들어보면 됩니다.

```java
balances.compute("A", (k, v) -> v - 100);
balances.compute("B", (k, v) -> v + 100);
```

각 호출의 원자성과 두 호출 전체의 원자성은 별개예요. 다른 스레드의 조회를 두 호출 사이에 끼워 넣으면 **“왜 트랜잭션이라는 추가 개념이 필요한가”**가 Java 코드에서 바로 나옵니다. ConcurrentHashMap도 맵 전체에 대한 원자적 스냅샷을 일반적으로 보장하지 않습니다.

참고: [OpenJDK 21 ConcurrentHashMap 소스](https://raw.githubusercontent.com/openjdk/jdk21u/master/src/java.base/share/classes/java/util/concurrent/ConcurrentHashMap.java)

## 3. Caffeine에서는 ‘조회도 내부적으로 쓰기 작업인가’를 파보기

접근 순서 기반 LRU는 `get()`을 해도 최근 사용 순서를 바꿔야 해요. 그러면 조회가 많은 상황에서도 공유 자료구조를 계속 수정하게 되죠.

Caffeine은 내부적으로 ConcurrentHashMap을 활용하면서 캐시 정책 관리 작업을 버퍼에 기록하고 묶어서 처리합니다.

특히 정책 최적화에 쓰이는 일부 읽기 기록은 유실을 허용하지만, 쓰기 기록은 그렇게 취급하지 않아요. 데이터의 정확성과 교체 정책의 정확도에 서로 다른 요구를 둔다는 점이 재미있는 부분입니다.

참고: [Caffeine 설계 문서](https://github.com/ben-manes/caffeine/wiki/Design)

여기서 동시성, 경합, 배치 처리, 메모리 가시성이 한꺼번에 연결돼요. 기존에 정리한 JMM이나 CPU 캐시 관련 내용을 실제 라이브러리가 왜 사용하는지도 보이고요.

## 확장: Java의 작은 저장소에서 DB로

그리고 DB로 이어가고 싶어졌을 때, Java로 작은 저장소에 다음 조건을 하나씩 붙이는 거예요.

- 두 키를 갱신하는 도중 예외가 나면 원래 상태로 되돌리기.
- 다른 스레드가 중간 상태를 읽지 못하게 하기.
- 프로세스를 종료하고 다시 실행해도 데이터를 복구하기.

이 단계에서 잠금, undo, 로그, commit 기록의 필요성을 직접 발견할 수 있어요. Caffeine의 정책 처리 버퍼는 메모리 안의 관리 장치라서, 장애 복구용 WAL과는 보장이 다르다는 점도 비교할 수 있고요.

## 첫 학습 범위와 진행 방식

그래서 다음 주제로 Map 구현 비교를 택하는 데 동의해요. 첫 범위는 HashMap → ConcurrentHashMap → 직접 만든 LRU → Caffeine까지가 적당하겠습니다. 소스를 처음부터 전부 읽기보다는, 매 단계에서 **“내가 만든 단순한 구현의 어떤 문제를 이 코드가 해결하는가?”**를 하나씩 확인하는 방식이 지금 원하는 공부에 잘 맞아요.
