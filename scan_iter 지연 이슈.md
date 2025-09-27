## SCAN 

```CAN cursor [MATCH pattern] [COUNT count]```

`cursor`

- 반복 위치를 나타내는 숫자
- 시작은 0, 반복 종료 시 응답으로 cursor=0이 나오면 완료

`MATCH`

- 패턴을 통해 Like 조회를 지원

`COUNT`

- 기본값은 **10**
- COUNT의 크기를 통해 분할 건수 조정 가능

## KEY

```KEYS pattern```

- 지정된 패턴에 일치하는 모든 키를 한 번에 반환
- Redis는 단일 스레드로 동작하기 때문에, KEYS 처럼시간이 오래 걸리는 작업이 실행되면 그 작업이 완료될 때까지 다른 클라이언트 명령어를 처리할 수 없음 → 블로킹
- 운영 환경에서 사용 지양 → SCAN 명령어를 사용하는 것을 권장

## 문제 원인

> 예시 코드 

```python
def get_reservation_count(self):
    """
    예약된 사용자 수 계산
    """
    redis_client = cache.client.get_client(write=True)
    reservation_count = sum(1 for _ in redis_client.scan_iter(f"*{self.reservation_prefix}*"))
    return reservation_count
```

**❓ Reservation Key를 사용한 이유**

- 선착순 이벤트에서 100명 제한을 두고 있기 때문에, 현재 예약된 사용자 수를 Redis 키로 관리하고 그 수를 집계하기 위해 **scan_iter**를 사용함
    - 99명이 참여한 상황에서 N명이 동시에 참여하는 경우 모두가 100원의 혜택을 받게되므로, 콘서트 자리 선점처럼 reservation_key을 가진 유저가 100원 혜택을 받을 수 있도록 하기 위해 사용 (TTL 30초)
- reservation_user_id_1234 → 유저 id를 포함한 키를 생성하여 해당 유저의 예약 정보를 저장

😢 **성능상 이슈**

- 캐시 서버에 약 14만 개 이상의 키가 존재
    - count 값을 주지않으면 default count값은 10이기 때문에, 약 14,000번의 호출이 일어나게 됨
- 해당 코드가 메인 화면에서 짧은 주기로 반복 호출되어 scan 지연이 발생했음 
- reservation_prefix가 명확한 접두사/접미사가 아니라 **중간에 위치하는 경우**, 더 많은 탐색 비용 O
    - 캐시 서버에 저장될때 `:1:`가 접두어로 같이 저장되어서 앞뒤로 와일드카드(*)를 적용했었음

> 변경 코드

```python
redis_client.scan_iter(f":1:{self.reservation_prefix}*", count=10_000)
```

- 적절한 count값 제공 + 접두어 `:1:`를 추가

> 생각해본 개선 방향

```python
reservation_key = f"event:{today_date}:reservation_user_ids"

redis_client.sadd(reservation_key, user_id)       # 예약 추가
redis_client.sismember(reservation_key, user_id)  # 예약 여부 확인
redis_client.scard(reservation_key)               # 총 예약자 수
redis_client.expire(reservation_key, TTL)
```

- `scard`
    - 예약자 수 확인
    - Set의 크기를 즉시 반환
- `sismember`
    - 유저 예약 여부 확인
    - 해당 요소가 Set에 포함되어 있는지 확인
- `sadd`
    - 예약 추가
    - 동일한 값이 이미 존재하면 추가하지 않고 무시
- 모두 시간복잡도 **O(1)**
- 하지만 개별 유저의 예약키의 TTL을 설정할 수 없는 문제점 발생
