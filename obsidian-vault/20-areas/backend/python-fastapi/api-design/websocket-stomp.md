---
title: "WebSocket — FastAPI native"
kind: knowledge
status: current
created_at: 2026-05-27T00:45:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, websocket]
---

# WebSocket — FastAPI native

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/websocket-stomp|↗ Java 원본]]** · **[[../../python-flask/api-design/websocket-stomp|↗ Flask 변형]]**

> FastAPI 는 **native WebSocket** 지원 (ASGI 라 자연스러움). Flask 가 Socket.IO 인 것과 다름.

---

## 1. WebSocket vs STOMP vs Socket.IO

(Flask 변형 [[../../python-flask/api-design/websocket-stomp#1|§1]] 와 동일 — FastAPI 는 **raw WebSocket** 권장)

---

## 2. 의존성 — 없음 (FastAPI 기본)

```bash
# 이미 fastapi[standard] 에 포함
# pip install websockets  # 옵션
```

---

## 3. 기본 WebSocket endpoint

```python
from fastapi import WebSocket, WebSocketDisconnect, Depends
from typing import Annotated

from app.common.jwt import decode_token
from app.errors import UnauthorizedException


@router.websocket("/ws")
async def websocket_endpoint(
    websocket: WebSocket,
    token: str | None = None,                   # query parameter 또는 first message
):
    # 1. JWT 검증
    if not token:
        await websocket.close(code=4401, reason="missing token")
        return
    try:
        claims = decode_token(token)
        user_id = int(claims["sub"])
    except Exception:
        await websocket.close(code=4401, reason="invalid token")
        return

    await websocket.accept()

    try:
        while True:
            data = await websocket.receive_json()
            # echo
            await websocket.send_json({"echo": data, "user_id": user_id})
    except WebSocketDisconnect:
        log.info("ws disconnect user=%d", user_id)
```

> **FastAPI idiom**: `@router.websocket("/path")` decorator. `accept()` / `receive_json()` / `send_json()` 모두 async.

---

## 4. ConnectionManager — 다중 클라이언트 broadcast

```python
from typing import Set
import asyncio


class ConnectionManager:
    def __init__(self) -> None:
        # user_id → set of WebSocket
        self.user_connections: dict[int, set[WebSocket]] = {}
        # room_id → set of (user_id, WebSocket)
        self.room_connections: dict[str, set[WebSocket]] = {}
        self.lock = asyncio.Lock()

    async def connect_user(self, user_id: int, ws: WebSocket) -> None:
        async with self.lock:
            self.user_connections.setdefault(user_id, set()).add(ws)

    async def disconnect_user(self, user_id: int, ws: WebSocket) -> None:
        async with self.lock:
            if user_id in self.user_connections:
                self.user_connections[user_id].discard(ws)
                if not self.user_connections[user_id]:
                    del self.user_connections[user_id]

    async def join_room(self, room_id: str, ws: WebSocket) -> None:
        async with self.lock:
            self.room_connections.setdefault(room_id, set()).add(ws)

    async def leave_room(self, room_id: str, ws: WebSocket) -> None:
        async with self.lock:
            if room_id in self.room_connections:
                self.room_connections[room_id].discard(ws)

    async def send_to_user(self, user_id: int, message: dict) -> None:
        connections = self.user_connections.get(user_id, set())
        await asyncio.gather(
            *[ws.send_json(message) for ws in connections],
            return_exceptions=True,
        )

    async def broadcast_to_room(self, room_id: str, message: dict) -> None:
        connections = self.room_connections.get(room_id, set())
        await asyncio.gather(
            *[ws.send_json(message) for ws in connections],
            return_exceptions=True,
        )


manager = ConnectionManager()
```

> **`asyncio.gather` + `return_exceptions=True`** — 일부 ws 가 죽어도 나머지에 전송.

---

## 5. Chat 예시

```python
@router.websocket("/ws/chat/{room_id}")
async def chat_ws(
    websocket: WebSocket,
    room_id: str,
    token: str | None = None,
):
    if not token:
        await websocket.close(code=4401)
        return
    try:
        claims = decode_token(token)
        user_id = int(claims["sub"])
    except Exception:
        await websocket.close(code=4401)
        return

    # 권한 검증 — 방 멤버인지
    async with AsyncSessionLocal() as db:
        if not await chat_service.is_member(db, user_id, room_id):
            await websocket.close(code=4403, reason="not a member")
            return

    await websocket.accept()
    await manager.join_room(room_id, websocket)

    try:
        while True:
            data = await websocket.receive_json()
            content = data.get("content", "")
            if not content:
                await websocket.send_json({"error": "empty"})
                continue

            # DB 저장
            async with AsyncSessionLocal() as db:
                message = await chat_service.send(db, user_id, room_id, content)

            # broadcast
            await manager.broadcast_to_room(room_id, {
                "messageId": message.id,
                "senderId": user_id,
                "content": content,
                "sentAt": message.sent_at.isoformat(),
            })

    except WebSocketDisconnect:
        await manager.leave_room(room_id, websocket)
```

---

## 6. 외부 push (다른 endpoint 또는 Celery 에서)

ConnectionManager 가 main process 안 instance — multi-worker / multi-instance broadcast 위해선 **Redis pub/sub**:

```python
import json
from redis import asyncio as aioredis


async def publish_to_all_workers(redis: aioredis.Redis, room_id: str, message: dict) -> None:
    await redis.publish(f"ws:room:{room_id}", json.dumps(message))


# Each worker subscribes
async def redis_subscriber(redis: aioredis.Redis, manager: ConnectionManager):
    pubsub = redis.pubsub()
    await pubsub.psubscribe("ws:room:*")
    async for msg in pubsub.listen():
        if msg["type"] != "pmessage":
            continue
        channel = msg["channel"]            # "ws:room:42"
        room_id = channel.removeprefix("ws:room:")
        await manager.broadcast_to_room(room_id, json.loads(msg["data"]))


# lifespan startup 에서 background subscriber
@asynccontextmanager
async def lifespan(app: FastAPI):
    redis = aioredis.from_url(settings.REDIS_URL)
    sub_task = asyncio.create_task(redis_subscriber(redis, manager))
    yield
    sub_task.cancel()
    await redis.close()
```

---

## 7. Client (JS — Socket.IO 없이 raw)

```javascript
const ws = new WebSocket(
  `wss://api.example.com/ws/chat/42?token=${accessToken}`
);

ws.onopen = () => console.log("connected");

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log("message:", data);
};

ws.onclose = (event) => {
  console.log("disconnected:", event.code, event.reason);
};

// 메시지 전송
ws.send(JSON.stringify({ content: "안녕" }));
```

---

## 8. 함정 모음

### 함정 1 — JWT 가 query string 으로 노출
access log 에 토큰 평문. **헤더 사용 또는 first message 로 보냄** (browser WebSocket API 한계로 헤더 어려움 — 따라서 short-lived ws-specific token 권장).

### 함정 2 — ConnectionManager dict 가 multi-process / multi-instance 에서 분리
worker A 의 ws 가 B 의 broadcast 못 받음. **Redis pub/sub** 필수.

### 함정 3 — `await ws.send_json` 가 close 후 호출
`RuntimeError`. **WebSocketDisconnect catch + manager.leave_room** 정확히.

### 함정 4 — close code 4000+ 미사용
표준 1000 (정상) / 1001 (server 종료) / 1006 (abnormal). custom = 4000-4999 사용 (4401 unauthorized, 4403 forbidden 권장).

### 함정 5 — heartbeat / keepalive 없음
NAT / proxy 가 idle 끊음. uvicorn 의 `--ws-ping-interval` / `--ws-ping-timeout` 명시.

### 함정 6 — Python 한정: ConnectionManager 의 dict thread-safety
async 코드는 single-thread 라 OK 지만 multi-thread (BackgroundTasks 안) 는 lock 필요. **`asyncio.Lock`** 사용.

### 함정 7 — multi-worker (uvicorn `--workers N`)
ws 가 sticky session 필요 — LB 가 같은 client 항상 같은 worker. Redis pub/sub 으로 broadcast 동기화.

### 함정 8 — `WebSocket.receive_json` 의 parsing error
malformed JSON = `RuntimeError`. try/except 명시 + send_json error 응답.

---

## 9. 운영 체크리스트

- [ ] `wss://` (TLS) 강제
- [ ] JWT 검증 (query / first message)
- [ ] 권한 검증 (방 멤버 / role)
- [ ] ConnectionManager + Redis pub/sub (multi-worker / multi-instance)
- [ ] uvicorn `--ws-ping-interval 20 --ws-ping-timeout 60`
- [ ] WebSocketDisconnect 처리 (manager cleanup)
- [ ] close code 표준 (1xxx) 또는 4000+
- [ ] CORS 가 wss 도메인 허용
- [ ] **고성능** = Socket.IO (python-socketio) 또는 Channels (Django) 대안 고려

---

## 10. 관련

- [[../../spring/java-spring/api-design/websocket-stomp|↗ Java 원본 (STOMP)]]
- [[../../python-flask/api-design/websocket-stomp|↗ Flask 변형 (Flask-SocketIO)]]
- [[../common/security-config]]
- [[feed-timeline]] / [[recommendation]]
- [[api-design|↑ hub]]
