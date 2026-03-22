# 뉴스 피드 (Twitter) 설계

> 출처: https://www.sysdesai.com/learn/case-studies/news-feed

---

## 문제 정의 (Problem Statement)

뉴스 피드는 사용자에게 그들이 팔로우하는 계정의 게시물을 개인화되고 순위가 매겨진 스트림으로 보여줍니다. Twitter/X, Instagram, LinkedIn 모두 이 과제에 직면해 있습니다. 핵심적인 어려움은 유명인(팔로워 1억 명)의 게시물 하나가 효율적으로 전파(fan-out)되어야 하는 동시에, 사용자의 피드 로딩은 빠르고(< 200ms) 비교적 최신 상태여야 한다는 점입니다.

## 요구사항 (Requirements)

- 사용자는 게시물(텍스트, 이미지, 링크)을 게시할 수 있습니다.
- 사용자는 팔로우하는 계정의 게시물로 구성된 순위가 매겨진 피드를 봅니다.
- 피드는 페이지네이션(pagination)을 지원하며, 커서 기반 페이지네이션(cursor-based pagination)을 통한 무한 스크롤을 제공합니다.
- 거의 실시간(Near-real-time): 새로운 게시물은 몇 초 내에 나타나야 합니다.
- 2억 명의 DAU, 하루 500만 개의 게시물, 하루 2억 건의 피드 조회를 지원합니다.
- 좋아요, 리트윗, 댓글 수는 피드 카드에 표시됩니다.

## 핵심 트레이드오프: Fan-Out 전략 (The Core Trade-Off: Fan-Out Strategy)

Alice가 게시물을 올리면 그녀의 팔로워 1,000명이 이를 볼 수 있어야 합니다. 두 가지 근본적인 접근 방식이 있으며, 이 선택이 전체 아키텍처를 결정합니다:

| 전략 (Strategy) | 작동 방식 | 쓰기 비용 (Write Cost) | 읽기 비용 (Read Cost) | 적합한 경우 |
| --- | --- | --- | --- | --- |
| Fan-out on Write (Push) | 게시 시점에 모든 팔로워의 피드 캐시에 게시물 ID 추가 | 게시물당 O(팔로워 수) | O(1) — 미리 구축된 피드 | 일반 사용자 (팔로워 1만 명 미만) |
| Fan-out on Read (Pull) | 피드 로드 시점에 팔로우하는 사용자들의 게시물을 병합 | 게시물당 O(1) | 조회당 O(팔로우 수) | 유명인 계정 (팔로워 100만 명 이상) |
| Hybrid | 일반 사용자는 Push; 유명인은 읽기 시점에 Pull하여 병합 | O(일반 팔로워 수) | O(팔로우하는 유명인 수) | 실제 운영 시스템 (Twitter, Instagram) |

> ⚠️
> 유명인 문제 (The Celebrity Problem)
> 수백만 명의 팔로워를 가진 계정의 경우 순수 Fan-out on write 방식은 한계가 있습니다. 트윗 하나에 대해 1억 개의 피드 항목을 작성하는 데는 몇 분이 걸리며 메시지 큐가 폭주할 수 있습니다. 하이브리드 접근 방식은 유명인의 게시물을 Push하지 않고, 대신 피드 로드 시점에 미리 구축된 피드 캐시와 사용자가 팔로우하는 주요 유명인의 게시물을 Pull하여 병합합니다.

## 상위 수준 아키텍처 (High-Level Architecture)
하이브리드 Fan-out 뉴스 피드 아키텍처 (Hybrid fan-out news feed architecture)

## 피드 저장소: Redis Sorted Sets (Feed Storage: Redis Sorted Sets)

각 사용자의 피드는 `feed:{userId}`를 키로 하는 **Redis Sorted Set**에 저장됩니다. 여기서 Score는 게시물의 타임스탬프나 랭킹 점수이며, Value는 `postId`입니다. 피드 로드 시 `ZREVRANGE feed:{userId} 0 19`를 실행하여 상위 20개의 게시물 ID를 가져온 다음, 게시물 캐시에서 상세 내용을 일괄 조회(batch-fetch)합니다.

python

```python
# Fan-out worker: 팔로워 피드에 게시물 Push
def fanout_post(post_id: str, author_id: str, timestamp: float, followers: list[str]):
    pipe = redis.pipeline()
    for follower_id in followers:
        feed_key = f"feed:{follower_id}"
        pipe.zadd(feed_key, {post_id: timestamp})
        pipe.zremrangebyrank(feed_key, 0, -501)  # 상위 500개 게시물만 유지
    pipe.execute()

# Feed read: 상위 20개 게시물 ID를 가져온 후 데이터 채우기(hydrate)
def get_feed(user_id: str, cursor: float = "+inf", limit: int = 20):
    feed_key = f"feed:{user_id}"
    post_ids = redis.zrevrangebyscore(feed_key, cursor, "-inf", start=0, num=limit)
    posts = batch_get_posts(post_ids)  # 게시물 캐시 / DB에서 조회
    return posts
```

## 피드 랭킹 (Feed Ranking)

단순히 시간순으로 나열된 피드는 간단하지만 매력적이지 않습니다. 현대적인 시스템은 최신성, 참여도(좋아요, 리트윗, 댓글), 작성자와의 친밀도(해당 인물과 얼마나 자주 상호작용하는지), 콘텐츠 유형 선호도를 결합한 **신호 가중치 점수(signal-weighted score)**를 기준으로 게시물의 순위를 매깁니다. 인터뷰에서는 ML을 직접 구현할 필요는 없으며, Sorted Set의 점수가 랭킹 모델의 결과값일 수 있고 캐시된 피드에 대해 주기적으로 재계산될 수 있다는 점만 언급하세요.

## 게시물 발행 흐름 (Post Publishing Flow)
게시물 생성 및 팔로워 피드 캐시로의 Fan-out (Post creation and fan-out to follower feed caches)

## 확장성 고려 사항 (Scaling Considerations)

- **Fan-out 처리량**: Kafka에서 소비하는 Fan-out Worker 풀을 사용합니다. 팔로워가 100만 명인 게시물 하나는 100만 번의 Redis 쓰기를 트리거하므로, `followerId`를 기준으로 파티셔닝하여 병렬 처리합니다.
- **피드 캐시 TTL**: 메모리 확보를 위해 비활성 사용자의 피드는 30일 후 Redis에서 제거합니다. 다음 로그인 시 재구성합니다.
- **게시물 DB 샤딩**: 쓰기 분산을 위해 `posts` 테이블을 `authorId` 기준으로 샤딩합니다. `(authorId, postId)`를 Partition Key로 사용하는 Cassandra가 적합합니다.
- **미디어 CDN**: 이미지와 비디오는 S3에 저장하고 CDN을 통해 서빙합니다. 피드에는 바이너리 데이터가 아닌 참조(URL)만 저장합니다.
- **카운터 비정규화 (Counter denormalization)**: 좋아요/댓글 수는 미리 계산되어 Redis 카운터에 캐싱됩니다. 매번 게시물 테이블에서 쿼리하는 대신 스트림 처리를 통해 업데이트합니다.

> 💡
> 인터뷰 팁 (Interview Tip)
> 이 문제에서 가장 중요한 결정은 Fan-out 전략입니다. 하이브리드 접근 방식을 초기에 언급하고, 유명인 임계값(예: 팔로워 100만 명 이상)을 설명하며, 피드 읽기 경로에서 미리 구축된 피드 캐시와 실시간 유명인 게시물 Pull이 어떻게 병합되는지 보여주세요. 면접관은 대규모 환경에서의 쓰기 증폭(write amplification)에 대해 추론할 수 있는지 테스트합니다.

이 패턴 연습하기
[Design a Twitter-like news feed system](https://www.sysdesai.com/design/new?prompt=Design%20a%20Twitter-like%20news%20feed%20system&mode=fast)
