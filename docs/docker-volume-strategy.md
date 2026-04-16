# Docker 팁 — macOS 로컬 운영 & 초기 설정 가이드

> 작성일: 2026-04-16  
> 환경: macOS + Rancher Desktop v1.22 + Apple VZ (vz) + virtiofs

---

## TL;DR — 결론부터

macOS에서 Rancher Desktop을 쓰면 두 가지를 반드시 설정해야 한다.

**1. data-root를 `~/docker-data`로 전역 설정 (필수)**  
기본값(`/var/lib/docker`)은 VM의 tmpfs 위에 있어 8GB RAM 기준 3.9GB밖에 안 된다.  
이미지/빌드 캐시가 쌓이면 `no space left on device`로 빌드가 터진다.  
`~/docker-data`로 옮기면 Mac SSD에 저장되어 용량 문제가 사라지고, `docker system prune` 시 공간도 즉시 반환된다.

**2. stateful 서비스는 반드시 bind mount 사용**  
PostgreSQL 등 DB 컨테이너는 named volume 대신 `./data` bind mount를 써야 VM 재시작 후에도 데이터가 유지된다.

---

## 왜 macOS에서 Docker가 이렇게 동작하는가

### Docker는 macOS에서 네이티브로 실행 불가

Docker 컨테이너는 Linux 커널 기능에 의존한다:

- **Namespaces**: 프로세스/네트워크/파일시스템 격리
- **cgroups**: CPU/메모리 자원 제한

macOS는 Darwin(BSD 기반) 커널을 사용하기 때문에 이 기능들이 없다.  
따라서 Docker Desktop과 Rancher Desktop 모두 **내부적으로 경량 Linux VM을 띄워서** 그 안에서 Docker 데몬을 실행한다.

### Rancher Desktop의 VM 구조 (Apple VZ + Lima)

```
macOS Host (APFS)
└── Rancher Desktop
    └── Lima VM (Apple Virtualization Framework)
        ├── basedisk   → VM OS 베이스 이미지 (읽기 전용, ~264MB)
        ├── diffdisk   → VM 상태 변경분 스냅샷 (sparse file, 100GB 할당)
        └── RAM에 tmpfs로 올라간 루트 파일시스템 (3.9GB = RAM/2)
```

**핵심 문제**: VM의 루트(`/`)가 **tmpfs (메모리 기반 파일시스템)** 이다.

- VM 시작 시: diffdisk 상태를 tmpfs로 로드
- VM 실행 중: 모든 Docker 데이터(`/var/lib/docker`)가 tmpfs에 저장됨
- VM 재시작: tmpfs 초기화 → **Docker 데이터 소멸**

---

## 문제 1: tmpfs 용량 부족 (빌드 실패)

### 증상

```
ERROR: no space left on device
```

- tmpfs 크기 = RAM / 2 (8GB RAM → 3.9GB tmpfs)
- Docker 이미지 레이어 + 빌드 캐시가 tmpfs를 채우면 빌드 실패

### 원인

Docker의 `data-root`가 기본값 `/var/lib/docker` (tmpfs 위)에 있어서  
이미지를 많이 pull/build할수록 tmpfs가 가득 찬다.

### 잘못된 대응

```bash
docker system prune -af  # tmpfs 안의 논리적 공간만 비워줌
```

`docker system prune`은 tmpfs 내부는 정리하지만,  
**diffdisk sparse file의 블록은 macOS에 반환되지 않는다.**  
→ prune 해도 macOS에서 `du -sh diffdisk`는 줄어들지 않는다.

### 올바른 해결: data-root를 virtiofs Mac 경로로 변경 (아래 참고)

---

## 문제 2: Named Volume 휘발 (DB 데이터 유실)

### 증상

- VM 재시작(macOS 리부트, Rancher Desktop 설정 변경 등) 후 `docker volume ls`가 비어있음
- PostgreSQL 컨테이너가 빈 DB로 시작됨

### 원인

Docker named volume은 `/var/lib/docker/volumes/`에 저장된다.  
이 경로가 tmpfs 위에 있으므로 **VM이 재시작되면 사라진다.**

```yaml
# 위험한 방식 (named volume)
volumes:
  - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:  # ← /var/lib/docker/volumes/pgdata 에 저장 → tmpfs → 재시작 시 소멸
```

### 올바른 해결: bind mount로 Mac 경로 직접 사용

```yaml
# 안전한 방식 (bind mount)
volumes:
  - ./data:/var/lib/postgresql/data  # Mac 파일시스템에 직접 저장
```

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

## 근본 해결: data-root를 virtiofs Mac 경로로 설정

> **Rancher Desktop을 쓴다면 이 설정은 선택이 아니라 필수다.**  
> 기본값을 그대로 두면 tmpfs가 꽉 차 빌드가 터지고, `docker system prune`을 해도 공간이 반환되지 않는다.  
> 새 맥 세팅 시 가장 먼저 해야 할 전역 설정이다.

virtiofs는 macOS 파일시스템을 VM 내부에 직접 마운트하는 방식이다.  
Docker의 `data-root`를 virtiofs로 연결된 Mac 경로(`~/docker-data`)로 바꾸면:

- Docker 이미지/레이어/빌드 캐시가 Mac SSD에 저장 → tmpfs 용량 문제 없음
- `docker system prune` 시 APFS가 공간 즉시 반환 → diffdisk 누적 문제 해소
- VM 재시작해도 데이터 유지
- Mac에서 직접 접근/백업 가능

> **경로 선택**: `~/docker-data`를 권장한다. data-root는 머신 전역 설정으로 모든 프로젝트의 이미지/레이어가 쌓이는 곳이므로, 특정 프로젝트 디렉터리 안에 두어서는 안 된다.

### 설정 방법

**1. Mac에 Docker data 폴더 생성**

```bash
mkdir -p ~/docker-data
```

**2. Docker daemon.json 수정**

**방법 A — Rancher Desktop GUI (권장)**

`Rancher Desktop → Preferences → Container Engine → dockerd options` 에서 JSON 직접 입력:

```json
{
  "data-root": "/Users/<username>/docker-data"
}
```

저장하면 Rancher Desktop이 자동으로 데몬을 재시작한다. 3단계 생략 가능.

> 기존 설정(`min-api-version`, `features` 등)이 텍스트박스에 있으면 지우지 말고 `"data-root"` 키만 추가할 것.

**방법 B — rdctl shell (CLI)**

```bash
rdctl shell sudo vi /etc/docker/daemon.json
```

```json
{
  "min-api-version": "1.41",
  "features": {
    "containerd-snapshotter": true
  },
  "data-root": "/Users/<username>/docker-data"
}
```

> `/Users/<username>`은 `mount0`으로 virtiofs를 통해 VM에 마운트되어 있어 접근 가능하다.

**3. Docker 데몬 재시작 (방법 B 사용 시)**

```bash
rdctl shell sudo service docker restart
# 또는 Rancher Desktop 재시작
```

**4. 정상 동작 확인**

```bash
docker info | grep "Docker Root Dir"
# Docker Root Dir: /Users/<username>/docker-data
```

### 주의사항

| 항목 | 내용 |
|---|---|
| 성능 | virtiofs I/O가 네이티브보다 느림. 빌드 시간 약 20-30% 증가 가능 |
| Spotlight | `docker-data` 폴더를 Spotlight 인덱싱에서 제외 권장 |
| Time Machine | `docker-data` 폴더를 Time Machine 제외 목록에 추가 권장 |
| 경로 공유 | 두 Docker 데몬이 같은 data-root를 공유하면 안 됨 |

**Spotlight 제외 방법**: 시스템 설정 → Siri 및 Spotlight → Spotlight 개인정보 → 폴더 추가

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

### 0단계 — 전제 조건 확인

```bash
# Rancher Desktop이 실행 중인지 확인
docker ps
```

`Cannot connect to the Docker daemon` 오류 → Rancher Desktop을 먼저 실행하고 재시도.

```bash
# 현재 data-root 위치 확인
docker info | grep "Docker Root Dir"
```

- `Docker Root Dir: /var/lib/docker` → 설정 필요 (아래 1단계 진행)
- `Docker Root Dir: /Users/.../docker-data` → 이미 설정됨 (2단계로 이동)

---

### 1단계 — data-root를 virtiofs Mac 경로로 이전

**1-1. Mac에 docker-data 폴더 생성**

```bash
mkdir -p ~/docker-data
```

**1-2. 기존 named volume 목록 확인 (있으면 데이터 백업 필요)**

```bash
docker volume ls
```

named volume에 중요 데이터가 있는 경우 컨테이너를 먼저 내리고 데이터를 bind mount 경로로 복사한다:

```bash
# 예시: pgdata named volume → ./data 로 복사
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

저장하면 자동으로 데몬이 재시작된다. 1-4 생략 가능.

> 기존 설정이 있으면 지우지 말고 `"data-root"` 키만 추가할 것.

**방법 B — rdctl shell (CLI)**

현재 내용 확인 후:

```bash
rdctl shell sudo cat /etc/docker/daemon.json
```

기존 내용을 유지하면서 `"data-root"` 키만 추가한다:

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

> `<username>`을 실제 macOS 사용자명으로 교체. `whoami` 명령으로 확인 가능.

**1-4. Docker 데몬 재시작 (방법 B 사용 시)**

```bash
rdctl shell sudo service docker restart
```

재시작 후 20~30초 대기 후 확인:

```bash
docker info | grep "Docker Root Dir"
# 기대값: Docker Root Dir: /Users/<username>/docker-data
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

- [ ] **data-root → `~/docker-data` 설정** ← 가장 먼저. 이게 없으면 빌드가 터진다
- [ ] `~/docker-data` → Spotlight/Time Machine 제외
- [ ] shared network 생성: `docker network create shared`
- [ ] 모든 stateful 서비스 bind mount로 전환 (named volume 사용 금지)

### 재발 방지 체크

- [ ] `docker system prune -af` 주기적으로 실행 (data-root가 `~/docker-data`이면 Mac SSD 공간 즉시 반환됨)
- [ ] named volume 사용하지 않기 (compose에서 `volumes:` 최상위 선언 금지)
- [ ] Rancher Desktop 설정 변경 전 실행 중인 DB 컨테이너 데이터 확인

### VM 재시작이 발생하는 상황

- macOS 재부팅
- Rancher Desktop CPU/RAM 설정 변경
- Rancher Desktop 업데이트
- 비정상 종료

---

## 환경 정보 (2026-04-16 기준)

| 항목 | 값 |
|---|---|
| Rancher Desktop | v1.22.0 |
| VM type | vz (Apple Virtualization Framework) |
| Mount type | virtiofs |
| CPU | 2코어 |
| RAM | 8GB |
| tmpfs root size | 3.9GB (RAM/2) |
| diffdisk 할당 | 100GB (sparse) |
| diffdisk 실제 점유 | ~79GB (누적) |
| Mac 여유 공간 | ~588GB |

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
로컬 macOS       → data-root virtiofs + bind mount 필수 (named volume 사용 금지)
로컬 Linux       → named volume 또는 bind mount 자유롭게 사용 가능
클라우드 단일 VM  → bind mount + 별도 데이터 볼륨(EBS 등) 분리
Kubernetes       → PersistentVolumeClaim + StorageClass
프로덕션 DB      → RDS / Cloud SQL 등 매니지드 서비스
```

---

## 참고 링크

- [Lima VZ Internals](https://lima-vm.io/docs/dev/internals/)
- [Rancher Desktop Disk Issues #1942](https://github.com/rancher-sandbox/rancher-desktop/issues/1942)
- [Lima Docker Volumes Volatility Discussion](https://github.com/lima-vm/lima/discussions/3106)
- [Docker daemon configuration reference](https://docs.docker.com/engine/daemon/)
- [Kubernetes PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [AWS EBS with Kubernetes](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)
