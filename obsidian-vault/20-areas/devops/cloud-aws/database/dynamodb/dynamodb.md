---
title: "AWS DynamoDB — Managed NoSQL"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:50:00+09:00
tags:
  - aws
  - database
  - dynamodb
  - nosql
---

# AWS DynamoDB

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | DynamoDB 개념 + 사용 |

**[[database|↑ Database]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **key-value / document NoSQL**. 무제한 확장 + single-digit ms latency + serverless.

---

## 2. 왜

- **RDS** = 좋지만 단일 노드 한계 (수만 TPS+ 어려움)
- DynamoDB = AWS 가 운영, 응용은 key 설계만
- **사용한 만큼** 과금 (on-demand) 또는 provisioned RCU/WCU

특징:
- single-digit ms latency
- 거대 throughput (수백만 RPS 가능)
- multi-region (Global Table)
- DynamoDB Streams (CDC)
- TTL (자동 만료)

한계:
- query 제약 (PK / SK only, 또는 GSI)
- JOIN X
- 트랜잭션 제한적
- 비용 모델 함정

대안:
- **MongoDB / DocumentDB** — document
- **Cassandra / Scylla / Keyspaces** — wide-column
- **RDS / Aurora** — RDB
- **Redis** — cache / 작은 KV

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Table** | 컬렉션 |
| **Item** | row 비슷 (max 400 KB) |
| **Attribute** | 필드 (typed: S/N/B/SS/NS/BS/M/L/BOOL/NULL) |
| **Primary Key** | partition key (PK) + 선택 sort key (SK) |
| **GSI / LSI** | secondary index (다른 키로 query) |
| **RCU / WCU** | Read / Write Capacity Unit (provisioned 모드) |
| **DAX** | DynamoDB Accelerator — in-memory cache |
| **Streams** | 변경 stream (CDC) |
| **TTL** | 자동 만료 |
| **Global Table** | multi-region 자동 replication |

---

## 4. Primary Key

### 4.1 partition key only (단순 hash)
```
PK = user_id
→ unique. 같은 PK 하나만.
```

### 4.2 partition + sort (composite)
```
PK = user_id
SK = timestamp
→ 같은 user_id 의 여러 row, SK 로 범위 쿼리.
```

→ 데이터 모델링이 핵심. 자세히 → [[data-modeling]] (비슷한 원리).

---

## 5. 설치 / 사용

### 5.1 CLI

```bash
# 테이블 생성
aws dynamodb create-table \
  --table-name UserEvents \
  --attribute-definitions \
    AttributeName=user_id,AttributeType=S \
    AttributeName=event_time,AttributeType=S \
  --key-schema \
    AttributeName=user_id,KeyType=HASH \
    AttributeName=event_time,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

# item put
aws dynamodb put-item --table-name UserEvents \
  --item '{"user_id":{"S":"alice"},"event_time":{"S":"2026-05-14T10:00"},"type":{"S":"login"}}'

# get
aws dynamodb get-item --table-name UserEvents \
  --key '{"user_id":{"S":"alice"},"event_time":{"S":"2026-05-14T10:00"}}'

# query (같은 partition)
aws dynamodb query --table-name UserEvents \
  --key-condition-expression "user_id = :uid AND event_time > :t" \
  --expression-attribute-values '{":uid":{"S":"alice"},":t":{"S":"2026-05-01"}}'

# scan (전체 — 비쌈)
aws dynamodb scan --table-name UserEvents
```

### 5.2 SDK (Python)

```python
import boto3
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("UserEvents")

table.put_item(Item={
    "user_id": "alice",
    "event_time": "2026-05-14T10:00",
    "type": "login"
})

resp = table.get_item(Key={"user_id":"alice","event_time":"2026-05-14T10:00"})
print(resp["Item"])

# query
from boto3.dynamodb.conditions import Key
resp = table.query(
    KeyConditionExpression=Key("user_id").eq("alice") & Key("event_time").gt("2026-05-01")
)
for item in resp["Items"]:
    print(item)
```

### 5.3 Terraform

```hcl
resource "aws_dynamodb_table" "events" {
  name           = "UserEvents"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "user_id"
  range_key      = "event_time"

  attribute { name = "user_id"    type = "S" }
  attribute { name = "event_time" type = "S" }

  ttl {
    enabled        = true
    attribute_name = "expire_at"
  }

  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  point_in_time_recovery { enabled = true }
}
```

---

## 6. 가격 모델

### 6.1 On-Demand (PAY_PER_REQUEST)
```
Write:  $1.25 / 1M
Read:   $0.25 / 1M (eventually consistent)
        $0.50 / 1M (strongly consistent)
```

→ 변동 / 시작 / 작은 워크로드.

### 6.2 Provisioned (RCU / WCU)
```
RCU: 1 strongly consistent read of 4 KB / sec
WCU: 1 write of 1 KB / sec
$0.0001425 / RCU·시간, $0.000713 / WCU·시간 (Seoul)
+ auto-scaling
```

→ 예측 가능 + 큰 부하 = 더 쌈.

### 6.3 Storage
```
$0.25 / GB·월
```

---

## 7. GSI / LSI

### 7.1 LSI — Local Secondary Index
- 같은 PK + 다른 SK
- 테이블 생성 시만 추가 가능
- 5 개 한계

### 7.2 GSI — Global Secondary Index
- 다른 PK + 다른 SK
- 언제든지 추가
- 20 개 (한계 증가 가능)
- 비동기 — eventual consistent

```bash
aws dynamodb update-table --table-name UserEvents \
  --attribute-definitions AttributeName=type,AttributeType=S \
  --global-secondary-index-updates '[{
    "Create": {
      "IndexName":"type-index",
      "KeySchema":[{"AttributeName":"type","KeyType":"HASH"}],
      "Projection":{"ProjectionType":"ALL"}
    }
  }]'
```

---

## 8. DynamoDB Streams + Lambda

```python
def lambda_handler(event, context):
    for r in event["Records"]:
        op = r["eventName"]                    # INSERT / MODIFY / REMOVE
        new = r["dynamodb"].get("NewImage")
        # → ES 인덱스 / aggregation / 다른 표 업데이트
```

→ CDC, denormalization, real-time analytics.

---

## 9. TTL

```python
import time
table.put_item(Item={
    "user_id": "alice",
    "session_id": "...",
    "expire_at": int(time.time()) + 3600        # 1시간 후 자동 삭제
})
```

→ session / cache / 임시 데이터. 무료.

---

## 10. Transactions

```python
client.transact_write_items(TransactItems=[
    {"Update": {"TableName":"Accounts","Key":{"id":{"S":"a"}},"UpdateExpression":"SET balance = balance - :v","ExpressionAttributeValues":{":v":{"N":"100"}}}},
    {"Update": {"TableName":"Accounts","Key":{"id":{"S":"b"}},"UpdateExpression":"SET balance = balance + :v","ExpressionAttributeValues":{":v":{"N":"100"}}}},
])
```

→ ACID — but 2x 비용. 25 items / 4 MB 한계.

---

## 11. Global Table

```bash
aws dynamodb create-global-table --global-table-name UserEvents \
  --replication-group RegionName=ap-northeast-2 RegionName=us-east-1
```

→ multi-region 자동 양방향 replication. eventual consistency. global app.

---

## 12. DAX (DynamoDB Accelerator)

```
응용 → DAX cluster → DynamoDB
       (μ초 캐시)
```

read-heavy 워크로드. eventually consistent 만.

---

## 13. 사용 시나리오

- 세션 / cart / leaderboard
- IoT / 시계열
- 거대 KV (사용자 / 디바이스)
- gaming
- ad tech
- Lambda 와 결합 — serverless stack

부적합:
- 복잡 JOIN / SQL
- ad-hoc analytics (RDS / Redshift)
- 작은 / 단순 (RDS 가 쉬움)

---

## 14. 함정

### 14.1 Hot partition
한 PK 에 거대 트래픽 → throttle. PK 설계 (suffix / hash prefix).

### 14.2 Scan 남용
table 전체 read = 거대 비용. query / GSI.

### 14.3 item 400 KB 한계
큰 데이터 → S3 + 포인터.

### 14.4 strongly consistent read = 2x RCU
약한 일관성 OK 면 eventual.

### 14.5 GSI 의 비용
별도 RCU/WCU. 거의 새 테이블 비용.

### 14.6 transactional 2x
trans 횟수 / 비용 모니터링.

### 14.7 on-demand vs provisioned
spike 패턴이면 on-demand. 안정 = provisioned 더 싸다.

### 14.8 query 의 1 MB / 100 item 한계
pagination 필수.

---

## 15. 학습 자료

- AWS DynamoDB docs
- **DynamoDB Book** — Alex DeBrie (필독)
- **NoSQL Workbench** — 모델링 도구
- **The DynamoDB Guide** — dynamodbguide.com

---

## 16. 관련

- [[database]] — DB hub
- [[rds]] / [[aurora]] — RDB 비교
- [[lambda]] — serverless stack
- [[cassandra|↗ Cassandra 비교]]
