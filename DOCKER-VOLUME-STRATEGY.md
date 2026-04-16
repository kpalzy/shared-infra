# Docker 팁 — macOS 로컬 운영 & 초기 설정 가이드

> 작성일: 2026-04-16  
> 환경: macOS + Rancher Desktop v1.22 + Apple VZ (vz) + virtiofs

---

## TL;DR — 결론부터

macOS에서 Rancher Desktop v1.22+을 쓸 때 핵심 설정 요약.

**0. Rancher Desktop 초기 설정 (새 맥 세팅 시 가장 먼저)**
- `Preferences → Kubernetes → Enable Kubernetes` **체크 해제** → RAM 약 500MB 절약
- `Preferences → Container Engine → General` → **containerd** 권장
  - K8s 표준 런타임, k3s와 이미지 스토어 공유, 미래 K8s 개발로 자연스럽게 이어짐

**1. VM 디스크(diffdisk) — 엔진 공통**

100GB 한도는 containerd/dockerd 무관하게 **VM 레벨 설정**이다. 엔진을 바꿔도 한도는 동일하다.  
Sparse file이라 실제 쓴 만큼만 SSD 점유 (factory reset 직후 ~400MB).

**2. data-root 설정 — 엔진별로 다름**

| | containerd | dockerd |
|---|---|---|
| 기본 저장 위치 | diffdisk 내 `/var/lib` | diffdisk 내 `/var/lib/docker` |
| Mac SSD 직접 저장 | **사실상 불가** (config.toml 재시작 시 덮어쓰임) | **GUI로 가능** — RD 자체 설정에 저장, 재시작 후 유지 |
| prune 후 공간 반환 | diffdisk sparse block 미반환 | Mac APFS가 즉시 반환 ✅ |
| 변경 필요 여부 | 불필요 (100GB 기본으로 충분) | 선택 사항 (diffdisk 밖으로 분리하고 싶을 때) |

> RD v1.22+에서 `/var/lib`은 tmpfs가 아닌 **100GB data-volume**에 마운트된다.  
> containerd는 기본값 그대로도 빌드 실패 없다. dockerd는 GUI로 Mac SSD 분리가 가능해 prune 효율이 높다.

**2. stateful 서비스는 반드시 bind mount 사용**  
PostgreSQL 등 DB 컨테이너는 named volume 대신 `./data` bind mount를 쓸 것.  
VM 재시작 후에도 데이터가 유지되며, **`rdctl factory-reset` 후에도 Mac SSD에 남는다.**

---

## 왜 macOS에서 Docker가 이렇게 동작하는가

### Docker는 macOS에서 네이티브로 실행 불가

Docker 컨테이너는 Linux 커널 기능에 의존한다:

- **Namespaces**: 프로세스/네트워크/파일시스템 격리
- **cgroups**: CPU/메모리 자원 제한

macOS는 Darwin(BSD 기반) 커널을 사용하기 때문에 이 기능들이 없다.  
따라서 Docker Desktop과 Rancher Desktop 모두 **내부적으로 경량 Linux VM을 띄워서** 그 안에서 Docker 데몬을 실행한다.

### Rancher Desktop의 VM 구조 (Apple VZ + Lima, v1.22+)

```
macOS Host (APFS/SSD)
└── Rancher Desktop
    └── Lima VM (Apple Virtualization Framework)
        ├── basedisk      → VM OS 베이스 이미지 (읽기 전용, ~264MB)
        ├── diffdisk      → 100GB sparse file (Mac SSD에 저장)
        │   └── data-volume (ext4) → VM 안에서 아래 경로로 마운트됨:
        │       ├── /var/lib   ← Docker/containerd 데이터 전체
        │       ├── /etc       ← 설정 파일
        │       ├── /home
        │       └── /tmp
        └── tmpfs → VM 루트(/) 전용 (OS 파일만, Docker 데이터 없음)
```

virtiofs로 Mac 홈 디렉터리도 VM에 공유됨:
```
/Users/j → virtiofs → VM 내부에서 /Users/j 로 접근 가능
```

**RD v1.22+ 핵심 구조**:
- `/var/lib`(Docker 데이터)이 **tmpfs가 아닌 data-volume**에 있음
- data-volume은 diffdisk(Mac SSD의 sparse file) 안 ext4 파티션
- 기본 100GB 할당 → 빌드 실패 용량 문제 없음
- **VM 재시작 후에도 Docker 데이터 유지됨**
- factory-reset 시에만 data-volume 초기화됨

---

## 문제 1: tmpfs 용량 부족 (빌드 실패)

> ⚠️ **RD v1.22+에서는 이 문제가 기본적으로 해결됨**  
> `/var/lib`이 100GB data-volume에 마운트되어 있어 tmpfs 용량과 무관하다.  
> 구버전 RD 또는 기존 설정이 잘못된 경우에만 해당.

### 증상

```
ERROR: no space left on device
```

### 원인 (구버전 RD 또는 잘못된 설정)

Docker의 `data-root`가 tmpfs(`/`) 위에 있을 경우:
- tmpfs 크기 = RAM / 2 (8GB RAM → 3.9GB)
- 이미지 레이어 + 빌드 캐시가 tmpfs를 채우면 빌드 실패

### 잘못된 대응

```bash
docker system prune -af
```

`docker system prune`은 논리적 공간만 정리하고  
**diffdisk sparse file의 블록은 macOS에 반환되지 않는다.**  
→ prune 해도 `du -sh diffdisk`는 줄어들지 않는다.

### 올바른 해결

- **RD v1.22+**: factory-reset 후 재시작하면 data-volume 구조로 자동 해결됨
- **dockerd 사용 시**: data-root를 `~/docker-data`로 변경하면 diffdisk 밖으로 분리 가능 (아래 참고)
- **containerd 사용 시**: data-root 변경 불필요 — data-volume이 이미 100GB

---

## 문제 2: Named Volume 데이터 유실

### RD v1.22+ 기준 named volume 영속성

| 상황 | named volume | bind mount (`./data`) |
|---|---|---|
| VM 재시작 (macOS 재부팅 포함) | **유지** ✅ | **유지** ✅ |
| Rancher Desktop 업데이트 | 대체로 유지 ✅ | **유지** ✅ |
| `rdctl factory-reset` | **소멸** ❌ | **유지** ✅ (Mac SSD에 남음) |

> RD v1.22+에서 named volume은 `/var/lib/docker/volumes/`에 저장되며,  
> 이 경로는 data-volume(diffdisk)에 마운트되어 **VM 재시작 후에도 유지된다.**  
> 단, factory-reset은 diffdisk 자체를 초기화하므로 소멸됨.

### 그래도 bind mount를 권장하는 이유

```yaml
# named volume — VM 재시작 OK, factory-reset 시 소멸
volumes:
  - pgdata:/var/lib/postgresql/data

# bind mount — VM 재시작 OK, factory-reset 후에도 Mac SSD에 유지 ✅
volumes:
  - ./data:/var/lib/postgresql/data
```

- factory-reset 시에도 데이터 보존
- Mac에서 직접 파일 접근/백업 가능
- 경로가 명확해 실수 위험 낮음

---

## 문제 3: diffdisk가 줄어들지 않음

### 증상

```bash
du -sh ~/Library/Application\ Support/rancher-desktop/lima/0/diffdisk
# 79G  ← docker prune 해도 안 줄어듦
```

### 원인

1. VM 내부 ext4 파일시스템에서 파일을 삭제해도 TRIM 명령이 macOS host에 전달되지 않음
2. macOS APFS의 sparse file은 hole-punching을 자동으로 하지 않음
3. 한번 기록된 블록은 `fstrim` 없이는 회수 불가

### 수동 회수 방법 (완전하지 않음)

```bash
rdctl shell sudo fstrim /
```

근본 해결은 diffdisk 삭제 (Factory Reset) 또는 data-root 이전이다.

---

## data-root 설정 — 엔진별 가이드

> **containerd 사용 시**: data-root 설정 불필요. 100GB data-volume 기본값으로 충분.  
> **dockerd 사용 시**: GUI로 `~/docker-data` 설정 가능. diffdisk 밖 Mac SSD에 직접 저장되어 prune 시 공간이 즉시 반환됨.

### 먼저 — 컨테이너 엔진 선택

> **권장: containerd**

| 항목 | containerd | dockerd (Moby) |
|---|---|---|
| CLI | `nerdctl`, `nerdctl compose` | `docker`, `docker compose` |
| compose 호환 | ✅ 일반 사용 문제없음 (엣지케이스 일부 차이) | ✅ 완전 호환 |
| K8s 표준 런타임 | ✅ K8s 1.24+ 공식 표준 | ❌ K8s 1.24에서 공식 제거 |
| k3s 이미지 공유 | ✅ 동일 스토어 — 빌드하면 바로 k3s에서 사용 | ❌ 별도 스토어 — 레지스트리 경유 필요 |
| data-root 설정 | `override.yaml` 한 번 작성 (자동 적용) | GUI에서 JSON 한 줄 |
| 리소스 | Lima VM이 병목 — 런타임 차이 미미 | Lima VM이 병목 — 런타임 차이 미미 |
| 권장 용도 | **일반 개발 + 미래 K8s 개발 모두** | 순수 docker compose만, K8s 계획 없을 때 |

**containerd를 권장하는 핵심 이유:**  
K8s 1.24부터 dockerd는 클러스터 런타임에서 공식 제거됐다. 나중에 K8s/k3s 개발로 이어질 때 별도 전환 없이 그대로 쓸 수 있다.  
data-root 설정도 `override.yaml` 한 번 작성하면 이후 자동 적용이라 초기 부담도 크지 않다.

> **리소스 절약 핵심은 엔진 선택이 아니라 Kubernetes 비활성화다.**  
> `Preferences → Kubernetes → Enable Kubernetes` 체크 해제만 해도 RAM 약 500MB 절약.

**Rancher Desktop 초기 설정:**
1. `Preferences → Kubernetes → Enable Kubernetes` **체크 해제**
2. `Preferences → Container Engine → General` → **containerd** 선택

현재 엔진 확인:
```bash
rdctl list-settings | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['containerEngine']['name'])"
```

- **containerd** → data-root 설정 불필요. 바로 shared network / 서비스 시작으로 이동
- **dockerd (moby)** → data-root 설정 원하면 아래 [dockerd 설정] 섹션 참고

---

### [dockerd] data-root 설정

**1. Mac에 data 폴더 생성**

```bash
mkdir -p ~/docker-data
```

**2. daemon.json 수정**

**방법 A — Rancher Desktop GUI (권장)**

`Rancher Desktop → Preferences → Container Engine → dockerd options` 에서 JSON 입력:

```json
{
  "data-root": "/Users/<username>/docker-data"
}
```

저장하면 자동으로 데몬 재시작. 3단계 생략 가능.

> 기존 설정(`min-api-version`, `features` 등)이 있으면 지우지 말고 `"data-root"` 키만 추가할 것.

**방법 B — rdctl shell**

```bash
rdctl shell sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "min-api-version": "1.41",
  "features": {
    "containerd-snapshotter": true
  },
  "data-root": "/Users/<username>/docker-data"
}
EOF
```

**3. 데몬 재시작 (방법 B 사용 시)**

```bash
rdctl shell sudo service docker restart
```

**4. 확인**

```bash
docker info | grep "Docker Root Dir"
# Docker Root Dir: /Users/<username>/docker-data
```

---

### [containerd] data-root 설정

> **설정 불필요 — 기본값으로 충분하다.**

RD v1.22+에서 containerd 데이터는 `/var/lib/nerdctl/`, `/var/lib/docker/` 등에 저장되며,  
이 경로는 100GB data-volume(diffdisk)에 마운트되어 있다.

**왜 변경이 어려운가:**
- `config.toml`은 Rancher Desktop이 재시작할 때마다 덮어씀 (GitHub #2708)
- Lima `override.yaml` provisioning script 방식은 YAML 파싱 이슈로 동작 불안정 (확인됨)
- K8s 비활성화 시 `/var/lib/rancher/k3s/` 경로도 무관함

**결론:** containerd data-root는 건드리지 말 것. 100GB 한도를 넘을 것 같으면 디스크 크기를 늘리거나 `nerdctl system prune`으로 정리할 것.

```bash
# 공간 정리
nerdctl system prune -af

# VM 디스크 한도 확장 (필요 시)
rdctl set --experimental.virtual-machine.disk-size=200GiB
```

---

### 주의사항

| 항목 | 내용 |
|---|---|
| 성능 | virtiofs I/O가 네이티브보다 느림. 빌드 시간 약 20-30% 증가 가능 |
| SSD 수명 | **걱정 불필요.** Apple Silicon SSD TBW는 256GB 모델 기준 ~300TB, 512GB↑는 600TB+. 헤비 빌드 기준 연간 소비량은 ~4TB 수준. 기본값(tmpfs) 사용 시에도 diffdisk가 결국 Mac SSD에 저장되므로 쓰기 자체는 어차피 SSD에 가고 있다. |
| Spotlight | `docker-data` 폴더를 Spotlight 인덱싱에서 제외 권장 |
| Time Machine | `docker-data` 폴더를 Time Machine 제외 목록에 추가 권장 |
| 경로 공유 | 두 데몬이 같은 data 경로를 공유하면 안 됨 |
| containerd 주의 | `config.toml`은 K3s가 재시작 시 덮어쓴다. 직접 편집하지 말고 반드시 `config.toml.tmpl` 또는 provisioning script를 사용할 것 |

**Spotlight 제외 방법**: 시스템 설정 → Siri 및 Spotlight → Spotlight 개인정보 → 폴더 추가

---

## VM 디스크 크기 (diffdisk)

### 기본값과 한도

| 항목 | 값 |
|---|---|
| 기본 할당 크기 | **100GiB** (Rancher Desktop 공식 기본값) |
| 파일 형식 | sparse file — 실제 쓴 만큼만 Mac SSD 차지 |
| 위치 | `~/Library/Application Support/rancher-desktop/lima/0/diffdisk` |
| 확장 | 가능 (명령어 1줄) |
| 축소 | **불가** |

> 100GiB는 SSD 용량과 무관한 고정 기본값이다. 2TB SSD여도 기본 100GiB로 시작한다.  
> Sparse file이므로 실제 사용량만큼만 점유 (factory reset 직후 ~400MB).

### 디스크 크기 확인

```bash
# 할당 크기 확인
rdctl list-settings | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['experimental']['virtualMachine']['diskSize'])"

# 실제 사용량 확인
du -sh ~/Library/Application\ Support/rancher-desktop/lima/0/diffdisk

# VM 내부에서 여유 공간 확인
rdctl shell df -h /var/lib
```

### 디스크 크기 확장

```bash
# GUI 불가 — rdctl 명령어로만 변경 가능
rdctl set --experimental.virtual-machine.disk-size=200GiB
# Rancher Desktop이 자동으로 재시작됨
```

> 줄이는 것은 불가능하므로 신중하게 설정할 것.  
> 기본 100GiB는 웬만한 개발 환경에 충분하다. 꽉 차기 전에 `nerdctl/docker system prune`을 주기적으로 실행할 것.

---

## PostgreSQL 및 상태 저장 서비스 설정

### 권장 compose 구성

```yaml
# shared-infra/postgres/docker-compose.yml

services:
  postgres:
    image: postgres:16-alpine
    container_name: shared_postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: shared_pg_su
      POSTGRES_PASSWORD: "password"
      POSTGRES_DB: postgres
    volumes:
      - ./data:/var/lib/postgresql/data   # bind mount (Mac 경로)
    ports:
      - "5432:5432"
    networks:
      - shared

# volumes: pgdata  ← named volume 사용 금지

networks:
  shared:
    external: true
```

`./data`는 `docker-compose.yml` 기준 상대 경로로, Mac 파일시스템에 저장된다.

### 초기 설정

```bash
cd shared-infra/postgres
mkdir -p data          # gitignore에 추가
docker compose up -d
```

`.gitignore`에 추가:
```
postgres/data/
```

---

## 신규 맥 초기 설정 절차 (Claude Code 자동화용)

> 이 섹션은 **Claude Code 등 AI 에이전트가 새 Mac에서 Docker 환경을 초기화**할 때 따르는 절차다.
> 아래 단계를 순서대로 실행하면 data-root virtiofs 설정 + stateful 서비스 bind mount 전환이 완료된다.

---

### 0단계 — 전제 조건 및 초기 설정

**Rancher Desktop 실행 확인:**
```bash
docker ps
```
`Cannot connect to the Docker daemon` 오류 → Rancher Desktop을 먼저 실행하고 재시도.

**Rancher Desktop 초기 설정 (새 맥이면 먼저 확인):**

1. `Preferences → Kubernetes → Enable Kubernetes` **체크 해제** — RAM 약 500MB 절약. docker compose 개발에 불필요
2. `Preferences → Container Engine → General` → **dockerd (moby)** 선택 권장

**컨테이너 엔진 확인:**
```bash
docker info | grep "Storage Driver"
```

결과에 따라 1단계 분기:
- **dockerd (moby)** → 1단계-A
- **containerd** → 1단계-B

---

### 1단계-A — [dockerd] data-root 이전

**1-1. Mac에 docker-data 폴더 생성**

```bash
mkdir -p ~/docker-data
```

**1-2. 기존 named volume 목록 확인 (있으면 데이터 백업 필요)**

```bash
docker volume ls
```

named volume에 중요 데이터가 있는 경우 bind mount 경로로 복사:

```bash
docker run --rm \
  -v pgdata:/source \
  -v /Users/<username>/shared-infra/postgres/data:/dest \
  alpine sh -c "cp -a /source/. /dest/"
```

**1-3. daemon.json에 data-root 추가**

**방법 A — Rancher Desktop GUI (권장)**

`Rancher Desktop → Preferences → Container Engine → dockerd options` 에서 `"data-root"` 키 추가:

```json
{
  "data-root": "/Users/<username>/docker-data"
}
```

저장하면 자동으로 데몬 재시작. 1-4 생략 가능.

**방법 B — rdctl shell**

```bash
rdctl shell sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "min-api-version": "1.41",
  "features": {
    "containerd-snapshotter": true
  },
  "data-root": "/Users/<username>/docker-data"
}
EOF
```

**1-4. 재시작 및 확인 (방법 B 사용 시)**

```bash
rdctl shell sudo service docker restart
```

```bash
docker info | grep "Docker Root Dir"
# 기대값: Docker Root Dir: /Users/<username>/docker-data
```

---

### 1단계-B — [containerd] data-root 이전

> containerd는 K3s가 내부 관리하며 `config.toml`이 재시작 시 덮어쓰인다.  
> provisioning script를 통해 부팅 시마다 설정이 적용되도록 해야 한다.

**1-1. Mac에 docker-data 폴더 생성**

```bash
mkdir -p ~/docker-data/containerd
```

**1-2. override.yaml에 provisioning script 작성**

`~/Library/Application Support/rancher-desktop/lima/_config/override.yaml` 파일을 열어 아래 내용 추가 (파일이 없으면 새로 생성):

```yaml
provision:
  - mode: system
    script: |
      #!/bin/sh
      mkdir -p /var/lib/rancher/k3s/agent/etc/containerd
      cat > /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl << 'EOF'
      {{ template "base" . }}

      root = "/Users/<username>/docker-data/containerd"
      EOF
```

> `<username>` → `whoami`로 확인 후 교체.  
> `{{ template "base" . }}`는 반드시 포함. 없으면 containerd가 동작하지 않는다.

**1-3. Rancher Desktop 재시작**

```bash
rdctl shutdown && rdctl start
```

**1-4. 확인**

```bash
rdctl shell sudo cat /var/lib/rancher/k3s/agent/etc/containerd/config.toml | grep root
# root = "/Users/<username>/docker-data/containerd"
```

---

### 2단계 — shared 네트워크 생성

```bash
docker network inspect shared 2>/dev/null || docker network create shared
```

---

### 3단계 — stateful 서비스 bind mount 확인 및 시작

`shared-infra/postgres/docker-compose.yml`을 열어 volumes 섹션 확인:

```bash
grep -A3 "volumes:" shared-infra/postgres/docker-compose.yml
```

`./data:/var/lib/postgresql/data` bind mount가 있으면 정상. named volume(`pgdata:` 등)이 있으면 아래 패턴으로 수정:

```yaml
# 잘못된 예 (named volume) → 이 패턴이 있으면 수정
volumes:
  - pgdata:/var/lib/postgresql/data
# 아래에 volumes: pgdata: 선언도 제거

# 올바른 예 (bind mount)
volumes:
  - ./data:/var/lib/postgresql/data
```

**postgres 시작**

```bash
cd shared-infra/postgres
mkdir -p data
docker compose up -d
docker logs shared_postgres --tail 10
```

---

### 4단계 — caddy 시작 (선택)

```bash
cd shared-infra/caddy
docker compose up -d
```

---

### 5단계 — Spotlight / Time Machine 제외 (수동)

터미널에서 직접 제외 명령은 없으므로 사용자에게 안내:

```
시스템 설정 → Siri 및 Spotlight → Spotlight 개인정보 → ~/docker-data 폴더 추가
시스템 설정 → 일반 → Time Machine → 제외 항목 → ~/docker-data 추가
```

---

### 진단 요약 — 현재 상태 점검용

새 맥 또는 기존 맥 상태를 한 번에 파악하려면 아래 명령을 순서대로 실행하고 출력을 확인:

```bash
# 1. Docker 실행 여부
docker ps

# 2. data-root 위치
docker info | grep "Docker Root Dir"

# 3. named volume 목록
docker volume ls

# 4. shared 네트워크 존재 여부
docker network inspect shared

# 5. postgres 컨테이너 상태
docker ps -a --filter name=shared_postgres

# 6. caddy 컨테이너 상태
docker ps -a --filter name=shared_caddy
```

---

## Rancher Desktop 설정 체크리스트

### 최초 설정 시 (순서대로)

- [ ] **컨테이너 엔진 확인** (`docker info | grep "Storage Driver"` 또는 Preferences → Container Engine)
- [ ] **data-root → `~/docker-data` 설정** ← 가장 먼저. 이게 없으면 빌드가 터진다
  - dockerd: GUI(`dockerd options`) 또는 `/etc/docker/daemon.json`
  - containerd: `override.yaml` provisioning script로 `config.toml.tmpl` 생성
- [ ] `~/docker-data` → Spotlight/Time Machine 제외
- [ ] shared network 생성: `docker network create shared`
- [ ] 모든 stateful 서비스 bind mount로 전환 (named volume 사용 금지)

### 재발 방지 체크

- [ ] 주기적으로 prune 실행 (data-root가 `~/docker-data`이면 Mac SSD 공간 즉시 반환됨)
  - dockerd: `docker system prune -af`
  - containerd: `nerdctl system prune -af`
- [ ] named volume 사용하지 않기 (compose에서 `volumes:` 최상위 선언 금지)
- [ ] Rancher Desktop 설정 변경 전 실행 중인 DB 컨테이너 데이터 확인
- [ ] containerd 사용 시: `config.toml` 직접 편집 금지 (재시작 시 덮어쓰임)

### VM 재시작이 발생하는 상황

- macOS 재부팅
- Rancher Desktop CPU/RAM 설정 변경
- Rancher Desktop 업데이트
- 비정상 종료

---

## 현재 확인된 설정 및 결론 (2026-04-16 기준)

### 확인된 환경

| 항목 | 값 |
|---|---|
| Rancher Desktop | v1.22.0 |
| 컨테이너 엔진 | **containerd** |
| Kubernetes | **비활성화** |
| VM type | vz (Apple Virtualization Framework) |
| Mount type | virtiofs |
| diffdisk 할당 | 100GiB (sparse) |
| diffdisk 실제 점유 | ~393MB (factory reset 직후) |

### VM 마운트 구조 (직접 확인)

```
/var/lib   → data-volume (diffdisk ext4, 97.9GB)  ← Docker 데이터 전체
/etc       → data-volume
/home      → data-volume
/Users/j   → virtiofs (Mac SSD 직접 공유)
tmpfs      → VM 루트(/) 전용, Docker 데이터 없음
```

### 결론 — 현재 설정으로 충분한 것들

| 항목 | 결론 |
|---|---|
| data-root 변경 | **불필요** — `/var/lib`이 이미 100GB data-volume에 있음 |
| named volume 사용 | **가능** — VM 재시작 후 유지됨 |
| named volume vs bind mount | **bind mount 권장** — factory-reset 후에도 Mac SSD에 유지 |
| 빌드 실패 (no space) | **문제없음** — 100GB 여유 |
| diffdisk 누적 | `nerdctl system prune`으로 정기 정리 |
| 디스크 한도 도달 시 | `rdctl set --experimental.virtual-machine.disk-size=200GiB` |

### 해야 할 것 (최초 1회)

```bash
# 1. shared 네트워크 생성
docker network create shared

# 2. postgres 시작
cd shared-infra/postgres && docker compose up -d

# 3. caddy 시작
cd shared-infra/caddy && docker compose up -d
```

---

---

## 환경별 Docker 스토리지 전략

### 한눈에 보기

| 환경 | 문제 존재 여부 | Named Volume | DB 스토리지 권장 |
|---|---|---|---|
| macOS 개발환경 (Rancher Desktop) | ✅ 있음 | ❌ 사용 금지 | bind mount |
| Linux 개발환경 (직접 설치) | ❌ 없음 | ✅ 사용 가능 | named volume 또는 bind mount |
| 클라우드 단일 서버 (EC2, GCP VM) | ❌ 없음 | ✅ 사용 가능 | bind mount + EBS 별도 마운트 |
| 클라우드 멀티 서버 / Kubernetes | ❌ 없음 | ❌ 노드 간 공유 불가 | PersistentVolume (EBS, EFS 등) |
| 클라우드 프로덕션 DB | ❌ 없음 | 해당 없음 | 매니지드 서비스 (RDS, Cloud SQL) 권장 |

---

### 개발 환경 (macOS + Rancher Desktop)

이 문서의 모든 문제(tmpfs 휘발, diffdisk 누적, named volume 소멸)는  
**macOS + Lima VM 조합에서만 발생하는 구조적 한계**다.

```
macOS Darwin 커널
  → Linux VM 필요 (Docker 네이티브 불가)
    → Lima VZ가 tmpfs를 루트로 사용
      → Docker 데이터가 메모리에 올라감
        → 재시작 시 소멸, 용량 한계 발생
```

**규칙:**
- data-root → virtiofs Mac 경로 필수
- 모든 stateful 컨테이너 → bind mount 필수
- named volume → 사용 금지

---

### Linux 개발 환경 (직접 설치)

Linux에서는 Docker가 커널에 네이티브로 실행된다. VM이 없고 tmpfs 문제도 없다.

```
Linux 커널
  → Docker 데몬 직접 실행
    → /var/lib/docker → 실제 SSD에 저장
      → 재시작해도 유지, docker prune 하면 공간 즉시 회수
```

**규칙:**
- named volume, bind mount 둘 다 사용 가능
- DB는 named volume도 안전하지만 bind mount가 백업/이관에 편리
- data-root 변경 불필요

---

### 클라우드 단일 서버 (EC2, GCP VM, DigitalOcean 등)

클라우드 VM도 Linux이므로 macOS 문제 없음.  
단, 서버 자체가 교체/삭제될 수 있으므로 데이터 볼륨을 분리하는 것이 중요하다.

```
EC2 Instance (Linux)
  ├── Root Volume (OS + Docker) → 인스턴스 삭제 시 함께 삭제
  └── Data Volume (EBS) → 인스턴스와 독립적으로 유지
```

**권장 구성:**

```bash
# EBS 볼륨을 /data 에 마운트 (fstab 등록으로 재시작 후에도 유지)
/dev/xvdb   /data   ext4   defaults,nofail   0   2
```

```yaml
# compose에서 EBS 마운트 경로 사용
volumes:
  - /data/postgres:/var/lib/postgresql/data
```

**규칙:**
- named volume: 사용 가능하지만 인스턴스 종료 시 날아갈 수 있음
- **중요 데이터는 EBS(별도 볼륨)에 bind mount**
- `docker system prune` 하면 공간 즉시 회수

---

### 클라우드 멀티 서버 / Kubernetes

컨테이너가 여러 노드에 스케줄링되므로 named volume(노드 로컬)은 사용 불가.  
Kubernetes의 PersistentVolume으로 관리한다.

```
Kubernetes Cluster
  ├── Node A  ─┐
  ├── Node B   ├── Pod (컨테이너) → PVC → PV (EBS, EFS, NFS...)
  └── Node C  ─┘
```

**스토리지 클래스 선택:**

| 유형 | 특징 | 적합한 용도 |
|---|---|---|
| EBS (AWS) | 단일 노드 연결, 고성능 | DB, 단일 파드 |
| EFS (AWS) | 멀티 노드 공유, NFS 기반 | 파일 공유, 로그 |
| S3 (오브젝트) | 무한 확장, API 접근 | 이미지, 백업, 정적 파일 |

**규칙:**
- named volume 사용 불가
- PVC(PersistentVolumeClaim) + StorageClass로 선언
- DB는 가급적 매니지드 서비스(RDS 등)로 분리

---

### 프로덕션 DB 권장 방식

컨테이너 안에서 DB를 직접 운영하는 것은 **개발/스테이징 환경에 한정**하는 것이 좋다.

| 환경 | DB 운영 방식 |
|---|---|
| 로컬 개발 | Docker compose (bind mount) |
| 스테이징 | Docker compose (bind mount + EBS) |
| 프로덕션 | **매니지드 서비스 권장** (AWS RDS, GCP Cloud SQL, Supabase 등) |

**매니지드 서비스를 쓰는 이유:**
- 자동 백업 / 복구 포인트
- 자동 패치 및 버전 업그레이드
- 읽기 복제본(Read Replica) 간단 추가
- 장애 시 자동 페일오버
- 스토리지 자동 확장

---

### 환경별 최종 요약

```
로컬 macOS (RD v1.22+)  → containerd: diffdisk 100GB 기본값으로 충분. DB는 bind mount 권장.
                           dockerd: GUI로 data-root ~/docker-data 설정 → Mac SSD 직접 저장, prune 즉시 반환.
로컬 Linux              → named volume 또는 bind mount 자유롭게 사용 가능
클라우드 단일 VM         → bind mount + 별도 데이터 볼륨(EBS 등) 분리
Kubernetes              → PersistentVolumeClaim + StorageClass
프로덕션 DB             → RDS / Cloud SQL 등 매니지드 서비스
```

---

## 참고 링크

- [Lima VZ Internals](https://lima-vm.io/docs/dev/internals/)
- [Rancher Desktop Disk Issues #1942](https://github.com/rancher-sandbox/rancher-desktop/issues/1942)
- [Lima Docker Volumes Volatility Discussion](https://github.com/lima-vm/lima/discussions/3106)
- [Docker daemon configuration reference](https://docs.docker.com/engine/daemon/)
- [Kubernetes PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [AWS EBS with Kubernetes](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)
