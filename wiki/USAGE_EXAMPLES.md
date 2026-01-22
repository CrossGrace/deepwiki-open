# 🚀 DeepWiki 사용 예제 모음

## 📋 기본 사용법

### 1. Public 저장소 위키 생성

```bash
python deepwiki_single.py \
  --repo https://github.com/facebook/react
```

**결과**: `./wiki_output/` 디렉토리에 React 위키 생성

---

### 2. Private 저장소 위키 생성

```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/private-repo \
  --token ghp_YourGitHubPersonalAccessToken123456
```

**GitHub Token 생성 방법**:
1. GitHub → Settings → Developer settings → Personal access tokens
2. Generate new token (classic)
3. `repo` 권한 선택
4. 생성된 토큰 복사

---

### 3. 커스텀 출력 디렉토리

```bash
python deepwiki_single.py \
  --repo https://github.com/microsoft/vscode \
  --output ./docs/vscode-wiki
```

**결과**: `./docs/vscode-wiki/` 디렉토리에 생성

---

## 🧪 테스트 및 디버깅

### 4. Dry-run 모드 (파일 생성 안 함)

```bash
python deepwiki_single.py \
  --repo https://github.com/nodejs/node \
  --dry-run
```

**출력 예시**:
```
[DRY RUN] Would write: ./wiki_output/Home.md (8245 chars)
[DRY RUN] Would write: ./wiki_output/Architecture.md (12821 chars)
```

---

### 5. 디버그 모드 (상세 로그)

```bash
python deepwiki_single.py \
  --repo https://github.com/django/django \
  --debug
```

**로그 예시**:
```
DEBUG: Loading file: src/main.py (2456 bytes)
DEBUG: Created chunk from src/main.py: 0-1000 words
DEBUG: Embedding batch 1/5 (100 chunks)
```

---

### 6. Dry-run + Debug 조합

```bash
python deepwiki_single.py \
  --repo https://github.com/kubernetes/kubernetes \
  --dry-run \
  --debug
```

**용도**: API 연결 테스트, 설정 검증, 파일 확인

---

## 🎯 실전 시나리오

### 7. 내부 기업 저장소 문서화

```bash
# 내부 GitLab 저장소
python deepwiki_single.py \
  --repo https://gitlab.company.com/backend/api-service \
  --token glpat-YourGitLabAccessToken \
  --output ./company-wikis/api-service

# 내부 Bitbucket 저장소
python deepwiki_single.py \
  --repo https://bitbucket.company.com/team/project \
  --output ./company-wikis/project
```

---

### 8. 여러 저장소 일괄 처리

**Bash 스크립트 작성** (`generate_all_wikis.sh`):

```bash
#!/bin/bash

# 저장소 목록
repos=(
  "https://github.com/your-org/repo1"
  "https://github.com/your-org/repo2"
  "https://github.com/your-org/repo3"
)

# 각 저장소에 대해 위키 생성
for repo in "${repos[@]}"; do
  repo_name=$(basename "$repo" .git)
  echo "Generating wiki for $repo_name..."

  python deepwiki_single.py \
    --repo "$repo" \
    --output "./wikis/$repo_name" \
    --token "$GITHUB_TOKEN"

  echo "✓ Completed: $repo_name"
done

echo "All wikis generated!"
```

**실행**:
```bash
chmod +x generate_all_wikis.sh
export GITHUB_TOKEN=ghp_YourToken
./generate_all_wikis.sh
```

---

### 9. 정기적 자동 업데이트 (Cron)

**Crontab 설정**:

```bash
# Crontab 편집
crontab -e

# 매주 월요일 오전 2시에 위키 재생성
0 2 * * 1 cd /path/to/deepwiki-open && python deepwiki_single.py --repo https://github.com/your-org/your-repo --output ./wiki_output
```

---

### 10. Docker로 실행

```bash
# Docker 이미지 빌드
docker build -t deepwiki:latest .

# Docker 컨테이너로 실행
docker run --rm \
  -e DEEPWIKI_LLM_BASE_URL=$DEEPWIKI_LLM_BASE_URL \
  -e DEEPWIKI_LLM_TOKEN=$DEEPWIKI_LLM_TOKEN \
  -e DEEPWIKI_EMBEDDING_BASE_URL=$DEEPWIKI_EMBEDDING_BASE_URL \
  -e DEEPWIKI_EMBEDDING_TOKEN=$DEEPWIKI_EMBEDDING_TOKEN \
  -v $(pwd)/wiki_output:/app/wiki_output \
  deepwiki:latest \
  --repo https://github.com/your-org/your-repo
```

---

## 📤 배포 시나리오

### 11. GitHub Wiki에 배포

```bash
# 1. 위키 생성
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --output ./wiki_output

# 2. GitHub Wiki 저장소 클론
git clone https://github.com/your-org/your-repo.wiki.git

# 3. 생성된 파일 복사
cp -r ./wiki_output/* ./your-repo.wiki/

# 4. 커밋 & 푸시
cd your-repo.wiki
git add .
git commit -m "docs: Update wiki documentation"
git push origin master
```

---

### 12. 저장소 내 wiki 폴더에 저장

```bash
# 1. wiki 폴더 생성
mkdir -p wiki

# 2. 위키 생성 (바로 wiki 폴더에)
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --output ./wiki

# 3. 커밋 & 푸시
git add wiki/
git commit -m "docs: Add wiki documentation"
git push origin main
```

---

### 13. GitHub Pages 배포

```bash
# 1. docs 폴더에 생성
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --output ./docs

# 2. 커밋 & 푸시
git add docs/
git commit -m "docs: Update GitHub Pages"
git push origin main

# 3. GitHub 설정
# Repository Settings → Pages
# Source: Deploy from branch
# Branch: main, Folder: /docs
```

---

## 🔧 고급 사용법

### 14. 커스텀 워크스페이스

```bash
# 임시 디렉토리 사용
python deepwiki_single.py \
  --repo https://github.com/torvalds/linux \
  --workspace /tmp/deepwiki-workspace \
  --output ./linux-wiki
```

**용도**:
- 디스크 공간 절약
- 빌드 후 자동 정리
- 여러 인스턴스 동시 실행

---

### 15. 환경 변수로 설정

```bash
# 환경 변수 설정
export DEEPWIKI_LLM_BASE_URL="https://api.company.com"
export DEEPWIKI_LLM_TOKEN="token123"
export DEEPWIKI_EMBEDDING_BASE_URL="https://embed.company.com"
export DEEPWIKI_EMBEDDING_TOKEN="token456"
export DEEPWIKI_OUTPUT="./wikis"
export DEEPWIKI_WORKSPACE="/tmp/workspace"

# 간단히 실행 (설정 불필요)
python deepwiki_single.py --repo https://github.com/your-org/repo
```

---

### 16. 로그 파일 저장

```bash
# 로그를 파일로 저장
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --debug \
  2>&1 | tee deepwiki-$(date +%Y%m%d-%H%M%S).log
```

**결과**: `deepwiki-20241201-143022.log` 형식으로 저장

---

## 📊 성능 최적화

### 17. 대규모 저장소 처리

```bash
# 큰 저장소는 더 많은 시간 필요
timeout 3600 python deepwiki_single.py \
  --repo https://github.com/chromium/chromium \
  --output ./chromium-wiki

# 1시간(3600초) 타임아웃 설정
```

---

### 18. 병렬 처리 (여러 저장소)

```bash
# GNU Parallel 사용
cat repos.txt | parallel -j 4 \
  "python deepwiki_single.py --repo {} --output ./wikis/{/.}"

# repos.txt 내용:
# https://github.com/org/repo1
# https://github.com/org/repo2
# https://github.com/org/repo3
```

---

## 🐛 문제 해결 예제

### 19. API 연결 테스트

```bash
# 환경 변수 확인
echo "LLM URL: $DEEPWIKI_LLM_BASE_URL"
echo "LLM Token: ${DEEPWIKI_LLM_TOKEN:0:10}..."

# API 직접 테스트
curl -X POST "$DEEPWIKI_LLM_BASE_URL/v1/chat/completions" \
  -H "x-dep-ticket: $DEEPWIKI_LLM_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-oss-130b",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

---

### 20. 네트워크 문제 시 재시도

```bash
# 3회 재시도
for i in {1..3}; do
  echo "Attempt $i..."
  python deepwiki_single.py \
    --repo https://github.com/your-org/your-repo \
    && break || sleep 10
done
```

---

## 📚 추가 자료

- [DETAILED_WIKI_GUIDE.ko.md](DETAILED_WIKI_GUIDE.ko.md) - 상세 설정 가이드
- [README_SINGLE_PROVIDER.md](README_SINGLE_PROVIDER.md) - 기본 사용법
- [DOCKER_SETUP.md](DOCKER_SETUP.md) - Docker 실행 가이드

---

**업데이트**: 2024년 12월
**버전**: 2.0
