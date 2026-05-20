---
title: "이미지 최적화 — WebP / AVIF / responsive / lazy"
kind: knowledge
project: frontend
status: current
tags: [frontend, performance, images, optimization]
---

# 이미지 최적화

**[[../performance-optimization|↑ Performance hub]]**

> 이미지는 페이지 무게의 50%+. 최적화 한 곳이 가장 큰 효과.

---

## 1. 포맷

| 포맷 | 압축 | 투명 | 브라우저 | 사용 |
| --- | --- | --- | --- | --- |
| JPEG | 좋음 (lossy) | X | 모두 | 사진 (옛) |
| PNG | 나쁨 (lossless) | O | 모두 | 로고 / 투명 (옛) |
| **WebP** | 매우 좋음 | O | 모두 (IE 제외) | **사진 / 일반** |
| **AVIF** | 최고 | O | 모던 (Safari 16.4+) | **최신 사이트** |
| SVG | 벡터 | O | 모두 | 아이콘 / 로고 |
| GIF | 나쁨 | O | 모두 | 짧은 애니메이션 (요즘은 video) |

권장: **AVIF + WebP fallback + JPEG fallback**.

---

## 2. 변환

### CLI
```bash
# sharp (가장 빠름)
npx sharp-cli -i input.jpg -o output.webp --quality 85

# squoosh
npx @squoosh/cli --webp '{}' --avif '{}' input.jpg

# imagemin
npx imagemin input.jpg --out-dir=dist
```

### 자동 — Next.js Image
```tsx
import Image from "next/image";

<Image src="/photo.jpg" alt="" width={800} height={600} />
```

자동으로 AVIF/WebP 변환 + 적절한 크기 제공.

### 자동 — Vite
```bash
npm install -D vite-imagetools
```

```typescript
import img from "./photo.jpg?w=800&format=webp";
```

빌드 시 자동 변환.

### 온라인 도구
- https://squoosh.app — 시각적 비교 + 다운로드

---

## 3. picture + source — fallback

```html
<picture>
  <source srcset="/hero.avif" type="image/avif">
  <source srcset="/hero.webp" type="image/webp">
  <img src="/hero.jpg" alt="" width="800" height="600">
</picture>
```

브라우저가 지원하는 첫 source 선택.

---

## 4. srcset — 해상도별 다른 사진

```html
<img
  src="/photo-800.jpg"
  srcset="
    /photo-400.jpg 400w,
    /photo-800.jpg 800w,
    /photo-1600.jpg 1600w
  "
  sizes="(max-width: 640px) 100vw, 50vw"
  alt=""
>
```

- `srcset` — 사용 가능 사진 목록 (각 폭)
- `sizes` — 화면 크기 별 사진의 표시 폭
- 브라우저가 자동으로 최적 사진 선택

### DPR (Retina) 별
```html
<img
  src="/photo.jpg"
  srcset="
    /photo.jpg 1x,
    /photo@2x.jpg 2x,
    /photo@3x.jpg 3x
  "
>
```

---

## 5. lazy loading

### Native (모던 브라우저)
```html
<img src="..." loading="lazy" alt="">
```

화면 밖이면 로드 X. 가까이 오면 자동.

### 첫 화면 (LCP)
```html
<img src="hero.jpg" loading="eager" fetchpriority="high" alt="">
```

LCP 후보면 eager + high priority.

### iframe / video 도 가능
```html
<iframe src="..." loading="lazy"></iframe>
```

---

## 6. width / height 명시 (CLS 방지)

```html
<!-- ❌ size 없으면 늦게 reflow -->
<img src="..." alt="">

<!-- ✅ 미리 자리 잡음 -->
<img src="..." width="800" height="600" alt="">
```

CSS 로 비율 유지하면서 반응형:
```css
img {
  width: 100%;
  height: auto;
  aspect-ratio: 4/3;     /* 또는 16/9 등 */
}
```

---

## 7. background-image

CSS 의 background 도 최적화:

### picture-set (Chrome 99+)
```css
.hero {
  background-image: image-set(
    url("hero.avif") type("image/avif") 1x,
    url("hero.webp") type("image/webp") 1x,
    url("hero.jpg") 1x
  );
}
```

또는 JS 로 detection 후 클래스 적용.

---

## 8. 콘텐츠 별 권장

### Hero 사진 (큰 메인)
- 포맷: AVIF + WebP fallback
- 크기: 1920px wide 정도 + responsive srcset
- loading: eager + fetchpriority="high"
- preload (LCP):
  ```html
  <link rel="preload" as="image" href="/hero.webp" fetchpriority="high">
  ```

### 갤러리 / 목록
- 포맷: WebP
- 크기: thumbnail (200~400px)
- loading: lazy
- 클릭 시 큰 이미지 로드 (lightbox)

### 아바타 / 작은 아이콘
- SVG (벡터) 또는 작은 WebP
- 인라인 SVG 가능 (요청 0)

### 로고
- SVG 권장

---

## 9. Next.js Image — 완전 가이드

```tsx
import Image from "next/image";

// 정적 (빌드 시 메타 자동)
import hero from "./hero.jpg";
<Image src={hero} alt="" placeholder="blur" />

// 동적 (URL)
<Image src="/hero.jpg" alt="" width={800} height={600} />

// fill (부모 채우기)
<div style={{ position: "relative", aspectRatio: "16/9" }}>
  <Image src="/hero.jpg" alt="" fill style={{ objectFit: "cover" }} />
</div>

// priority (LCP)
<Image src="/hero.jpg" alt="" width={1200} height={600} priority />

// quality (1~100, default 75)
<Image src="/photo.jpg" alt="" width={800} height={600} quality={90} />
```

### 외부 도메인 허용
```typescript
// next.config.ts
images: {
  remotePatterns: [
    { protocol: "https", hostname: "cdn.example.com" },
  ],
}
```

### 자동 처리
- AVIF / WebP 변환
- 적응형 srcset
- lazy loading (priority 제외)
- placeholder blur (정적 import)

---

## 10. 압축 quality

| Quality | 크기 | 시각 |
| --- | --- | --- |
| 100 | 매우 큼 | 완벽 (눈으로 차이 X) |
| 85 | 보통 | 거의 완벽 (권장) |
| 70 | 작음 | 사진은 OK, 일부 흠 |
| 50 | 매우 작음 | 흐림 보임 |

**권장**: 사진 80~85, 일러스트 75~80.

---

## 11. SVG 최적화

```bash
npm install -D svgo
npx svgo input.svg -o output.svg
```

또는 https://jakearchibald.github.io/svgomg/ 온라인.

### 인라인 SVG (요청 0)
```tsx
function CheckIcon() {
  return (
    <svg viewBox="0 0 24 24" width="16" height="16" fill="currentColor">
      <path d="..." />
    </svg>
  );
}
```

색 자동 (currentColor) → CSS 로 색 변경.

---

## 12. CDN 활용

이미지 호스팅 + 변환 서비스:
- **Cloudinary** — 가장 유명, URL 파라미터로 변환
- **Cloudflare Images** — 저렴
- **imgix** — 강력
- **Vercel Image Optimization** — Next.js 자동

### Cloudinary URL 예
```
https://res.cloudinary.com/demo/image/upload/w_800,q_auto,f_auto/sample.jpg
```

- `w_800` — 폭 800
- `q_auto` — 자동 quality
- `f_auto` — 자동 포맷 (브라우저 따라 AVIF/WebP)

---

## 13. 체크리스트

- [ ] Hero 이미지 AVIF + WebP fallback
- [ ] 모든 `<img>` 에 `width` `height` 또는 `aspect-ratio`
- [ ] 화면 밖 이미지 `loading="lazy"`
- [ ] LCP 이미지 `loading="eager"` + `fetchpriority="high"`
- [ ] LCP 이미지 preload
- [ ] responsive `srcset` + `sizes`
- [ ] 아이콘은 SVG (가능하면 인라인)
- [ ] alt 텍스트 모든 이미지에

---

## 14. 도구 모음

- **Squoosh** — https://squoosh.app
- **SVGOMG** — https://jakearchibald.github.io/svgomg/
- **TinyPNG** — https://tinypng.com (단순)
- **next/image** — Next.js 자동
- **vite-imagetools** — Vite 자동
- **sharp** — Node.js 변환 라이브러리

---

## 15. 함정

| 증상 | 원인 |
| --- | --- |
| 모바일 데이터 폭주 | srcset 안 함 → 풀 해상도 |
| CLS 가 높음 | width/height 없어서 reflow |
| AVIF 가 흐림 | quality 너무 낮음. 80+ 권장 |
| Cloudinary 의 크기 안 줄어듦 | `q_auto` `f_auto` 빠뜨림 |

---

## 16. 다음 단계

1. **[[../bundle/code-splitting]]** — JS 분할
2. **[[../web-vitals/core-web-vitals]]** — 측정

---

연결: [[../performance-optimization|↑ hub]]
