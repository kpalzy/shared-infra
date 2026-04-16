# shared-infra/AGENTS.md

로컬 공용 인프라 관리 규칙. 개별 앱의 AGENTS.md/CLAUDE.md와 별개로 항상 읽어라.

---

## 역할

이 디렉터리는 여러 앱이 공유하는 인프라를 관리한다.

```
shared-infra/
├── caddy/      — shared_caddy 컨테이너 (포트 80/443, 프로덕션 라우팅)
├── postgres/   — shared_postgres 컨테이너 (포트 5432, DB 공유)
└── docs/       — 운영 참고 문서
    └── docker-macos-settings.md  — macOS Docker 스토리지 구조 및 영속성 주의사항
```

---

## 포트 관리 — 반드시 먼저 읽을 것

**새 앱 추가, 서버 실행, 포트 변경 전에 반드시 확인:**
→ [`../shared-infra/PORTS.md`](../shared-infra/PORTS.md)

PORTS.md는 이 workspace의 **포트 단일 진실**이다.
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
docker compose restart
```

### 주의
```
❌ 로컬 dev 디버깅 중 Caddyfile 건드리지 말 것
❌ shared_caddy 컨테이너를 내리면 80포트 서비스 전체 중단
✅ 앱 컨테이너가 내려가도 caddy는 502 반환할 뿐 — caddy 자체는 내리지 않아도 됨
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
> 자세한 내용: [`docs/docker-macos-settings.md`](docs/docker-macos-settings.md)

### 시작 / 상태 확인 / 재시작

```bash
# 상태 확인
docker ps -a --filter name=shared_postgres

# 시작 (처음 or 내려가 있을 때)
cd /path/to/shared-infra/postgres
docker compose up -d

# 중지 (데이터 보존)
docker compose stop

# 로그 확인
docker logs shared_postgres --tail 30
```

> **컨테이너 런타임**: Rancher Desktop(nerdctl/docker) 사용.
> 앱 기동 전 반드시 Rancher Desktop이 실행 중인지 확인할 것.
> `docker ps` 가 에러를 내면 Rancher Desktop을 먼저 시작해야 한다.

### 새 앱 DB 추가

```bash
docker exec -it shared_postgres psql -U shared_pg_su -d postgres \
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

## 운영 체크리스트 (새 앱 배포 시)

1. `PORTS.md`에 포트 등록 확인
2. 앱 컨테이너 `shared` 네트워크 연결 확인
3. `caddy/Caddyfile`에 라우팅 추가
4. `docker compose restart` (caddy)
5. 앱 `.env`의 `CORS_ORIGINS`에 운영 도메인 포함 확인
