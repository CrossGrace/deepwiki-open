# 🚀 DeepWiki 상세 문서 생성 가이드

> **GPT-OSS 120B/130B를 사용한 아주 자세한 위키 문서 생성 완벽 가이드**

## 📋 목차

1. [개요](#개요)
2. [환경 설정](#환경-설정)
3. [상세 문서 생성 설정](#상세-문서-생성-설정)
4. [실행 방법](#실행-방법)
5. [생성된 문서 배포](#생성된-문서-배포)
6. [트러블슈팅](#트러블슈팅)

---

## 🎯 개요

이 가이드는 **GPT-OSS 120B/130B** 모델을 사용하여 코드베이스에서 **아주 자세하고 상세한** 위키 문서를 생성하는 방법을 설명합니다.

### 최적화된 설정 (2024년 12월 업데이트)

이 프로젝트는 **상세한 문서 생성**을 위해 다음과 같이 최적화되었습니다:

| 파라미터 | 기본값 | 상세 문서용 설정 | 효과 |
|---------|-------|----------------|------|
| `chunk_size` | 500 단어 | **1000 단어** | 더 큰 코드 블록 분석 |
| `overlap` | 100 단어 | **200 단어** | 더 나은 컨텍스트 연결 |
| `max_context_tokens` | 6000 토큰 | **10000 토큰** | 페이지당 더 많은 정보 |
| `top_k` (RAG) | 10개 | **20개** | 더 많은 관련 코드 검색 |
| `matched_files` | 10개 | **20개** | 더 많은 파일 참조 |
| `file_content_limit` | 2000자 | **4000자** | 파일당 더 긴 내용 |
| `chunk_text_limit` | 1000자 | **2000자** | 청크당 더 긴 내용 |

**결과**: 기본 설정 대비 **약 2.5배 더 자세한** 문서 생성!

---

## ⚙️ 환경 설정

### 1단계: 환경 변수 파일 생성

`.env` 파일을 생성하고 GPT-OSS API 정보를 입력합니다:

```bash
# .env 파일 생성
cp .env.example .env
```

`.env` 파일 내용:

```bash
# =============================================================================
# REQUIRED: LLM API Configuration (gpt-oss-120b or gpt-oss-130b)
# =============================================================================
DEEPWIKI_LLM_BASE_URL=https://your-internal-llm-api.company.com
DEEPWIKI_LLM_TOKEN=your-llm-authentication-token

# =============================================================================
# REQUIRED: Embedding API Configuration (BGE-M3)
# =============================================================================
DEEPWIKI_EMBEDDING_BASE_URL=https://your-internal-embedding-api.company.com
DEEPWIKI_EMBEDDING_TOKEN=your-embedding-authentication-token

# =============================================================================
# OPTIONAL: Embedding API Rate Limit 설정
# =============================================================================
# 배치 크기 (기본값: 50, Rate Limit 발생 시 30 또는 20으로 줄이기)
DEEPWIKI_EMBEDDING_BATCH_SIZE=50

# 배치 간 대기 시간 (기본값: 1.0초, Rate Limit 발생 시 2.0 또는 3.0으로 늘리기)
DEEPWIKI_EMBEDDING_BATCH_DELAY=1.0

# =============================================================================
# OPTIONAL: 출력 디렉토리 설정
# =============================================================================
DEEPWIKI_WORKSPACE=./workspace
DEEPWIKI_OUTPUT=./wiki_output
```

### 2단계: 의존성 설치

```bash
# Python 가상환경 생성 (선택사항)
python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 필요한 패키지 설치
pip install -r requirements_single.txt
```

### 3단계: API 연결 테스트

```bash
# 환경 변수 확인
python -c "import os; print('LLM URL:', os.getenv('DEEPWIKI_LLM_BASE_URL'))"
python -c "import os; print('Embedding URL:', os.getenv('DEEPWIKI_EMBEDDING_BASE_URL'))"
```

---

## 🔧 상세 문서 생성 설정

### 현재 적용된 최적화 설정

이 프로젝트는 이미 **상세 문서 생성 모드**로 설정되어 있습니다.

#### ✅ `deepwiki_single.py:94-96` - 청킹 설정

```python
# Increased chunk_size and overlap for more detailed documentation
chunker = TextChunker(chunk_size=1000, overlap=200)
```

**효과**:
- 코드 블록을 더 크게 분할하여 컨텍스트 유지
- 청크 간 중복이 커져서 정보 손실 방지

#### ✅ `deepwiki_single.py:109-113` - 컨텍스트 토큰 설정

```python
# Increased max_context_tokens for more detailed documentation
generator = PageGenerator(
    llm_client=llm,
    embedder_client=embedder,
    max_context_tokens=10000,  # 기본값: 6000
)
```

**효과**:
- 각 위키 페이지에 더 많은 정보 포함
- 약 40,000자의 컨텍스트 사용 가능 (기본값: 24,000자)

#### ✅ `api/pipeline/generate.py:145` - RAG 검색 개수

```python
# Retrieve relevant chunks (increased for detailed documentation)
relevant_chunks = self._retrieve_chunks(query, top_k=20)  # 기본값: 10
```

**효과**:
- 관련 코드를 2배 더 많이 검색
- 더 풍부한 예제와 설명

#### ✅ `api/pipeline/generate.py:154-156` - 파일 참조 한도

```python
# Add file summaries (increased limits for detailed documentation)
for file_data in matched_files[:20]:  # 기본값: 10
    path = file_data['path']
    content = file_data['content'][:4000]  # 기본값: 2000
```

**효과**:
- 더 많은 파일 참조
- 파일당 2배 더 많은 내용 포함

#### ✅ `api/pipeline/generate.py:161-163` - 청크 텍스트 한도

```python
# Add retrieved chunks (increased limit for detailed documentation)
for chunk in relevant_chunks:
    context_parts.append(f"\n[{chunk['source']}]\n{chunk['text'][:2000]}\n")  # 기본값: 1000
```

**효과**:
- 각 코드 청크의 내용이 2배로 증가
- 더 완전한 함수/클래스 정의 포함

---

## 🚀 실행 방법

### 옵션 1: 전체 프로세스 (권장)

```bash
# 1단계: Dry-run으로 테스트 (파일 생성 안 함, 결과 미리보기)
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --output ./wiki_output \
  --dry-run \
  --debug

# 2단계: 실제 위키 생성
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --output ./wiki_output
```

### 옵션 2: Private 저장소

```bash
# GitHub 토큰 사용 (private repo)
python deepwiki_single.py \
  --repo https://github.com/your-org/private-repo \
  --token ghp_YourGitHubPersonalAccessToken \
  --output ./wiki_output
```

### 옵션 3: 커스텀 워크스페이스

```bash
# 특정 디렉토리에 저장소 클론
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --workspace /tmp/deepwiki-workspace \
  --output ./wiki_output
```

### 실행 과정 설명

실행하면 다음 7단계가 진행됩니다:

```
[1/7] Initializing clients...
✓ LLM client initialized
✓ Embedding client initialized

[2/7] Ingesting repository...
✓ Loaded 156 files

[3/7] Chunking files...
✓ Created 423 chunks (chunk_size=1000, overlap=200)

[4/7] Planning wiki structure...
✓ Planned 5 pages
  - Home: Project overview and quick start
  - Architecture: System design and components
  - API Reference: API documentation
  - Configuration: Setup and configuration guide
  - Development: Development workflow

[5/7] Computing embeddings...
✓ Embeddings computed (1024-dimensional vectors)

[6/7] Generating wiki pages...
  ✓ Generated: Home
  ✓ Generated: Architecture
  ✓ Generated: API Reference
  ✓ Generated: Configuration
  ✓ Generated: Development

[7/7] Writing wiki files...
✓ Wiki written to ./wiki_output
```

---

## 📁 생성된 파일 구조

위키 생성 후 다음과 같은 파일이 생성됩니다:

```
wiki_output/
├── Home.md                    # 프로젝트 개요 및 Quick Start
├── Architecture.md            # 시스템 아키텍처 및 설계
├── API-Reference.md           # API 상세 문서
├── Configuration.md           # 설정 및 환경 구성
├── Development.md             # 개발 가이드
└── _Sidebar.md               # GitHub Wiki 사이드바
```

각 마크다운 파일은:
- **2-4배 더 자세한 설명** 포함
- **더 많은 코드 예제** (20개 파일 참조, 기본값 10개)
- **더 긴 코드 스니펫** (파일당 4000자, 기본값 2000자)
- **더 풍부한 컨텍스트** (10000 토큰, 기본값 6000 토큰)

---

## 🌐 생성된 문서 배포

### 방법 1: GitHub Wiki (가장 추천!)

GitHub의 Wiki 기능을 사용하는 방법입니다.

```bash
# 1. Wiki 저장소 클론 (저장소 설정에서 Wiki 활성화 필요)
git clone https://github.com/your-org/your-repo.wiki.git

# 2. 생성된 문서 복사
cp -r ./wiki_output/* ./your-repo.wiki/

# 3. 커밋 & 푸시
cd your-repo.wiki
git add .
git commit -m "docs: Add detailed wiki documentation generated by DeepWiki"
git push origin master
```

**접근 URL**: `https://github.com/your-org/your-repo/wiki`

**장점**:
- ✅ 사이드바 자동 생성 (`_Sidebar.md`)
- ✅ 검색 기능 내장
- ✅ 별도 설정 불필요
- ✅ 깔끔한 URL

---

### 방법 2: 저장소 내 wiki 폴더

프로젝트 저장소에 직접 포함하는 방법입니다.

```bash
# 1. wiki 디렉토리 생성
mkdir -p wiki

# 2. 생성된 문서 복사
cp -r ./wiki_output/* ./wiki/

# 3. 커밋 & 푸시
git add wiki/
git commit -m "docs: Add detailed wiki documentation"
git push origin main
```

**접근 URL**: `https://github.com/your-org/your-repo/tree/main/wiki`

**장점**:
- ✅ 코드와 함께 버전 관리
- ✅ 로컬에서 바로 볼 수 있음
- ✅ PR 리뷰 가능

---

### 방법 3: GitHub Pages

정적 웹사이트로 호스팅하는 방법입니다.

```bash
# 1. docs 디렉토리에 복사
mkdir -p docs
cp -r ./wiki_output/* ./docs/

# 2. 커밋 & 푸시
git add docs/
git commit -m "docs: Add wiki documentation for GitHub Pages"
git push origin main

# 3. GitHub 저장소 설정
# Settings → Pages → Source: Deploy from branch
# Branch: main, Folder: /docs
```

**접근 URL**: `https://your-org.github.io/your-repo`

**장점**:
- ✅ 커스텀 테마 적용 가능
- ✅ Jekyll 자동 변환
- ✅ 독립적인 웹사이트

---

## 🔍 생성된 문서 품질 확인

### Dry-run으로 미리보기

파일을 실제로 쓰지 않고 어떤 내용이 생성될지 확인:

```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --dry-run \
  --debug
```

**출력 예시**:
```
[DRY RUN] Would write: ./wiki_output/Home.md (8245 chars)
[DRY RUN] Would write: ./wiki_output/Architecture.md (12821 chars)
[DRY RUN] Would write: ./wiki_output/API-Reference.md (9502 chars)
```

**주의**: Dry-run이어도 LLM API는 실제로 호출되므로 비용이 발생합니다!

### 생성된 문서 통계

```bash
# 파일 개수 및 크기 확인
ls -lh wiki_output/

# 총 단어 수 확인
wc -w wiki_output/*.md

# 총 라인 수 확인
wc -l wiki_output/*.md
```

---

## 📊 비용 및 성능

### 예상 API 호출 수

중간 크기 프로젝트 (100-200 파일) 기준:

| 단계 | API 호출 | 토큰 사용량 (추정) |
|------|---------|------------------|
| Planning | 1회 (LLM) | ~2000 토큰 |
| Embedding | 1회 (Batch) | N/A (BGE-M3) |
| Generation | 페이지당 1회 (LLM) | 페이지당 ~10000 토큰 |

**예시**: 5개 페이지 생성 시
- LLM 호출: 6회 (Planning 1 + Pages 5)
- 총 토큰: ~52000 토큰 (Planning 2000 + Pages 50000)

### 실행 시간

| 프로젝트 크기 | 파일 수 | 예상 시간 |
|-------------|--------|---------|
| 소규모 | 10-50 | 2-5분 |
| 중규모 | 50-200 | 5-15분 |
| 대규모 | 200-1000 | 15-45분 |

**참고**: 실행 시간은 네트워크 속도와 API 응답 시간에 따라 다름

---

## 🛠️ 트러블슈팅

### 문제 1: API 연결 실패

**증상**:
```
Error: Failed to connect to LLM API
```

**해결 방법**:
```bash
# 1. 환경 변수 확인
echo $DEEPWIKI_LLM_BASE_URL
echo $DEEPWIKI_LLM_TOKEN

# 2. 네트워크 연결 테스트
curl -H "x-dep-ticket: $DEEPWIKI_LLM_TOKEN" $DEEPWIKI_LLM_BASE_URL

# 3. .env 파일 재로드
source .env  # 또는 새 터미널 세션 시작
```

---

### 문제 2: 임베딩 계산 실패

**증상**:
```
Error: Embedding API returned invalid dimensions
Expected: 1024, Got: 768
```

**해결 방법**:
- BGE-M3 모델이 맞는지 확인
- API 엔드포인트가 올바른지 확인
- 다른 임베딩 모델 사용 시 코드 수정 필요

---

### 문제 3: 메모리 부족

**증상**:
```
MemoryError: Unable to allocate array
```

**해결 방법**:
```python
# deepwiki_single.py에서 설정 조정
chunker = TextChunker(chunk_size=750, overlap=150)  # 1000 → 750로 감소

generator = PageGenerator(
    max_context_tokens=8000,  # 10000 → 8000으로 감소
)
```

---

### 문제 4: 생성된 문서가 너무 짧음

**원인**: 설정이 기본값으로 되어있을 수 있음

**확인 방법**:
```bash
# 현재 설정 확인
grep "chunk_size" deepwiki_single.py
grep "max_context_tokens" deepwiki_single.py
grep "top_k" api/pipeline/generate.py
```

**해결 방법**:
- 이 가이드의 [상세 문서 생성 설정](#상세-문서-생성-설정) 섹션 참조
- 설정 값이 올바른지 재확인

---

### 문제 5: GitHub Wiki 푸시 실패

**증상**:
```
fatal: could not read Username for 'https://github.com'
```

**해결 방법**:
```bash
# 1. GitHub 인증 설정
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# 2. Personal Access Token 사용
# GitHub Settings → Developer settings → Personal access tokens
# 토큰 생성 후 비밀번호 대신 사용

# 3. SSH 사용
git clone git@github.com:your-org/your-repo.wiki.git
```

---

### 문제 6: Embedding API Rate Limit 초과 ⚠️

**증상**:
```
HTTP 429: Rate limit exceeded
Error: Too many requests
```

**원인**:
- 짧은 시간 내 너무 많은 API 호출
- 배치 크기가 너무 큼 (기본값: 50)
- 배치 간 delay가 부족 (기본값: 1초)

**해결 방법**:

**옵션 1: 환경 변수로 설정 (권장)**

`.env` 파일에 추가:
```bash
# Rate Limit이 약할 때
DEEPWIKI_EMBEDDING_BATCH_SIZE=50
DEEPWIKI_EMBEDDING_BATCH_DELAY=1.0

# Rate Limit이 중간일 때
DEEPWIKI_EMBEDDING_BATCH_SIZE=30
DEEPWIKI_EMBEDDING_BATCH_DELAY=2.0

# Rate Limit이 강할 때
DEEPWIKI_EMBEDDING_BATCH_SIZE=20
DEEPWIKI_EMBEDDING_BATCH_DELAY=3.0
```

**옵션 2: 자동 재시도 활용**

코드가 이미 자동 재시도를 지원합니다:
- 1차 실패 → 5초 대기 후 재시도
- 2차 실패 → 10초 대기 후 재시도
- 3차 실패 → 20초 대기 후 재시도

**상세 가이드**: [RATE_LIMIT_GUIDE.ko.md](RATE_LIMIT_GUIDE.ko.md) 참조

---

## 📚 추가 자료

### 관련 문서

- [README.md](README.md) - 프로젝트 전체 문서
- [README_SINGLE_PROVIDER.md](README_SINGLE_PROVIDER.md) - Single Provider 가이드
- [DOCKER_SETUP.md](DOCKER_SETUP.md) - Docker 설치 가이드
- [RATE_LIMIT_GUIDE.ko.md](RATE_LIMIT_GUIDE.ko.md) - **Rate Limit 해결 가이드** ⭐
- [wiki/USAGE_EXAMPLES.md](wiki/USAGE_EXAMPLES.md) - 20개 실전 예제
- [.env.example](.env.example) - 환경 변수 템플릿

### 명령줄 옵션 전체 목록

```bash
python deepwiki_single.py --help
```

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `--repo` | GitHub 저장소 URL (필수) | - |
| `--output` | 출력 디렉토리 | `./wiki_output` |
| `--token` | GitHub 토큰 (private repo용) | - |
| `--workspace` | 클론 워크스페이스 | `./workspace` |
| `--dry-run` | 파일 쓰기 없이 테스트 | `False` |
| `--debug` | 디버그 로그 활성화 | `False` |

---

## 🎯 요약

### 빠른 시작 체크리스트

- [ ] `.env` 파일 설정 (LLM, Embedding API)
- [ ] 의존성 설치 (`pip install -r requirements_single.txt`)
- [ ] Dry-run 테스트 (`--dry-run --debug`)
- [ ] 실제 위키 생성 (`python deepwiki_single.py --repo URL`)
- [ ] 생성된 문서 확인 (`ls -lh wiki_output/`)
- [ ] GitHub Wiki 또는 저장소에 배포

### 설정 비교표

| 항목 | 기본 설정 | 상세 문서 설정 (현재) | 증가율 |
|------|---------|-------------------|--------|
| Chunk Size | 500 단어 | 1000 단어 | **+100%** |
| Overlap | 100 단어 | 200 단어 | **+100%** |
| Context Tokens | 6000 | 10000 | **+67%** |
| RAG Top-K | 10개 | 20개 | **+100%** |
| Matched Files | 10개 | 20개 | **+100%** |
| File Content | 2000자 | 4000자 | **+100%** |
| Chunk Text | 1000자 | 2000자 | **+100%** |

**전체적으로 약 2-3배 더 자세한 문서가 생성됩니다!**

---

## 🤝 지원

문제가 발생하거나 질문이 있으시면:

1. 이 가이드의 [트러블슈팅](#트러블슈팅) 섹션 확인
2. [README.md](README.md)의 전체 문서 참조
3. GitHub Issues에 문의

---

**생성일**: 2024년 12월
**버전**: 2.0 (Detailed Documentation Mode)
**최적화 대상**: GPT-OSS 120B/130B + BGE-M3
