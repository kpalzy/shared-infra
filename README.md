# genai/ — 로컬 개발 워크스페이스

개인 AI/웹 앱 개발 프로젝트들이 모여있는 루트 디렉터리.
공용 인프라를 공유하며 여러 앱을 동시에 개발하는 환경이다.

---

## 디렉터리 구조

```
genai/
├── PORT-REGISTRY.md          ← 포트 레지스트리 (필독 — 새 앱 추가 전 항상 확인)
├── shared-infra/             ← 공용 인프라 (Caddy, Postgres)
│   ├── README.md             ← 지금 여기
│   ├── AGENTS.md             ← 인프라 운영 규칙
│   ├── PORT-REGISTRY.md      ← 포트 레지스트리 (필독 — 새 앱 추가 전 항상 확인)
│   ├── caddy/                ← shared_caddy (포트 80/443)
│   ├── postgres/             ← shared_postgres (포트 5432)
│   └── docs/                 ← 운영 참고 문서
│       └── DOCKER-VOLUME-STRATEGY.md
│
├── kepture/                  ← 쇼핑 리서치 큐레이션 앱
├── samil_workforce_hub/      ← 인력 운영 관리 앱
└── (기타 실험/프로토타입 프로젝트들)
```

---

## 핵심 문서 2개

| 파일 | 역할 |
|------|------|
| **[PORT-REGISTRY.md](shared-infra/PORT-REGISTRY.md)** | 앱별 포트 배정 현황 + 새 앱 추가 체크리스트 |
| **[shared-infra/AGENTS.md](shared-infra/AGENTS.md)** | Caddy·Postgres 운영 규칙, 배포 시 Caddyfile 수정법 |
| **[shared-infra/docs/DOCKER-VOLUME-STRATEGY.md](shared-infra/docs/DOCKER-VOLUME-STRATEGY.md)** | 환경별 Docker 볼륨 전략 (macOS/Linux/클라우드/K8s) |

---

## AGENTS.md 관리 규칙

### 심볼릭 링크 패턴

각 프로젝트 루트에는 **`AGENTS.md` 하나만 작성**하고,
`CLAUDE.md`와 `GEMINI.md`는 이를 가리키는 심볼릭 링크로 둔다.

```
프로젝트/
├── AGENTS.md       ← 실제 파일 (여기에만 쓴다)
├── CLAUDE.md -> AGENTS.md   ← 심볼릭 링크 (Claude Code가 읽음)
└── GEMINI.md -> AGENTS.md   ← 심볼릭 링크 (Gemini CLI가 읽음)
```

새 프로젝트에서 심볼릭 링크 만드는 법:
```bash
cd 새프로젝트/
ln -s AGENTS.md CLAUDE.md
ln -s AGENTS.md GEMINI.md
```

이렇게 해두면 `AGENTS.md`만 수정해도 Claude·Gemini 양쪽에 자동 반영된다.

### AGENTS.md에 반드시 포함할 것

새 프로젝트의 `AGENTS.md`를 쓸 때 아래 항목을 빠뜨리지 말 것:

```
1. 프로젝트 한 줄 정의
2. 기술 스택 (언어, 프레임워크, 컨테이너 런타임)
3. 로컬 개발 포트 — Frontend / Backend 포트 명시 + PORT-REGISTRY.md 링크
4. 디렉터리 구조 / 핵심 파일 위치
5. 개발 명령어 (dev 서버, 빌드, 테스트, 마이그레이션)
6. 절대 하지 말 것 (금지 패턴, 구현 금지 사항)
7. 하위 AGENTS.md 경로 (backend/, frontend/ 등)
```

---

## 새 앱 추가 체크리스트

새 프로젝트를 시작할 때 순서대로 진행한다.

### 1단계 — 포트 배정
- [ ] [PORT-REGISTRY.md](PORT-REGISTRY.md) 열어서 비어있는 Frontend / Backend 포트 확인
- [ ] `lsof -i :{포트}` 로 실제 사용 중인지 확인
- [ ] PORT-REGISTRY.md 앱별 포트 표에 등록

### 2단계 — 프로젝트 세팅
- [ ] `AGENTS.md` 작성 (위 필수 항목 포함)
- [ ] 심볼릭 링크 생성: `ln -s AGENTS.md CLAUDE.md && ln -s AGENTS.md GEMINI.md`
- [ ] `vite.config.ts`에 `port` + `strictPort: true` 명시
- [ ] `docker-compose.yml`에 백엔드 호스트 포트 고정 (`"8002:8000"` 형식)
- [ ] `.env.local` 생성 (`VITE_API_URL=/api/v1`, git 제외)

### 3단계 — 공용 인프라 연결
- [ ] `docker-compose.yml`에 `shared` 네트워크 연결 확인
  ```yaml
  networks:
    default: {}
    shared:
      external: true
  ```
- [ ] `shared_postgres`에 새 database 생성
  ```bash
  docker exec -it shared_postgres psql -U shared_pg_su -d postgres -c "CREATE DATABASE 앱이름;"
  ```

### 4단계 — 운영 배포 시 (선택)
- [ ] `shared-infra/caddy/Caddyfile`에 라우팅 추가
- [ ] `docker compose restart` (shared-infra/caddy/)
- [ ] `.env`의 `CORS_ORIGINS`에 운영 도메인 추가

---

## 로컬 개발 환경

- **컨테이너 런타임**: Rancher Desktop + `docker` / `docker compose`
- **런타임 관리**: `mise` ([mise.toml](mise.toml) — Node, Python 버전)
- **공용 네트워크**: `shared` (docker network)

공용 네트워크가 없으면 먼저 생성:
```bash
docker network create shared
```

> ⚠️ **macOS 데이터 영속성**: Rancher Desktop은 Linux VM 위에서 동작한다.
> Docker named volume은 VM 재시작 시 소실되므로, **stateful 서비스(postgres 등)는 반드시 bind mount(`./data`)를 사용**할 것.
> 자세한 내용: [`docs/DOCKER-VOLUME-STRATEGY.md`](docs/DOCKER-VOLUME-STRATEGY.md)
