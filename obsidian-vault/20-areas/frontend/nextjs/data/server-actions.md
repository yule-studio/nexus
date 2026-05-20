---
title: "Next.js Server Actions — form 으로 서버 호출"
kind: knowledge
project: frontend
status: current
tags: [frontend, nextjs, server-actions, forms, mutations]
---

# Server Actions

**[[../nextjs|↑ Next.js hub]]**

> 클라이언트에서 서버 함수를 그냥 호출. API 만들 필요 X.

---

## 1. 한 줄 요약

```tsx
"use server";    // 이 함수는 서버에서

export async function createPost(formData) {
  await db.insert(...);
}
```

```tsx
// 클라이언트
<form action={createPost}>
  <input name="title" />
  <button>저장</button>
</form>
```

→ 폼 제출 시 createPost 가 서버에서 자동 실행. fetch / API endpoint 불필요.

---

## 2. 가장 단순한 예

```tsx
// app/page.tsx
export default function Page() {
  async function create(formData: FormData) {
    "use server";
    const title = formData.get("title");
    console.log("Saving:", title);
    // 여기서 DB 저장
  }

  return (
    <form action={create}>
      <input name="title" />
      <button>저장</button>
    </form>
  );
}
```

### 작동 흐름
1. 사용자가 form 제출
2. Next 가 자동으로 createPost 호출 (RPC 같이)
3. 서버에서 함수 실행
4. 결과 / 에러 처리

---

## 3. 분리된 파일로 export

```tsx
// app/actions.ts
"use server";

import { db } from "@/lib/db";
import { revalidatePath } from "next/cache";

export async function createPost(formData: FormData) {
  const title = formData.get("title") as string;
  const body  = formData.get("body") as string;

  await db.posts.create({ title, body });
  revalidatePath("/posts");      // 목록 페이지 캐시 무효화
}

export async function deletePost(id: string) {
  await db.posts.delete(id);
  revalidatePath("/posts");
}
```

```tsx
// app/posts/new/page.tsx
import { createPost } from "@/app/actions";

export default function NewPost() {
  return (
    <form action={createPost}>
      <input name="title" required />
      <textarea name="body" required />
      <button>저장</button>
    </form>
  );
}
```

---

## 4. Client Component 에서 사용

```tsx
"use client";
import { createPost } from "@/app/actions";

export default function PostForm() {
  return (
    <form action={createPost}>
      ...
    </form>
  );
}
```

Client 도 직접 import 가능. Next 가 자동 RPC.

---

## 5. useActionState (이전 useFormState)

폼 결과 처리.

```tsx
// app/actions.ts
"use server";

export async function login(prevState, formData) {
  const email = formData.get("email");
  const pw    = formData.get("password");
  if (!email || !pw) return { error: "필수 입력" };

  const user = await authenticate(email, pw);
  if (!user) return { error: "인증 실패" };

  redirect("/dashboard");   // 또는 return { success: true }
}
```

```tsx
"use client";
import { useActionState } from "react";
import { login } from "@/app/actions";

export default function LoginForm() {
  const [state, formAction] = useActionState(login, { error: null });

  return (
    <form action={formAction}>
      <input name="email" />
      <input name="password" type="password" />
      <button>로그인</button>
      {state?.error && <p style={{ color: "red" }}>{state.error}</p>}
    </form>
  );
}
```

---

## 6. useFormStatus — pending 상태

```tsx
"use client";
import { useFormStatus } from "react-dom";

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? "저장 중..." : "저장"}</button>;
}

export default function Form() {
  return (
    <form action={createPost}>
      <input name="title" />
      <SubmitButton />
    </form>
  );
}
```

`useFormStatus` 는 가까운 `<form>` 의 상태 알려줌.

---

## 7. 버튼 onClick 으로 호출 (form 없이)

```tsx
"use client";
import { deletePost } from "@/app/actions";

export default function DeleteButton({ id }) {
  return (
    <button onClick={() => deletePost(id)}>삭제</button>
  );
}
```

또는 form + hidden input:
```tsx
<form action={deletePost}>
  <input type="hidden" name="id" value={id} />
  <button>삭제</button>
</form>
```

---

## 8. bind — 인자 미리 채우기

```tsx
"use client";
import { deletePost } from "@/app/actions";

export default function DeleteButton({ id }) {
  const deleteWithId = deletePost.bind(null, id);
  return (
    <form action={deleteWithId}>
      <button>삭제</button>
    </form>
  );
}
```

```tsx
// actions.ts
"use server";
export async function deletePost(id: string, formData: FormData) {
  await db.posts.delete(id);
}
```

---

## 9. progressive enhancement

JS 가 로드되기 전에도 form 동작 (HTML form 의 기본 동작 활용). Server Action 의 큰 장점.

JS 꺼진 브라우저 / 느린 네트워크에서도 form 제출 가능.

---

## 10. revalidate / redirect

Server Action 안에서:

```tsx
"use server";
import { revalidatePath, revalidateTag } from "next/cache";
import { redirect } from "next/navigation";

export async function createPost(formData) {
  await db.posts.create({...});
  revalidatePath("/posts");      // 페이지 캐시 무효화
  revalidateTag("posts");        // tag 캐시 무효화
  redirect("/posts");            // 다른 페이지로
}
```

`redirect` 이후 코드는 실행 안 됨.

---

## 11. 보안 — 인증 검사

```tsx
"use server";
import { auth } from "@/lib/auth";

export async function deletePost(id) {
  const user = await auth();
  if (!user) throw new Error("로그인 필요");

  const post = await db.posts.find(id);
  if (post.userId !== user.id) throw new Error("권한 없음");

  await db.posts.delete(id);
}
```

⚠️ Server Action 은 외부에서 직접 호출 가능 (Endpoint 처럼). **반드시 인증/권한 검사**.

---

## 12. 검증 (zod 예시)

```tsx
"use server";
import { z } from "zod";

const schema = z.object({
  title: z.string().min(1).max(200),
  body: z.string().min(10),
});

export async function createPost(formData: FormData) {
  const data = schema.safeParse({
    title: formData.get("title"),
    body: formData.get("body"),
  });
  if (!data.success) return { errors: data.error.flatten() };

  await db.posts.create(data.data);
}
```

---

## 13. 파일 업로드

```tsx
"use server";

export async function uploadFile(formData: FormData) {
  const file = formData.get("file") as File;
  const buffer = Buffer.from(await file.arrayBuffer());
  // S3 / 디스크 저장
}
```

```tsx
<form action={uploadFile}>
  <input name="file" type="file" />
  <button>업로드</button>
</form>
```

---

## 14. optimistic update

```tsx
"use client";
import { useOptimistic } from "react";
import { addItem } from "@/app/actions";

export default function List({ items }) {
  const [optimistic, addOptimistic] = useOptimistic(
    items,
    (state, newItem) => [...state, newItem]
  );

  async function add(formData) {
    const newItem = { id: "temp", title: formData.get("title") };
    addOptimistic(newItem);
    await addItem(formData);
  }

  return (
    <>
      <ul>{optimistic.map(i => <li>{i.title}</li>)}</ul>
      <form action={add}>
        <input name="title" />
        <button>추가</button>
      </form>
    </>
  );
}
```

→ 서버 응답 전 UI 미리 업데이트. 실패 시 자동 롤백.

---

## 15. 함정

| 증상 | 원인 |
| --- | --- |
| `"use server"` 안 인식 | 파일 또는 함수 첫 줄 명시 |
| Server Action 이 외부 호출 가능 | Endpoint 처럼 노출. 인증 / 권한 체크 필수 |
| FormData 가 String 만 | 숫자 → `Number(formData.get(x))` |
| 빈 form 도 제출 | `required` / validation |
| redirect 후 코드 실행 X | redirect 가 throw 함 |

---

## 16. 다음 단계

1. **[[../api-routes]]** — API endpoint 만들기
2. **[[../deploy/vercel]]** — 배포

---

연결: [[../nextjs|↑ hub]] · [[server-components]] · [[data-fetching]]
