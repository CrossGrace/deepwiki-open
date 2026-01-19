# DeepWiki-Open Docker 설정 가이드

DeepWiki-Open을 Docker로 실행하기 위한 완벽 가이드입니다.

## 목차

- [빠른 시작](#빠른-시작)
- [두 가지 배포 옵션](#두-가지-배포-옵션)
- [사전 요구사항](#사전-요구사항)
- [환경 설정](#환경-설정)
- [옵션 1: 전체 웹 애플리케이션](#옵션-1-전체-웹-애플리케이션)
- [옵션 2: CLI 전용](#옵션-2-cli-전용)
- [환경 변수](#환경-변수)
- [문제 해결](#문제-해결)

---

## 빠른 시작

### 1. 저장소 클론

```bash
git clone https://github.com/CrossGrace/deepwiki-open.git
cd deepwiki-open
```

### 2. 환경 설정

```bash
# 예제 환경 파일 복사
cp .env.example .env

# .env 파일을 편집하여 API 자격 증명 입력
nano .env  # 또는 선호하는 편집기 사용
```

### 3. 배포 옵션 선택

**전체 웹 애플리케이션:**
```bash
docker-compose up -d
```

**CLI 전용:**
```bash
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/your-org/your-repo \
  --output /app/output
```

---

## 두 가지 배포 옵션

### 옵션 1: 전체 웹 애플리케이션

- **포함 내용**: Next.js 프론트엔드 + FastAPI 백엔드 + CLI 도구
- **사용 사례**: 위키 생성을 위한 대화형 웹 인터페이스
- **리소스 요구사항**: ~4-6 GB RAM, 2개 이상의 CPU 코어
- **포트**: 3000 (프론트엔드), 8001 (백엔드 API)
- **Docker 파일**: `Dockerfile`, `docker-compose.yml`

### 옵션 2: CLI 전용

- **포함 내용**: 명령줄 도구만 (`deepwiki_single.py`)
- **사용 사례**: 자동화된 위키 생성, CI/CD 파이프라인, 예약 작업
- **리소스 요구사항**: ~1-2 GB RAM, 1개 이상의 CPU 코어
- **포트**: 없음 (CLI만)
- **Docker 파일**: `Dockerfile.cli`, `docker-compose.cli.yml`

---

## 사전 요구사항

### 필수 소프트웨어

- **Docker**: 버전 20.10 이상
- **Docker Compose**: 버전 1.29 이상

### 필수 API 접근

다음에 대한 접근이 필요합니다:

1. **GPT-OSS-130b LLM API** (OpenAI 호환 엔드포인트)
   - Base URL
   - 인증 토큰

2. **BGE-M3 Embedding API** (1024차원 임베딩)
   - Base URL
   - 인증 토큰

### 선택 사항

- **GitHub Personal Access Token** (비공개 저장소용)
  - 생성 위치: https://github.com/settings/tokens
  - 필요한 권한: `repo` (비공개 저장소 전체 접근)

---

## 환경 설정

### 1단계: 환경 파일 생성

```bash
cp .env.example .env
```

### 2단계: 환경 파일 편집

`.env` 파일을 텍스트 편집기로 열고 필수 변수를 설정합니다:

```bash
# 필수: LLM API
DEEPWIKI_LLM_BASE_URL=https://your-llm-api.company.com
DEEPWIKI_LLM_TOKEN=your-llm-auth-token

# 필수: Embedding API
DEEPWIKI_EMBEDDING_BASE_URL=https://your-embedding-api.company.com
DEEPWIKI_EMBEDDING_TOKEN=your-embedding-auth-token

# 선택: 비공개 저장소용 GitHub 토큰
GITHUB_TOKEN=ghp_your_github_token_here
```

---

## 옵션 1: 전체 웹 애플리케이션

### 빌드 및 시작

```bash
# 빌드하고 백그라운드에서 시작
docker-compose up -d

# 또는 먼저 빌드한 다음 시작
docker-compose build
docker-compose up -d
```

### 애플리케이션 접속

- **프론트엔드**: http://localhost:3000
- **백엔드 API**: http://localhost:8001
- **API 문서**: http://localhost:8001/docs

### 로그 확인

```bash
# 모든 로그 확인
docker-compose logs -f

# 특정 서비스 로그 확인
docker-compose logs -f deepwiki
```

### 중지 및 제거

```bash
# 컨테이너 중지
docker-compose stop

# 컨테이너 중지 및 제거
docker-compose down

# 컨테이너, 볼륨까지 모두 제거
docker-compose down -v
```

---

## 옵션 2: CLI 전용

### CLI 이미지 빌드

```bash
docker-compose -f docker-compose.cli.yml build
```

### CLI 명령 실행

**기본 사용법:**

```bash
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/anthropics/anthropic-sdk-python \
  --output /app/output
```

**비공개 저장소용 GitHub 토큰 사용:**

```bash
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/your-org/private-repo \
  --token $GITHUB_TOKEN \
  --output /app/output
```

**드라이 런 (테스트, 파일 쓰지 않음):**

```bash
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/your-org/your-repo \
  --dry-run \
  --debug
```

### 생성된 출력 확인

생성된 위키 파일은 `./output` 디렉토리에 있습니다:

```bash
ls -la ./output/
cat ./output/Home.md
cat ./output/_Sidebar.md
```

---

## 환경 변수

### 필수 변수

| 변수 | 설명 | 예시 |
|------|------|------|
| `DEEPWIKI_LLM_BASE_URL` | GPT-OSS-130b API의 Base URL | `https://llm-api.company.com` |
| `DEEPWIKI_LLM_TOKEN` | LLM API 인증 토큰 | `your-secret-token` |
| `DEEPWIKI_EMBEDDING_BASE_URL` | BGE-M3 API의 Base URL | `https://embed-api.company.com` |
| `DEEPWIKI_EMBEDDING_TOKEN` | Embedding API 인증 토큰 | `your-secret-token` |

### 선택 변수 (애플리케이션)

| 변수 | 설명 | 기본값 |
|------|------|--------|
| `PORT` | 백엔드 API 포트 | `8001` |
| `NODE_ENV` | Node 환경 | `production` |
| `LOG_LEVEL` | 로깅 레벨 | `INFO` |
| `DEEPWIKI_WORKSPACE` | 저장소 클론용 작업 공간 | `./workspace` |
| `DEEPWIKI_OUTPUT` | 기본 출력 디렉토리 | `./wiki_output` |
| `GITHUB_TOKEN` | GitHub 액세스 토큰 | (비어있음) |

---

## 문제 해결

### 문제: 컨테이너가 시작되지 않음

**로그 확인:**

```bash
docker-compose logs deepwiki
```

**일반적인 원인:**
1. `.env` 파일에 환경 변수 누락
2. 잘못된 API 자격 증명
3. 포트 충돌 (3000 또는 8001이 이미 사용 중)

**해결 방법:**

```bash
# .env 파일이 존재하고 필수 변수가 있는지 확인
cat .env

# 포트 충돌 확인
sudo lsof -i :3000
sudo lsof -i :8001

# 다른 포트 사용
PORT=8002 docker-compose up
```

### 문제: API 인증 실패

**에러 메시지:** `HTTP 401 Unauthorized`

**해결 방법:**

1. `.env`의 API 토큰이 올바른지 확인
2. 토큰이 만료되지 않았는지 확인
3. API 연결 테스트:

```bash
# LLM API 테스트
curl -H "x-dep-ticket: your-token" \
     https://your-llm-api.com/health

# Embedding API 테스트
curl -H "x-dep-ticket: your-token" \
     https://your-embed-api.com/health
```

### 문제: 메모리 부족 오류

**에러 메시지:** `Container killed due to memory limit`

**해결 방법:**

`.env`에서 메모리 제한 증가:

```bash
DOCKER_MEMORY_LIMIT=8g
DOCKER_MEMORY_RESERVATION=4g
```

새 제한으로 재시작:

```bash
docker-compose down
docker-compose up -d
```

---

## 사용 예시

### 예시 1: 공개 저장소 위키 생성

```bash
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/anthropics/anthropic-sdk-python \
  --output /app/output
```

### 예시 2: 비공개 저장소 위키 생성

```bash
docker-compose -f docker-compose.cli.yml run --rm \
  -e GITHUB_TOKEN=ghp_your_token_here \
  deepwiki-cli \
  --repo https://github.com/your-company/private-repo \
  --token $GITHUB_TOKEN \
  --output /app/output
```

### 예시 3: 디버그 로깅을 사용한 드라이 런

```bash
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/your-org/your-repo \
  --dry-run \
  --debug
```

---

## 자주 사용하는 명령어

```bash
# 전체 웹 애플리케이션
docker-compose up -d                    # 시작
docker-compose logs -f                  # 로그 확인
docker-compose stop                     # 중지
docker-compose down                     # 중지 및 제거
docker-compose down -v                  # 중지, 제거, 볼륨 삭제

# CLI 전용
docker-compose -f docker-compose.cli.yml build                            # 빌드
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli --help     # 도움말
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli [옵션]     # 실행

# 유지보수
docker-compose build --no-cache         # 재빌드
docker-compose exec deepwiki bash       # 셸 접속
docker system prune -a                  # 정리
```

---

## 파일 개요

```
deepwiki-open/
├── .env.example              # 환경 설정 예제
├── .env                      # 사용자 환경 설정 (생성 필요)
├── Dockerfile                # 전체 웹 애플리케이션 이미지
├── Dockerfile.cli            # CLI 전용 경량 이미지
├── docker-compose.yml        # 전체 웹 애플리케이션 compose
├── docker-compose.cli.yml    # CLI 전용 compose
├── DOCKER_SETUP.md           # 영문 가이드
├── DOCKER_SETUP.kr.md        # 한글 가이드 (이 파일)
└── output/                   # 생성된 위키 파일 (마운트된 볼륨)
```

---

## 지원

문제나 질문이 있는 경우:

1. 이 문서를 철저히 확인
2. 로그에서 오류 메시지 확인: `docker-compose logs -f`
3. 디버그 모드 활성화: `--debug` 플래그 또는 `LOG_LEVEL=DEBUG`
4. 기존 이슈 확인: https://github.com/CrossGrace/deepwiki-open/issues
5. 다음 정보와 함께 새 이슈 생성:
   - Docker 버전: `docker --version`
   - Docker Compose 버전: `docker-compose --version`
   - 오류 로그
   - 재현 단계

---

**최종 업데이트**: 2026년 1월

**버전**: 1.0
