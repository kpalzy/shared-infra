# shared-infra/AGENTS.md

로컬 공용 인프라 관리 규칙. 개별 앱의 AGENTS.md/CLAUDE.md와 별개로 항상 읽어라.

> **참고**: `CLAUDE.md`와 `GEMINI.md`는 이 파일(`AGENTS.md`)의 심링크다.
> 세 파일 모두 동일한 내용이며, AI 에이전트 종류에 무관하게 이 파일 하나로 관리한다.

---

## 역할

이 디렉터리는 여러 앱이 공유하는 인프라를 관리한다.

```
shared-infra/
├── AGENTS.md   — 이 파일 (CLAUDE.md, GEMINI.md는 이 파일의 심링크)
├── DOCKER-VOLUME-STRATEGY.md — 환경별 Docker 볼륨 전략 (macOS/Linux/클라우드/K8s)
├── caddy/      — shared_caddy 컨테이너 (포트 80/443, 프로덕션 라우팅)
│   ├── Caddyfile                  — 라우팅 설정 (git 관리, 시크릿 없이 플레이스홀더만)
│   ├── caddy.secrets.env          — 실제 시크릿 (gitignore — 절대 커밋 금지)
│   └── caddy.secrets.env.example  — 시크릿 템플릿 (git 관리, 더미값)
└── postgres/   — shared_postgres 컨테이너 (포트 5432, DB 공유)
```

---

## Docker 셋업 — 반드시 먼저 읽을 것

**새 맥 세팅, Docker 컨테이너 추가, compose 파일 작성 전에 반드시 확인:**
→ [`DOCKER-VOLUME-STRATEGY.md`](DOCKER-VOLUME-STRATEGY.md)

macOS(Rancher Desktop) 환경에서 Docker를 올바르게 쓰려면 data-root 설정과 bind mount 전략을 반드시 이해해야 한다.
이 문서를 읽지 않고 Docker를 세팅하면 빌드가 터지거나 DB 데이터가 날아간다.

---

## 포트 관리 — 반드시 먼저 읽을 것

**새 앱 추가, 서버 실행, 포트 변경 전에 반드시 확인:**
→ [`PORT-REGISTRY.md`](PORT-REGISTRY.md)

PORT-REGISTRY.md는 이 workspace의 **포트 단일 진실**이다.
거기서 배정받지 않은 포트를 임의로 사용하지 말 것.

---

## shared_caddy

- **역할**: 프로덕션/스테이징 라우팅 전용. 로컬 dev에서는 직접 쓰지 않는다.
- **설정 파일**: `caddy/Caddyfile`
- **현재 라우팅**: `localhost:80` → `kepture-frontend-1:80`

### 새 앱을 배포할 때 Caddyfile 수정

```
# caddy/Caddyfile 예시
:80 {
    reverse_proxy kepture-frontend-1:80
}

newapp.yourdomain.com {
    reverse_proxy newapp-frontend-1:80
}
```

수정 후 반드시 재시작:
```bash
cd shared-infra/caddy
nerdctl compose restart
```

### Caddyfile 시크릿 관리 규칙

Caddyfile은 git으로 관리한다. 단, **시크릿(API 토큰, 비밀번호 등)은 절대 직접 쓰지 말고 환경변수 플레이스홀더로 분리**한다.

#### 파일 구조

| 파일 | git 관리 | 용도 |
|------|----------|------|
| `Caddyfile` | ✅ 커밋 | 라우팅 설정. `{$VAR}` 플레이스홀더만 사용 |
| `caddy.secrets.env` | ❌ gitignore | 실제 시크릿 값. 머신마다 직접 생성 |
| `caddy.secrets.env.example` | ✅ 커밋 | 어떤 시크릿이 필요한지 문서화한 템플릿 |

#### Caddyfile 작성 방법

시크릿은 `{$변수명}` 플레이스홀더로 작성:

```
yourdomain.com {
    tls {
        dns cloudflare {$CF_API_TOKEN}
    }
    reverse_proxy kepture-frontend-1:80
}
```

> **`{$VAR}` vs `{env.VAR}`**: `{$VAR}`는 파싱 시 치환(모든 디렉티브 지원), `{env.VAR}`는 런타임 치환(일부 디렉티브만 지원). Caddyfile에서는 `{$VAR}` 사용을 권장.

#### caddy.secrets.env 형식

```bash
# DNS 인증 (Let's Encrypt wildcard 등)
CF_API_TOKEN=실제_클라우드플레어_토큰

# basic_auth 해시 — bcrypt 해시의 $ 는 반드시 $$ 로 이스케이프
# 해시 생성: docker run --rm caddy caddy hash-password --plaintext "비밀번호"
BASIC_AUTH_HASH=$$2a$$12$$...
```

> ⚠️ **`basic_auth` 주의**: bcrypt 해시는 `$2a$12$...` 형태인데, env 파일에서 `$`를 `$$`로 이스케이프하지 않으면 Docker가 변수로 해석해 해시가 깨진다.

#### autosave.json 주의

Caddy는 `{$VAR}` 치환된 **실제 값**을 `caddy/config/` 아래 `autosave.json`에 저장한다.  
→ `caddy/config/`는 이미 `.gitignore`에 등록되어 있으므로 별도 조치 불필요.

#### 새 머신 셋업 시

```bash
cd shared-infra/caddy
cp caddy.secrets.env.example caddy.secrets.env
# caddy.secrets.env 열어서 실제 값 입력
docker compose up -d
```

### 주의
```
❌ 로컬 dev 디버깅 중 Caddyfile 건드리지 말 것
❌ shared_caddy 컨테이너를 내리면 80포트 서비스 전체 중단
❌ Caddyfile에 API 토큰, 비밀번호 등 시크릿 직접 입력 금지
❌ caddy.secrets.env 절대 커밋 금지
✅ 앱 컨테이너가 내려가도 caddy는 502 반환할 뿐 — caddy 자체는 내리지 않아도 됨
✅ 시크릿은 반드시 {$VAR} 플레이스홀더로 분리 → caddy.secrets.env에 보관
✅ 새 시크릿 추가 시 caddy.secrets.env.example도 함께 업데이트
```

---

## shared_postgres

- **역할**: 여러 앱이 DB를 공유. 앱마다 별도 database name 사용.
- **호스트 포트**: `5432` (docker-compose.yml에서 `"5432:5432"` 노출 — 호스트에서도 `localhost:5432` 접근 가능)
- **컨테이너 접근**: 각 앱 컨테이너에서 hostname `shared_postgres`로 접근
- **설정 파일**: `postgres/docker-compose.yml`
- **데이터 위치**: `postgres/data/` (bind mount → macOS SSD에 영속 저장)

> ⚠️ **macOS 데이터 영속성 주의**: Rancher Desktop은 Linux VM 위에서 동작하며, Docker named volume은 VM 재시작 시 소실된다.
> 이 때문에 postgres는 named volume 대신 `./data` bind mount를 사용한다. VM 재시작 후에도 데이터가 유지된다.
> 자세한 내용: [`DOCKER-VOLUME-STRATEGY.md`](DOCKER-VOLUME-STRATEGY.md)

### 시작 / 상태 확인 / 재시작

```bash
# 상태 확인
nerdctl ps -a --filter name=shared_postgres

# 시작 (처음 or 내려가 있을 때)
cd /path/to/shared-infra/postgres
nerdctl compose up -d

# 중지 (데이터 보존)
nerdctl compose stop

# 로그 확인
nerdctl logs shared_postgres --tail 30
```

> **컨테이너 런타임**: Rancher Desktop(containerd/nerdctl) 사용. 개발계 기준.
> 앱 기동 전 반드시 Rancher Desktop이 실행 중인지 확인할 것.
> `nerdctl ps` 가 에러를 내면 Rancher Desktop을 먼저 시작해야 한다.

### 새 앱 DB 추가

```bash
nerdctl exec -it shared_postgres psql -U shared_pg_su -d postgres \
  -c "CREATE DATABASE newapp;"
```

### 주의
```
❌ shared_postgres 컨테이너를 내리면 모든 앱 DB 접근 불가
❌ 앱마다 postgres 컨테이너를 별도로 띄우지 말 것 — 공유 인프라 활용
❌ brew/homebrew로 postgres를 별도 설치하지 말 것 — 이 컨테이너로 통일
❌ postgres에 named volume 사용 금지 — macOS VM 재시작 시 데이터 소실
✅ hostname 충돌 주의: 컨테이너 이름이 shared_postgres로 고정되어 있음
✅ Rancher Desktop 실행 여부를 먼저 확인 후 docker 명령 실행
✅ postgres/data/ 는 .gitignore에 등록됨 — git에 올라가지 않음
```

---

## Rancher Desktop — 필수 초기 설정

> macOS에서 Rancher Desktop을 쓴다면 아래를 새 맥 세팅 시 반드시 먼저 설정해야 한다.

### 0. 컨테이너 엔진 및 Kubernetes 설정

**컨테이너 엔진: containerd — 개발계 기준**

`Preferences → Container Engine → General` → **containerd** 선택.

- **현재 개발계는 containerd 사용 중** → CLI는 `nerdctl` / `nerdctl compose`
- K8s 1.24+ 표준 런타임. k3s와 이미지 스토어 공유 → K8s 개발로 자연스럽게 이어짐
- dockerd는 K8s 1.24에서 공식 제거됨. 나중에 K8s 쓸 계획이 있다면 처음부터 containerd 권장
- 리소스 차이는 미미 (무거운 건 Lima VM 자체)
- Rancher Desktop은 containerd 모드에서도 `/var/run/docker.sock` docker compat socket을 노출 → Promtail 등 docker socket 의존 도구 정상 작동

**Kubernetes 비활성화 (RAM 500MB 절약)**

`Preferences → Kubernetes → Enable Kubernetes` **체크 해제**.  
docker compose 기반 개발에는 Kubernetes가 필요 없다. 비활성화만 해도 RAM 약 500MB 절약.

### 1. VM 디스크 및 data-root

RD v1.22+에서 `/var/lib`(Docker 데이터 전체)은 tmpfs가 아닌 **100GB data-volume(diffdisk)**에 마운트된다.  
`no space left on device` 문제는 기본값으로 해결된다. **별도 설정 불필요.**

- **containerd**: data-root 변경 불가 (config.toml 재시작 시 덮어쓰임). 기본값으로 충분
- **dockerd**: GUI `dockerd options`로 `~/docker-data` 설정 가능 — diffdisk 밖 Mac SSD에 직접 저장. 단 virtiofs 경유 특성상 prune 후 즉각 공간 반환은 미보장 (containerd와 실질 차이 불분명)
- **공통**: 어느 엔진이든 정기 prune은 필수. SSD 점유가 줄지 않으면 `rdctl shell sudo fstrim /var/lib` 시도, 근본 해결은 factory-reset

이미지/캐시가 쌓이면 주기적으로 정리:
```bash
nerdctl system prune -af   # containerd
docker system prune -af    # dockerd
```

> 상세 내용: [`DOCKER-VOLUME-STRATEGY.md`](DOCKER-VOLUME-STRATEGY.md)

### 2. stateful 서비스는 bind mount 사용 (필수)

named volume은 VM 재시작 후 유지되지만, `rdctl factory-reset` 시 소멸된다.  
모든 DB 컨테이너는 `./data` bind mount를 써야 factory-reset 후에도 데이터가 Mac SSD에 남는다.

> 상세 내용: [`DOCKER-VOLUME-STRATEGY.md`](DOCKER-VOLUME-STRATEGY.md)

---

## 컨테이너 네트워크

모든 앱 컨테이너는 `shared` 네트워크에 연결해야 shared_caddy, shared_postgres에 접근 가능하다.

```yaml
# 각 앱의 docker-compose.yml 필수 설정
networks:
  default: {}
  shared:
    external: true
```

---

## 모니터링 (Grafana + Loki + Promtail)

- **역할**: 컨테이너 로그 수집 + DB 모니터링 + Telegram 알림
- **설정 파일**: `monitoring/docker-compose.yml`
- **데이터 위치**: `monitoring/grafana/data/`, `monitoring/loki/data/` (bind mount, gitignore)

### 구성

| 컨테이너 | 역할 | 포트 |
|----------|------|------|
| shared_grafana | 대시보드 + 알림 | 3000 (호스트) |
| shared_loki | 로그 저장 | 3100 (내부) |
| shared_promtail | 컨테이너 로그 → Loki 전송 | - |

### 시작

```bash
cd shared-infra/monitoring
cp .env.example .env
# .env 열어서 GF_ADMIN_PASSWORD, GF_PG_PASSWORD 입력
# Telegram 알림 원하면 TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID 도 입력
nerdctl compose up -d
# localhost:3000 접속 (admin / .env에 설정한 비밀번호)
```

### 자동 프로비저닝 (UI 설정 불필요)

- 데이터소스: `grafana/provisioning/datasources/` — Loki + PostgreSQL 자동 등록
- 알림 채널: `grafana/provisioning/alerting/contact-points.yml` — **현재 주석 처리됨** (Telegram 토큰 설정 후 활성화)
- 알림 규칙: `grafana/provisioning/alerting/alert-rules.yml` — 에러 급증, 컨테이너 다운, DB 커넥션

### Telegram 알림 활성화 방법

1. `.env`에 `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` 입력
2. `grafana/provisioning/alerting/contact-points.yml` 주석 해제
3. `nerdctl compose restart grafana`

### Loki 로그 보관

- 보관 기간: **14일** (`loki/loki-config.yml` `retention_period: 14d`)
- 삭제 방식: compactor 10분 간격 정리 (`delete_request_store: filesystem`)
- 변경 시: `loki-config.yml` 수정 → `nerdctl compose restart loki`

### 주의
```
❌ grafana/data/, loki/data/ 커밋 금지 (gitignore 등록됨)
❌ .env 커밋 금지 (Telegram 토큰 + DB 비밀번호 포함)
✅ Telegram 활성화 전에도 Grafana 대시보드 + 로그 조회는 동작함
✅ 알림 규칙 변경 시 YAML 수정 → nerdctl compose restart grafana
✅ 새 컨테이너 추가 시 Promtail이 자동 감지 (docker compat socket 기반)
```

---

## 운영 체크리스트 (새 앱 배포 시)

1. `PORT-REGISTRY.md`에 포트 등록 확인
2. 앱 컨테이너 `shared` 네트워크 연결 확인
3. `caddy/Caddyfile`에 라우팅 추가
4. `docker compose restart` (caddy)
5. 앱 `.env`의 `CORS_ORIGINS`에 운영 도메인 포함 확인
