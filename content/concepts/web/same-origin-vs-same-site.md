---
title: Same-Origin vs Same-Site
type: concept
tags: [web, browser, security, cors, cookie]
related: []
last_reviewed: 2026-05-03
publish: false
---

# Same-Origin vs Same-Site

## 목차

- [한 줄 정의](#한-줄-정의)
- [등장 배경](#등장-배경)
- [URL 구성 요소](#url-구성-요소)
- [Same-Origin 정의](#same-origin-정의)
- [Same-Site 정의](#same-site-정의)
- [두 개념의 차이](#두-개념의-차이)
- [어디에 어떤 기준이 적용되는가](#어디에-어떤-기준이-적용되는가)
- [실무 영향](#실무-영향)
- [함정 / 자주 하는 오해](#함정--자주-하는-오해)
- [사고 모델](#사고-모델)
- [참고](#참고)

## 한 줄 정의

브라우저가 **두 리소스가 "같은 출처"인지 판정하는 두 가지 기준**. Same-Origin은 엄격(scheme + host + port 일치), Same-Site는 느슨(등록 도메인 일치).

> ⚠️ **자주 하는 오해**: "Same-Origin과 Same-Site는 거의 같은 말이다" — **아니다.**
> 두 개념은 **적용되는 영역이 다르다**. JavaScript 접근 권한·CORS는 Same-Origin, 쿠키는 Same-Site 기준으로 판정된다.

## 등장 배경

웹은 **여러 도메인의 자원이 한 페이지에 섞이는** 환경이다. 브라우저는 보안을 위해 출처가 다른 자원 간의 상호작용을 제한해야 한다. 그런데 *"다르다"* 의 기준은 맥락에 따라 달라야 했다:

- **JS가 다른 페이지의 DOM을 조작**할 수 있는가? → 매우 엄격해야 함 (보안 위협 큼)
- **쿠키가 서브도메인 간에 공유**되는가? → 어느 정도 느슨해도 됨 (편의성 필요)

이 두 요구를 동시에 만족시키기 위해 **두 가지 기준**이 만들어졌다.

## URL 구성 요소

두 개념을 이해하려면 URL이 어떤 부품으로 분해되는지부터 알아야 한다.

```
https://api.example.com:443/path?q=1
└─┬─┘   └─────┬───────┘ └┬┘
scheme      host        port
└──────────┬───────────┘
       Origin (3개 합친 것)

         example.com  ← eTLD+1 (등록 도메인)
         └────┬─────┘
            Site
```

| 부품 | 예시 |
|------|------|
| **scheme** | `https`, `http` |
| **host** | `api.example.com` (서브도메인 포함) |
| **port** | `443`, `80`, `8080` |
| **eTLD+1** | `example.com` (등록 도메인) |

### eTLD+1이란

- **eTLD** (effective Top-Level Domain): `.com`, `.co.kr`, `.github.io` 같은 공용 접미사
- **eTLD+1**: eTLD 바로 앞 라벨 한 개를 포함한 부분

브라우저는 [Public Suffix List](https://publicsuffix.org/)를 기반으로 eTLD를 판정한다.

| 도메인 | eTLD | eTLD+1 |
|--------|------|--------|
| `www.example.com` | `.com` | `example.com` |
| `api.example.com` | `.com` | `example.com` |
| `mall.shop.co.kr` | `.co.kr` | `shop.co.kr` |
| `user.github.io` | `.github.io` | `user.github.io` |

## Same-Origin 정의

**scheme + host + port가 모두 정확히 일치**해야 같은 origin.

```
✅ Same-Origin
https://example.com   ↔ https://example.com

❌ Different-Origin
https://example.com   ↔ http://example.com          (scheme 다름)
https://example.com   ↔ https://api.example.com     (host 다름)
https://example.com   ↔ https://example.com:8080    (port 다름)
```

서브도메인은 **Different-Origin**이라는 점에 주의.

## Same-Site 정의

**eTLD+1이 일치**하면 같은 site.

```
✅ Same-Site
https://example.com   ↔ https://api.example.com    (eTLD+1 = example.com)
https://example.com   ↔ https://cdn.example.com    (eTLD+1 = example.com)
https://example.com   ↔ https://example.com:8080   (port 무관)

❌ Different-Site
https://example.com   ↔ https://example.co.kr      (eTLD+1 다름)
https://a.github.io   ↔ https://b.github.io        (github.io가 Public Suffix)
```

> **Schemeful Same-Site**: 최신 브라우저는 scheme도 비교한다. `http://example.com`과 `https://example.com`은 **다른 site**로 취급된다.

## 두 개념의 차이

`example.com` ↔ `api.example.com` 비교로 핵심 차이를 보면:

|  | Same-Origin? | Same-Site? |
|---|---|---|
| `example.com` ↔ `api.example.com` | ❌ (host 다름) | ✅ (eTLD+1 같음) |

→ 서브도메인은 **"같은 사이트지만 다른 origin"** 인 상태가 흔하다.

### 엄격함 스펙트럼

```
가장 엄격 ─────────────────────────────── 가장 느슨
   │                                          │
   ▼                                          ▼
Same-Origin                              Same-Site
   │                                          │
   ├ scheme 같음                              ├ eTLD+1 같음
   ├ host 같음 (서브도메인 X)                  └ (서브도메인 OK)
   └ port 같음
```

## 어디에 어떤 기준이 적용되는가

이게 핵심이다. **두 개념은 서로 다른 영역에서 사용**된다.

| 영역 | 적용 기준 |
|------|-----------|
| `iframe` 내부 DOM 접근 (`contentWindow.document`) | **Same-Origin** |
| `window.postMessage` 수신자 검증 | **Same-Origin** |
| `XMLHttpRequest` / `fetch` 권한 (CORS) | **Same-Origin** |
| `localStorage`, `sessionStorage`, IndexedDB 접근 | **Same-Origin** |
| Service Worker 스코프 | **Same-Origin** |
| 쿠키 공유 (Domain 속성) | **Same-Site** |
| `SameSite=Lax` / `Strict` 속성 동작 | **Same-Site** |
| CSRF 보호 판정 (브라우저 휴리스틱) | **Same-Site** |

### 패턴

- **JavaScript의 직접 접근 권한** = Same-Origin (보안 위협 크므로 엄격)
- **쿠키·요청 컨텍스트 판정** = Same-Site (서브도메인 운영 편의 필요)

## 실무 영향

### 1. 서브도메인 간 fetch — CORS 필요, 쿠키는 공유 가능

```
example.com 페이지에서 api.example.com 호출
  ├ Same-Origin? ❌ → CORS 헤더 필요
  └ Same-Site?   ✅ → 쿠키 자동 동봉 가능
```

```javascript
// example.com 클라이언트
fetch('https://api.example.com/data', {
    credentials: 'include'  // Same-Site라 쿠키 동봉 가능
});
```

```http
# api.example.com 서버 응답
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Credentials: true
```

### 2. 서브도메인 간 쿠키 공유

`Domain` 속성을 부모 도메인으로 설정하면 서브도메인 전체에 공유된다.

```http
Set-Cookie: session_id=xxx; Domain=.example.com; Path=/; Secure; HttpOnly
```

→ `example.com`, `api.example.com`, `cdn.example.com` 모두 같은 쿠키 접근.

### 3. iframe DOM 접근 — Same-Origin 필요

```html
<!-- example.com 페이지 -->
<iframe src="https://api.example.com/widget"></iframe>

<script>
  const iframe = document.querySelector('iframe');
  // Different-Origin이라 직접 접근 불가
  iframe.contentWindow.document  // SecurityError ❌
</script>
```

서브도메인이라도 **DOM 직접 접근은 막힌다**. `postMessage`로 통신해야 한다.

### 4. localStorage는 Origin별 분리

```
example.com의 localStorage     ≠  api.example.com의 localStorage
                                ≠  http://example.com의 localStorage
                                ≠  example.com:8080의 localStorage
```

서브도메인 간 데이터 공유가 필요하면 쿠키 또는 서버 세션을 써야 한다.

### 5. SameSite 쿠키 속성

| 값 | 동작 |
|----|------|
| `Strict` | Same-Site 요청에만 쿠키 동봉. 외부 사이트에서의 링크 클릭에도 동봉 안 됨 |
| `Lax` | Same-Site 요청 + 외부에서의 top-level 네비게이션(GET) 시 동봉 |
| `None` | 모든 요청에 동봉 (단 `Secure` 필수) |

최신 브라우저 기본값은 `Lax`.

## 함정 / 자주 하는 오해

### ① "서브도메인이면 다 1st-party니까 자유롭게 통신된다"

**반은 맞고 반은 틀림.** 쿠키는 공유되지만 JS 직접 접근·fetch는 별개.

- 쿠키 공유 ✅ (Same-Site)
- DOM 직접 접근 ❌ (Different-Origin)
- fetch 시 CORS 필요 (Different-Origin)

### ② "같은 회사 도메인이면 Same-Site다"

**아니다.** 브라우저는 회사가 아니라 도메인을 본다.

```
naver.com ↔ navercorp.com   → 다른 eTLD+1 → Different-Site ❌
naver.com ↔ blog.naver.com  → 같은 eTLD+1 → Same-Site ✅
```

같은 회사가 운영하는 별도 도메인 간에는 **공식적인 Site 관계가 없다**. 필요하다면 [First-Party Sets](https://developer.chrome.com/docs/privacy-sandbox/first-party-sets/) 같은 별도 메커니즘이 필요하다.

### ③ "포트가 다른데 같은 origin 아닌가?"

**아니다.** `https://example.com`과 `https://example.com:8080`은 **Different-Origin**. 개발 환경에서 자주 만나는 함정.

### ④ "http와 https는 같은 사이트 아닌가?"

**Schemeful Same-Site 이전엔 같았지만, 최신 브라우저에서는 다른 site**. `http://example.com`과 `https://example.com`은 Different-Site.

### ⑤ "Public Suffix List에 있는 도메인은 항상 eTLD다"

`github.io`, `vercel.app`, `netlify.app` 같은 호스팅 플랫폼 도메인은 Public Suffix로 등록되어 있다. 그래서:

```
user-a.github.io ↔ user-b.github.io → Different-Site
```

같은 호스팅 서비스를 쓰는 서로 다른 사용자의 사이트가 **격리된 site로 취급**되도록 설계된 것이다.

## 사고 모델

> **두 개념은 서로 다른 보안 트레이드오프를 다룬다.**
>
> - **Same-Origin**: "JS가 다른 페이지를 침범할 수 있는가?" — 침범의 위험이 크므로 **엄격**
> - **Same-Site**: "이 요청이 같은 서비스 컨텍스트에서 발생했는가?" — 서비스 운영 편의를 위해 **느슨**

핵심 직관:

```
"같은 사이트지만 다른 origin"  ← 서브도메인의 흔한 상태
                                  쿠키는 공유, JS 접근은 격리
```

이 모델을 잡으면 다음이 자연스러워진다:

- 서브도메인에 API 서버를 두면 쿠키 공유는 되지만 CORS 설정은 필요하다
- iframe으로 같은 회사의 다른 도메인 콘텐츠를 띄워도 DOM 직접 조작은 못 한다
- localStorage로 서브도메인 간 데이터 공유는 안 된다 (쿠키나 서버 경유 필요)

## 참고

- [MDN — Same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)
- [web.dev — Understanding "same-site" and "same-origin"](https://web.dev/articles/same-site-same-origin)
- [RFC 6454 — The Web Origin Concept](https://datatracker.ietf.org/doc/html/rfc6454)
- [Public Suffix List](https://publicsuffix.org/)
- [HTTP State Management Mechanism (RFC 6265)](https://datatracker.ietf.org/doc/html/rfc6265)
