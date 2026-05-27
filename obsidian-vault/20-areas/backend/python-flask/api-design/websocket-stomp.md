---
title: "WebSocket — Flask-SocketIO"
kind: knowledge
status: current
created_at: 2026-05-26T22:20:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, api-design, recipe, websocket, socketio]
---

# WebSocket — Flask-SocketIO

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/websocket-stomp|↗ Java 원본]]**

> Flask 진영은 **STOMP 직접 지원 없음**. 가장 흔한 대안 = **Flask-SocketIO** (Socket.IO 프로토콜). 또는 ASGI 로 넘어가서 [[../python-fastapi/api-design/websocket-stomp|FastAPI]] / Channels.

---

## 1. WebSocket vs STOMP vs Socket.IO

| | WebSocket (raw) | STOMP | Socket.IO |
| --- | --- | --- | --- |
| 표준 | RFC 6455 | sub-protocol on WS | 자체 프로토콜 (WS + polling fallback) |
| broker / channel | 없음 (직접) | 있음 (topic / queue) | 있음 (room / namespace) |
| Spring | `@MessageMapping` (STOMP) | ✅ 기본 | 별도 |
| Flask | 단순 echo 가능 | 없음 (라이브러리 부족) | **Flask-SocketIO** |
| 클라이언트 | browser WS API | stomp.js | socket.io-client (JS) |

→ Flask 는 **Socket.IO** 가 표준 선택. broker 통합 + room / namespace + auto fallback.

---

## 2. 의존성

```bash
pip install flask-socketio eventlet
# 또는 더 modern:
pip install flask-socketio gevent gevent-websocket
```

> **production async backend** — eventlet / gevent / threading 중 하나. 본 가이드는 eventlet (가장 흔함).

---

## 3. 설정 + JWT 인증

```python
# app/__init__.py
from flask import Flask
from flask_socketio import SocketIO
from flask_jwt_extended import decode_token, exceptions as jwt_exc

socketio = SocketIO()


def create_app() -> Flask:
    app = Flask(__name__)
    # ...
    socketio.init_app(
        app,
        cors_allowed_origins=app.config["CORS_ORIGINS"],
        message_queue="redis://localhost:6379/0",        # multi-instance broadcast
        async_mode="eventlet",
        logger=False,
        engineio_logger=False,
    )
    return app
```

```python
# app/sockets/auth.py
from flask import request
from flask_socketio import disconnect
from flask_jwt_extended import decode_token

from app import socketio


@socketio.on("connect")
def on_connect(auth: dict | None):
    """클라이언트가 Authorization 헤더 또는 auth payload 로 토큰 전달."""
    token = None
    if auth and "token" in auth:
        token = auth["token"]
    else:
        # Authorization 헤더 (Bearer xxx)
        header = request.headers.get("Authorization", "")
        if header.startswith("Bearer "):
            token = header[7:]

    if not token:
        disconnect()
        return False

    try:
        claims = decode_token(token)
    except Exception:
        disconnect()
        return False

    # session 에 user_id 저장 — 다음 event 들이 활용
    from flask import session
    session["user_id"] = int(claims["sub"])
    return True


@socketio.on("disconnect")
def on_disconnect():
    from flask import session
    user_id = session.get("user_id")
    print(f"WS disconnect user={user_id}")
```

> **socket.io 의 `auth` payload**: client (JS) 가 `io({ auth: { token: "Bearer xxx" } })` 처럼 보냄.

---

## 4. Room 구독 (Java STOMP 의 `/topic/chat/{roomId}` 동등)

```python
# app/sockets/chat.py
from flask import session
from flask_socketio import join_room, leave_room, emit

from app import socketio
from app.services import chat_service


@socketio.on("subscribe_room")
def subscribe_chat_room(data: dict):
    """클라이언트가 'subscribe_room' event 보내면 방 join."""
    room_id = data.get("roomId")
    user_id = session.get("user_id")

    if not user_id:
        emit("error", {"message": "인증 필요"})
        return

    # 권한 검증 — 방 멤버인지
    if not chat_service.is_member(user_id, room_id):
        emit("error", {"message": "방 멤버가 아닙니다."})
        return

    join_room(f"chat:{room_id}")
    emit("subscribed", {"roomId": room_id})


@socketio.on("send_message")
def send_chat_message(data: dict):
    user_id = session.get("user_id")
    room_id = data.get("roomId")
    content = data.get("content", "")

    if not user_id or not room_id or not content:
        emit("error", {"message": "invalid"})
        return

    message = chat_service.send(user_id, room_id, content)

    # 모든 방 구독자에게 broadcast
    socketio.emit(
        "new_message",
        {
            "messageId": message.id,
            "senderId": user_id,
            "content": content,
            "sentAt": message.sent_at.isoformat(),
        },
        room=f"chat:{room_id}",
    )
```

---

## 5. user 개인 알림 (Spring `/user/queue/notifications` 동등)

socket.io 는 user-specific room 을 `connect` 시 자동 join 으로 흉내:

```python
@socketio.on("connect")
def on_connect(auth: dict | None):
    # ... JWT 검증 ...
    from flask_socketio import join_room
    join_room(f"user:{user_id}")            # user 개인 room
    return True


# 외부에서 알림 broadcast
def push_notification(user_id: int, notification: dict) -> None:
    socketio.emit("notification", notification, room=f"user:{user_id}")
```

---

## 6. 서버 → 클라이언트 push (다른 Flask endpoint / Celery 에서)

```python
# 일반 Flask view 에서
@app.post("/api/v1/messages/<int:room_id>")
@jwt_required()
def send_message_via_http(room_id: int):
    user_id = int(get_jwt_identity())
    content = request.json["content"]
    message = chat_service.send(user_id, room_id, content)

    # WebSocket 으로 broadcast
    socketio.emit("new_message", {...}, room=f"chat:{room_id}")
    return ok(message)


# Celery task 에서
@shared_task
def send_notification_async(user_id: int, message: str):
    socketio.emit(
        "notification",
        {"message": message},
        room=f"user:{user_id}",
    )
```

> **`message_queue="redis://..."`** 가 핵심 — multi-instance / multi-worker 환경에서 Redis pub/sub 으로 broadcast 동기화.

---

## 7. 실행

```python
# run.py
from app import create_app, socketio

app = create_app()

if __name__ == "__main__":
    socketio.run(app, host="0.0.0.0", port=5000, debug=True)
```

```bash
python run.py
# 또는 운영:
gunicorn -k eventlet -w 1 -b 0.0.0.0:5000 run:app
```

> **gunicorn `-w 1`** — Socket.IO 는 sticky session 필요 (multi-worker 시 같은 client 가 다른 worker 에 갈 수 있음). 대신 `--workers N` + Redis message_queue + LB sticky session 설정.

---

## 8. Client (JS)

```javascript
import { io } from "socket.io-client";

const socket = io("wss://shop.example.com", {
  auth: { token: localStorage.getItem("accessToken") },
  transports: ["websocket"],
});

socket.on("connect", () => {
  console.log("connected");

  socket.emit("subscribe_room", { roomId: 42 });
});

socket.on("new_message", (data) => {
  console.log("message:", data);
});

socket.on("notification", (data) => {
  console.log("notification:", data);
});

socket.on("error", (err) => {
  console.error("ws error:", err);
});

socket.on("disconnect", (reason) => {
  console.log("disconnected:", reason);
});
```

---

## 9. 함정 모음

### 함정 1 — `async_mode` 설정 안 함
default = threading — 운영에선 부족. **eventlet 또는 gevent** 명시.

### 함정 2 — `message_queue` 없이 multi-instance 운영
인스턴스 A 의 emit 이 B 의 client 에 안 감. **Redis message_queue 필수**.

### 함정 3 — `gunicorn -w 4` (multi-worker) + Socket.IO
같은 client 가 다른 worker 에 가면 연결 끊김. **`-w 1` + multiple instances + LB sticky** 또는 ASGI로 전환 (FastAPI).

### 함정 4 — JWT 가 query string 으로 노출
`io("?token=xxx")` 처럼 query 로 보내면 access log 에 토큰 평문. **`auth` payload 또는 Authorization 헤더**.

### 함정 5 — `disconnect()` 호출 후 추가 emit
`disconnect()` 직후 emit = error. **`return False`** 로 명확히 거부.

### 함정 6 — Flask session 의 `user_id` 가 multi-tab 에서 공유
같은 브라우저의 다른 탭이 같은 session 공유 가능. **socket id 별 dict** 또는 socketio 의 client_id 사용.

### 함정 7 — Python 한정: `gevent` vs `eventlet` 호환성
일부 라이브러리 (psycopg) 가 gevent monkey-patch 와 충돌. **`monkey.patch_all()`** 위치 / Postgres driver 호환 확인.

### 함정 8 — Python 한정: Socket.IO 4.x vs 3.x 클라이언트 호환
서버 4.x + 클라 3.x 면 protocol 불일치. **버전 lock** (server + client 둘 다).

---

## 10. 운영 체크리스트

- [ ] `async_mode="eventlet"` 또는 `gevent`
- [ ] `message_queue="redis://..."` (multi-instance broadcast)
- [ ] `cors_allowed_origins` 명시 (와일드카드 X)
- [ ] JWT 검증 — `auth` payload 또는 Authorization 헤더 (query X)
- [ ] gunicorn `-k eventlet -w 1` + 인스턴스 multi (sticky session)
- [ ] heartbeat (`ping_interval` / `ping_timeout`) 명시
- [ ] disconnect 사유 logging
- [ ] CORS 가 wss 도메인 허용
- [ ] socket.io-client 버전 lock
- [ ] **고성능 / 다중 클라이언트 = FastAPI WebSocket 또는 Django Channels 고려**

---

## 11. 관련

- [[../../spring/java-spring/api-design/websocket-stomp|↗ Java 원본 (STOMP)]]
- [[../common/security-config]] — JWT 검증
- [[feed-timeline]] / [[recommendation]] — 알림 broadcast 활용
- [[api-design|↑ hub]]
