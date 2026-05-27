---
title: "troubleshooting · localStorage QuotaExceededError"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, browser, storage, localstorage, indexeddb]
home_hub: "60-troubleshooting"
related: ["[[frontend]]"]
---

# localStorage QuotaExceededError

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

```
DOMException: Failed to execute 'setItem' on 'Storage':
  Setting the value of 'cache' exceeded the quota.
```

- 사용자 일부에서만 발생 (5-10MB 도달)
- iOS Safari private mode 에서는 0KB 한도
- 큰 cache 저장 시점

## 원인

- localStorage 한도 5-10MB (브라우저별)
- iOS Safari private = 0KB
- Firefox 와 Chrome 도 origin 별 한도

## 해결

### try/catch 로 fallback
```ts
function safeSetItem(key: string, value: string) {
  try {
    localStorage.setItem(key, value);
    return true;
  } catch (e) {
    if (e instanceof DOMException && e.name === 'QuotaExceededError') {
      console.warn('localStorage quota exceeded');
      return false;
    }
    throw e;
  }
}
```

### IndexedDB 전환 — 50MB+ 가능
```ts
import { openDB } from 'idb';

const db = await openDB('myapp', 1, {
  upgrade(db) {
    db.createObjectStore('cache');
  },
});

await db.put('cache', value, key);
const cached = await db.get('cache', key);
```

→ async API, 큰 데이터 가능, structured cloning (object 직접 저장).

### 압축
```ts
import LZString from 'lz-string';

localStorage.setItem('cache', LZString.compress(JSON.stringify(data)));
const data = JSON.parse(LZString.decompress(localStorage.getItem('cache')!));
```

### Quota check (Storage API)
```ts
async function getQuota() {
  const { quota, usage } = await navigator.storage.estimate();
  return { quota, usage, percentage: (usage! / quota!) * 100 };
}
```

### LRU 제거
```ts
class LRUStorage {
  set(key: string, value: string) {
    try {
      localStorage.setItem(key, value);
    } catch (e) {
      this.evictOldest();
      localStorage.setItem(key, value);
    }
  }

  private evictOldest() {
    const keys = Object.keys(localStorage);
    keys.sort((a, b) => {
      const aTime = JSON.parse(localStorage.getItem(a) || '{}').accessedAt || 0;
      const bTime = JSON.parse(localStorage.getItem(b) || '{}').accessedAt || 0;
      return aTime - bTime;
    });
    keys.slice(0, 10).forEach(k => localStorage.removeItem(k));
  }
}
```

### iOS private mode 감지
```ts
function isPrivateMode(): boolean {
  try {
    localStorage.setItem('test', 'x');
    localStorage.removeItem('test');
    return false;
  } catch {
    return true;
  }
}
```

→ private mode 면 memory-based fallback (Map).

## 진단

```
# DevTools Application → Storage
- Cookies / Local Storage / IndexedDB / Cache Storage 별 용량
- 가장 큰 entry 확인

# Storage API
navigator.storage.estimate().then(console.log);
```

## 재발 방지

- localStorage 는 작은 데이터 (token / 설정) 만
- 큰 cache / offline data 는 IndexedDB
- 모든 `setItem` 호출은 safe wrapper
- 모니터링: client-side error tracking (Sentry) 에 QuotaExceededError 추적

## 관련

- [[frontend]] / [[service-worker-stale-cache]]
