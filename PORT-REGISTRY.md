# Local Dev Port Registry

> **이 파일은 단일 진실(Single Source of Truth)이다.**
> 새 앱을 만들거나 서버를 띄우기 전, 반드시 여기서 포트를 확인하고 등록하라.

---

## 왜 이 파일이 필요한가

이 디렉터리(`genai/`)에는 여러 앱이 동시에 개발된다. 각 앱은 프론트엔드·백엔드 서버를 로컬에서 실행하는데, **포트가 겹치면 다른 앱의 서버가 요청을 가로채 디버깅이 매우 어려워진다.**

실제 발생했던 문제:
- `samil_workforce_hub`의 uvicorn이 호스트 `8000`번을 점유한 상태에서
- `kepture` 백엔드도 `8000`번으로 포트 포워딩 → 로그인 요청이 엉뚱한 앱으로 가서 404
- 증상만 보면 "인증 버그"처럼 보여 원인 찾는 데 시간이 걸림

**포트를 여기서 미리 배정하면 이런 충돌을 사전에 막을 수 있다.**

---

## 로컬 개발 구조

```
[브라우저]
    │
    ├─ localhost:5173  →  Vite dev server (kepture, 호스트 프로세스)
    │       └─ /api/* 요청  →  Vite proxy  →  localhost:8001 (kepture 백엔드 컨테이너)
    │
    ├─ localhost:15173 →  samil_workforce_hub 프론트 (컨테이너) or Vite 직접
    │       └─ API 요청  →  localhost:18000 (samil 백엔드 컨테이너)
    │
    └─ localhost:80   →  shared_caddy  →  kepture-frontend-1:80 (프로덕션 전용)

[공용 인프라 — shared-infra/]
    shared_caddy     : 80, 443  (프로덕션 라우팅)
    shared_postgres  : 5432     (컨테이너 내부 전용, 호스트 미노출)
```

### 로컬 개발 vs 운영 배포 차이

| 구분 | 로컬 개발 (dev) | 운영 배포 (prod) |
|------|----------------|-----------------|
| 프론트 진입 | `localhost:{전용 포트}` 직접 | `localhost:80` → shared_caddy → 컨테이너 |
| 백엔드 진입 | `localhost:{전용 포트}` 직접 or Vite proxy | shared_caddy가 내부 컨테이너로 라우팅 |
| HMR / React DevTools | Vite dev server로 가능 | 불가 (빌드된 정적 파일) |
| CORS | Vite proxy 덕분에 same-origin | 컨테이너 CORS_ORIGINS 설정 필요 |
| 포트 충돌 위험 | 높음 (여러 앱 동시 실행) | 낮음 (caddy가 단일 진입점) |

---

## 공용 인프라 포트

| 서비스 | 호스트 포트 | 비고 |
|--------|-------------|------|
| shared_caddy | 80, 443 | 프로덕션 라우팅 전용. **dev에서는 건드리지 말 것** |
| shared_postgres | 5432 | 컨테이너 내부 전용. 호스트 미노출 |

설정: [shared-infra/caddy/docker-compose.yml](shared-infra/caddy/docker-compose.yml)

---

## 앱별 포트 배정

| 앱 | Frontend (dev) | Backend (host) | 실행 방식 | 설정 파일 |
|----|---------------|----------------|-----------|-----------|
| **kepture** | **5173** | **8001** | Vite dev + docker 컨테이너 | [kepture/docker-compose.yml](kepture/docker-compose.yml) · [kepture/frontend/vite.config.ts](kepture/frontend/vite.config.ts) |
| **samil_workforce_hub** | **5174** (Vite dev) · 15173 (컨테이너) | **18000** (컨테이너) · ⚠️ 8000 (uvicorn 직접) | compose or uvicorn 직접 | [samil_workforce_hub/docker-compose.yml](samil_workforce_hub/docker-compose.yml) |

> ⚠️ **samil 주의**: uvicorn을 호스트에서 직접 실행(`uvicorn main:app --port 8000`)하면
> compose의 18000이 아닌 8000번을 점유해 kepture와 충돌한다.
> 직접 실행 시 반드시 `--port 18000`으로 맞추거나, compose로 띄울 것.

---

## 다음 앱 추가 시 사용 가능한 포트

| 용도 | 사용 가능 포트 |
|------|---------------|
| Frontend (Vite dev) | 5175, 5176, 5177 ... |
| Backend (host) | 8002, 8003, 8004 ... |

---

## 새 프로젝트 추가 체크리스트

새 앱 개발을 시작할 때 아래 순서로 진행한다.

### 1. 포트 배정 (가장 먼저)
- 이 파일에서 비어있는 Frontend / Backend 포트 하나씩 골라 앱별 포트 표에 먼저 등록
- 등록 전에 `lsof -i :{포트번호}` 로 실제 사용 중인지 재확인

### 2. Vite 설정 고정 (`vite.config.ts`)
```ts
server: {
  port: 5175,        // 위에서 배정한 포트로
  strictPort: true,  // 포트 충돌 시 자동 증가 대신 에러 — 충돌을 즉시 알 수 있음
  proxy: {
    '/api': {
      target: 'http://localhost:8002',  // 배정된 백엔드 포트
      changeOrigin: true,
    },
  },
},
```

### 3. API 클라이언트 설정
```ts
// .env.local (Vite dev 전용, git 제외)
VITE_API_URL=/api/v1
```
```ts
// .env (기본값, git 포함)
VITE_API_URL=http://localhost:8002/api/v1
```
개발 시 `.env.local`의 상대경로(`/api/v1`)를 우선 사용 → Vite proxy 경유 → CORS 이슈 없음.

### 4. 컨테이너 백엔드 포트 고정 (`docker-compose.yml`)
```yaml
ports:
  - "8002:8000"   # 호스트:컨테이너. 호스트 포트만 앱마다 다르게
```

### 5. CORS 설정 (`.env`)
```
CORS_ORIGINS=["http://localhost:5175","http://localhost","https://yourdomain.com"]
```
Vite proxy를 쓰면 브라우저 요청은 same-origin이라 CORS 불필요하지만,
운영 배포 전환 시를 대비해 명시해두는 것이 좋다.

### 6. 운영 배포 시 추가 작업
- `shared-infra/caddy/Caddyfile`에 새 앱 라우팅 추가
  ```
  yourdomain.com {
      reverse_proxy new-app-frontend-1:80
  }
  ```
- `shared_caddy` 컨테이너 재시작: `cd shared-infra/caddy && docker compose restart`
- `.env`의 `CORS_ORIGINS`에 운영 도메인 추가 확인

---

## 주의사항 요약

```
❌ 포트 배정 없이 기본값(8000, 5173)으로 서버 실행
❌ Vite strictPort 없이 실행 (충돌을 모르고 넘어감)
❌ 호스트에서 uvicorn/node 직접 실행 시 포트 미확인
❌ 새 앱 추가 후 이 파일 미업데이트

✅ 새 앱 시작 전 이 파일에 포트 먼저 등록
✅ Vite: port + strictPort: true 명시
✅ 백엔드: docker-compose.yml 호스트 포트 고정
✅ 배포 전환 시 Caddyfile + CORS_ORIGINS 동시 업데이트
```
