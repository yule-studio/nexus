---
title: "Video Player — HTML5 video + 업로드 흐름 + 단계적 storage 교체"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, video, upload, storage, recipes]
related:
  - "[[large-component-decomposition]]"
  - "[[supabase-with-react-query]]"
created_at: 2026-05-18
---

# Video Player — HTML5 video + 업로드 흐름 + 단계적 storage 교체

**[[recipes|↑ Recipes hub]]**

> 강의 영상 / 시뮬레이션 영상처럼 영상 업로드·재생 기능을 React 에 붙이는 패턴. 처음에는 시뮬레이션 (mock) 으로 흐름을 검증하고, 나중에 Supabase Storage / AWS S3 로 교체하는 게 안전.

## 1. 왜 단계적 교체인가

### 왜 필요
- 영상 파일은 100MB ~ 5GB — 진짜 업로드 붙이면 개발 중 디스크 / 네트워크 / Storage 비용 / RLS 정책 등 한꺼번에 다 해결해야 함.
- 화면 흐름 (드롭존 → 진행률 → 완료 alert) 만 먼저 검증하면 UX 디자인 / 사이즈 검증 / 에러 메시지 등 디테일을 빨리 잡을 수 있음.
- 실 storage 붙일 때는 hook 안 한 함수 (`simulateUpload` → `supabase.storage.upload`) 만 갈아끼움 — UI 변경 0.

### 안 하면 문제
- 처음부터 진짜 Supabase 업로드 = 인증 / 토큰 / RLS / signed URL / progress event 다 동시에 디버깅. 한 군데 막히면 전체 흐름 stuck.
- 화면 흐름이 검증 안 된 상태에서 실 업로드 붙이면 "왜 1GB 파일 올렸는데 진행률이 안 보이지?" 같은 UX bug 가 storage 디버깅에 묻힘.

### 대안
- **처음부터 실 storage**: 작은 영상 (5MB) 한 종류만 테스트. UX 디테일 누락 위험.
- **mock fetch (MSW)**: API 응답을 가짜로. 파일 크기 / progress event 시뮬레이션은 직접 해야 함.
- **시뮬레이션 hook + 단계적 교체** (본 패턴): 흐름 검증 + storage 교체 분리. 시뮬레이션 단계에서 5GB 검증 / 드래그앤드롭 / 진행률 다 완성됨.

### 트레이드오프
- 본 패턴: storage 교체 단계가 추가 commit / PR 필요. 하지만 그 PR 은 hook 안 한 함수만 바뀌므로 PR 크기 작음 (~30줄).

## 2. 업로드 흐름 — useUploadFlow hook

```ts
// src/pages/.../lessons/useUploadFlow.ts
export function useUploadFlow(onTitleSuggest?: (title: string) => void) {
  const [selectedFile, setSelectedFile] = useState<File | null>(null)
  const [isDragging, setIsDragging] = useState(false)
  const [uploading, setUploading] = useState(false)
  const [progress, setProgress] = useState(0)
  const fileInputRef = useRef<HTMLInputElement>(null)

  const handleFileSelect = (file: File) => {
    if (!file.type.startsWith('video/')) { alert('...'); return }
    if (file.size > MAX_UPLOAD_SIZE_BYTES) { alert('...'); return }
    setSelectedFile(file)
    if (onTitleSuggest) onTitleSuggest(file.name.replace(/\.[^/.]+$/, ''))
  }

  const handleDrop = (e: React.DragEvent) => {
    e.preventDefault()
    setIsDragging(false)
    const file = e.dataTransfer.files[0]
    if (file) handleFileSelect(file)
  }

  // S3 단계 — supabase.storage.from('lessons').upload(path, file, {
  //   onProgress: (e) => setProgress(Math.round((e.loaded / e.total) * 100))
  // }) 호출로 교체
  const simulateUpload = async (): Promise<void> => {
    setUploading(true)
    setProgress(0)
    for (let i = 0; i <= 20; i++) {
      await new Promise((r) => setTimeout(r, 150))
      setProgress(Math.round((i / 20) * 100))
    }
  }

  return { selectedFile, isDragging, uploading, progress, fileInputRef,
           handleFileSelect, handleDrop, handleDragOver, handleDragLeave,
           handleFileClick, simulateUpload, reset }
}
```

핵심:
- **검증 우선** — type / size 를 hook 안에서 검증, 컴포넌트는 alert 만 트리거.
- **콜백으로 자동 제목 채우기** — `onTitleSuggest` prop 으로 hook 이 form 에 의존하지 않음.
- **simulateUpload 만 교체 가능** — S3 시 supabase.storage.upload + onProgress 콜백으로 한 함수 swap.

## 3. 드롭존 컴포넌트

```tsx
function DropZone({ isDragging, onDrop, onDragOver, onDragLeave, onClick, inputRef, onFileSelect }) {
  return (
    <div
      onDrop={onDrop}
      onDragOver={onDragOver}
      onDragLeave={onDragLeave}
      onClick={onClick}
      className="border-2 border-dashed rounded-2xl p-10 cursor-pointer"
      style={{ borderColor: isDragging ? accent : '#D1D5DB' }}
    >
      <input
        ref={inputRef}
        type="file"
        accept="video/*"
        className="hidden"
        onChange={(e) => {
          const file = e.target.files?.[0]
          if (file) onFileSelect(file)
        }}
      />
      <div>{isDragging ? '여기에 놓으세요' : '드래그하거나 클릭'}</div>
    </div>
  )
}
```

- `<input type="file">` 은 hidden, 클릭은 ref 로 트리거.
- 드래그 상태에 따라 border / background 변경으로 UX 신호.
- `accept="video/*"` 로 파일 선택 dialog 가 영상만 보여줌 (검증은 별도 — accept 는 강제 아님).

## 4. 진행률 표시

```tsx
{uploading && (
  <div>
    <div className="flex justify-between">
      <div>📤 업로드 중...</div>
      <div>{progress}%</div>
    </div>
    <div className="h-2 bg-white rounded-full overflow-hidden">
      <div
        className="h-full rounded-full transition-all"
        style={{ width: `${progress}%`, background: gradient }}
      />
    </div>
  </div>
)}
```

- `width: ${progress}%` + `transition-all` → 매끄러운 진행 애니메이션.
- 업로드 중에는 모달 close / 폼 변경 차단 (parent 의 `if (uploading) return` 가드).

## 5. 영상 재생 — 단계적 적용

### Stage 1: Placeholder (mock 단계)

```tsx
function PlayerPlaceholder({ lesson }) {
  return (
    <div style={{ paddingBottom: '56.25%', background: 'linear-gradient(...)' }}>
      <div className="absolute inset-0 flex items-center justify-center flex-col">
        <div>🎬</div>
        <div>영상 플레이어 영역</div>
        <div>📁 {lesson.fileName}</div>
      </div>
    </div>
  )
}
```

- 16:9 비율 (`paddingBottom: 56.25%`) — 진짜 영상과 같은 비율로 레이아웃 검증.
- 파일명 / 크기 / 해상도 메타 표시 — 데이터 흐름 검증.

### Stage 2: HTML5 video + Supabase Storage signed URL

```tsx
function LessonPlayer({ lesson }) {
  const { data: signedUrl } = useQuery({
    queryKey: ['lesson-url', lesson.fileName],
    queryFn: async () => {
      const { data } = await supabase.storage
        .from('lessons')
        .createSignedUrl(lesson.fileName, 60 * 60) // 1 시간 유효
      return data?.signedUrl
    },
  })

  if (!signedUrl) return <PlayerPlaceholder lesson={lesson} />

  return (
    <video
      controls
      src={signedUrl}
      style={{ width: '100%' }}
      onError={(e) => console.error('video error:', e)}
    />
  )
}
```

- **Signed URL** — Supabase Storage 의 private bucket 에 직접 접근 못 함. 1 시간짜리 signed URL 발급받아 `<video src>` 에 사용.
- React Query 로 캐싱 — 같은 영상 여러 번 열어도 URL 재발급 안 함 (만료 전까지).
- `<video controls>` 로 기본 컨트롤 (재생 / 일시정지 / 음량 / 전체화면). 커스텀 컨트롤 필요하면 controls 제거하고 onPlay / onPause / onTimeUpdate 이벤트로 구현.

### Stage 3 (선택): 재생 속도 / 자막 / picture-in-picture

```tsx
const videoRef = useRef<HTMLVideoElement>(null)

const setRate = (rate: number) => {
  if (videoRef.current) videoRef.current.playbackRate = rate
}

const togglePip = () => {
  if (document.pictureInPictureElement) document.exitPictureInPicture()
  else videoRef.current?.requestPictureInPicture()
}

// JSX
<video ref={videoRef} controls src={signedUrl}>
  <track kind="captions" src="/captions/ko.vtt" srcLang="ko" label="한국어" default />
</video>
```

- `playbackRate` — 0.5 / 1 / 1.25 / 1.5 / 2x. 학습 영상은 1.25-1.5x 가 가장 자주 쓰임.
- `<track kind="captions">` — WebVTT 형식 자막. 별도 .vtt 파일 필요.
- `requestPictureInPicture()` — 멀티탭 사용 시 영상 작은 창으로 계속 재생.

## 6. Storage 비용 고려

- **5GB 영상 100개 = 500GB** — Supabase free tier (1GB) 부족. Pro plan (100GB / $25) 또는 AWS S3 (스토리지 + transfer 비용 별도).
- **CDN 캐싱** 필수 — 학원 47개가 같은 영상 본다고 47번 다운로드하면 transfer 비용 폭발. Supabase Storage 는 CDN 자동, S3 는 CloudFront 별도.
- **압축 / 트랜스코딩** — 원본 (4K MOV 2GB) 그대로 두지 말고 H.264 1080p (300MB) + 720p / 480p 자동 트랜스코딩. CloudFlare Stream / AWS MediaConvert / Mux 같은 서비스 활용.

## 7. 함정

1. **`<input type="file">` 의 검증 신뢰** — `accept="video/*"` 는 UI 힌트일 뿐. 사용자가 dev tool 로 끄거나 다른 파일 끌어다 놓을 수 있음. 항상 JS 에서 `file.type.startsWith('video/')` 재검증.
2. **5GB 파일 한 번에 메모리 로드** — `URL.createObjectURL(file)` 로 미리보기는 OK, fetch 로 한 번에 보내면 메모리 폭발. supabase.storage.upload 는 자체 streaming.
3. **드래그앤드롭 `preventDefault` 누락** — `onDragOver` 에서 preventDefault 안 하면 drop 이벤트 자체가 발생 안 함.
4. **업로드 중 모달 close** — close 시 abort signal 없으면 백그라운드에서 계속 업로드. parent 에서 `if (uploading) return` 가드 필수.
5. **signed URL 만료 후 재생 중단** — 1 시간 유효한 URL 이 재생 중 만료되면 video 가 멈춤. 영상 길이 + 여유 시간 (예: 영상 1시간이면 URL 3시간) 발급.
6. **mock 데이터의 `fileSize: '287 MB'` string** — number 가 아니라 string 으로 두면 sort / 합산 시 `parseFloat` 필요. 실 데이터는 bytes (number) 로 저장.

## 8. 관련

- [[large-component-decomposition]] — Lessons.tsx 1104줄 분리 (드롭존 + 폼 + 진행률 + 결과를 한 모달에서 10 sub-component 로)
- [[supabase-with-react-query]] — Storage signed URL + React Query 캐싱 (S3 단계)
- [[react/recipes/file-export]] — 다운로드 (PDF / Excel) 관련 — Lessons 와 반대 방향
