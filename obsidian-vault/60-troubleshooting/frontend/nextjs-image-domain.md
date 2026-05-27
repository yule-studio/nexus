---
title: "troubleshooting · Next.js Image — Invalid src prop"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, nextjs, image, optimization]
home_hub: "60-troubleshooting"
related: ["[[frontend]]"]
---

# Next.js Image — `Invalid src prop on next/image`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

```
Error: Invalid src prop (https://cdn.example.com/img.png) on `next/image`,
hostname "cdn.example.com" is not configured under images in your `next.config.js`
```

- 또는 4xx — `next/image` API 가 외부 URL fetch 실패
- dev 에서는 정상, build / prod 에서 발생

## 원인

`next/image` 는 server 측 optimizer (resize / WebP 변환). 외부 hostname 은 `next.config.js` 에 명시된 것만 허용 (SSRF 방어).

## 해결

### `images.remotePatterns` (Next.js 13+)
```js
// next.config.js
module.exports = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'cdn.example.com',
        pathname: '/uploads/**',
      },
      {
        protocol: 'https',
        hostname: '**.cloudfront.net',   // wildcard
      },
    ],
  },
};
```

### `images.domains` (deprecated, 14+)
```js
images: {
  domains: ['cdn.example.com', 'avatars.githubusercontent.com'],
},
```

→ remotePatterns 권장 (path filter 가능).

### `unoptimized={true}` — 우회
```tsx
<Image src={url} width={300} height={200} alt="..." unoptimized />
```

→ optimizer 거치지 않음. WebP / resize 안 됨. 가급적 안 씀.

### Custom loader
```tsx
const myLoader = ({ src, width, quality }: ImageLoaderProps) => {
  return `https://my-cdn.com/${src}?w=${width}&q=${quality || 75}`;
};

<Image loader={myLoader} src="image.png" width={300} height={200} alt="..." />
```

## SVG 함정

```
Error: Image with src "...svg" has unconfigured `dangerouslyAllowSVG`
```

```js
images: {
  dangerouslyAllowSVG: true,
  contentSecurityPolicy: "default-src 'self'; script-src 'none'; sandbox;",
},
```

→ SVG 는 XSS 위험 — `dangerouslyAllowSVG` + CSP 필수. 또는 SVG 는 `<img>` 직접 사용 / inline.

## `placeholder="blur"` + remote

```tsx
<Image
  src="https://cdn.example.com/img.png"
  width={1200}
  height={800}
  alt="..."
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,..."   // ← remote 는 미리 생성
/>
```

local image 는 자동 blur, remote 는 직접 제공.

## Static export 함정

```js
// next.config.js
module.exports = {
  output: 'export',
  images: {
    unoptimized: true,   // ← static export 는 optimizer 사용 X
  },
};
```

## 진단

```
# build log
- "Image domain not configured" 메시지
- /_next/image API 404 / 400

# DevTools Network
- /_next/image?url=... 요청 / 응답
```

## 재발 방지

- 새 CDN / 외부 image 호스트 추가 시 `remotePatterns` 업데이트
- CI: image src 의 hostname 추출 → config 와 비교
- placeholder blur 는 build time 생성 (plaiceholder library)

## 관련

- [[frontend]]
