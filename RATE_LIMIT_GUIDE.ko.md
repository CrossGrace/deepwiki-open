# 🚨 Embedding API Rate Limit 해결 가이드

## 문제 증상

Embedding API 호출 시 다음과 같은 에러가 발생합니다:

```
HTTP 429: Rate limit exceeded
Error: Too many requests
```

---

## 🔍 원인

### 1. 짧은 시간 내 많은 API 호출
- **Chunk 개수 증가**: `chunk_size=1000, overlap=200` 설정으로 더 많은 청크 생성
- **배치 처리**: 100개씩 배치로 나눠서 처리
- **연속 호출**: 배치 간 delay 없이 즉시 다음 호출
- **예시**: 423개 청크 → 5번 API 호출이 연속으로 발생

### 2. API 서버의 Rate Limit 정책
- 특정 시간(예: 1초, 1분) 내 최대 요청 수 제한
- 제한 초과 시 429 에러 반환

---

## ✅ 해결 방법

### 방법 1: 환경 변수로 설정 (권장!)

`.env` 파일에 다음 설정 추가:

```bash
# Embedding API Rate Limit 설정
DEEPWIKI_EMBEDDING_BATCH_SIZE=30        # 배치 크기 줄이기 (기본값: 50)
DEEPWIKI_EMBEDDING_BATCH_DELAY=2.0      # 배치 간 대기 시간 증가 (기본값: 1.0초)
```

**설정 설명**:
- `BATCH_SIZE`: 한 번에 처리할 텍스트 개수 (낮을수록 안전)
- `BATCH_DELAY`: 각 배치 사이의 대기 시간 (높을수록 안전)

---

### 방법 2: 상황별 권장 설정

#### 🟢 약한 Rate Limit (권장)
```bash
DEEPWIKI_EMBEDDING_BATCH_SIZE=50
DEEPWIKI_EMBEDDING_BATCH_DELAY=1.0
```
- **언제**: 대부분의 경우
- **효과**: 423개 청크 → 약 9초 (9배치 × 1초)

---

#### 🟡 중간 Rate Limit
```bash
DEEPWIKI_EMBEDDING_BATCH_SIZE=30
DEEPWIKI_EMBEDDING_BATCH_DELAY=2.0
```
- **언제**: 429 에러가 가끔 발생할 때
- **효과**: 423개 청크 → 약 28초 (14배치 × 2초)

---

#### 🔴 강한 Rate Limit
```bash
DEEPWIKI_EMBEDDING_BATCH_SIZE=20
DEEPWIKI_EMBEDDING_BATCH_DELAY=3.0
```
- **언제**: 429 에러가 자주 발생할 때
- **효과**: 423개 청크 → 약 63초 (21배치 × 3초)

---

#### 🔥 매우 강한 Rate Limit
```bash
DEEPWIKI_EMBEDDING_BATCH_SIZE=10
DEEPWIKI_EMBEDDING_BATCH_DELAY=5.0
```
- **언제**: 위 설정으로도 안 될 때
- **효과**: 423개 청크 → 약 210초 (42배치 × 5초)

---

## 🔧 코드 변경 사항

### 자동 적용된 개선 사항

이 프로젝트는 이미 다음과 같이 최적화되었습니다:

#### 1. 배치 크기 감소
```python
# api/embedding/bge_m3_client.py:30
batch_size: int = 50  # 기존: 100 → 새: 50
```

#### 2. 배치 간 Delay 추가
```python
# api/embedding/bge_m3_client.py:86-95
# 각 배치 사이에 1초 대기
if i + self.batch_size < len(texts):
    logger.debug(f"Waiting {self.batch_delay}s before next batch...")
    time.sleep(self.batch_delay)
```

#### 3. Rate Limit 에러 시 더 오래 대기
```python
# api/embedding/bge_m3_client.py:148-155
if status == 429:
    wait = 5 * (2 ** attempt)  # 5초, 10초, 20초
    logger.warning(f"Rate limit exceeded, waiting {wait}s before retry...")
```

**재시도 로직**:
- 1차 시도 실패 → 5초 대기 후 재시도
- 2차 시도 실패 → 10초 대기 후 재시도
- 3차 시도 실패 → 20초 대기 후 재시도

---

## 📊 실행 시간 비교

### 예시: 423개 청크 처리

| 설정 | 배치 크기 | Delay | 총 배치 수 | 예상 시간 |
|------|---------|-------|----------|----------|
| **기존** | 100 | 0초 | 5 | ~0초 (즉시) ⚠️ |
| **기본** | 50 | 1초 | 9 | ~9초 ✅ |
| **안전** | 30 | 2초 | 15 | ~28초 ✅ |
| **매우 안전** | 20 | 3초 | 22 | ~63초 ✅ |
| **초안전** | 10 | 5초 | 43 | ~210초 ✅ |

**참고**: 실제 API 호출 시간은 포함되지 않음 (추가 시간 필요)

---

## 🧪 테스트 방법

### 1. 현재 설정으로 테스트

```bash
# Dry-run으로 먼저 테스트 (파일 생성 안 함)
python deepwiki_single.py \
  --repo https://github.com/your-org/small-repo \
  --dry-run \
  --debug
```

**확인 사항**:
- ✅ `Processing batch X/Y` 로그 확인
- ✅ `Waiting 1.0s before next batch...` 로그 확인
- ✅ 429 에러가 발생하지 않는지 확인

---

### 2. 설정 조정 후 재테스트

429 에러가 발생하면 `.env` 파일 수정:

```bash
# .env 파일 수정
DEEPWIKI_EMBEDDING_BATCH_SIZE=30
DEEPWIKI_EMBEDDING_BATCH_DELAY=2.0

# 재시도
python deepwiki_single.py \
  --repo https://github.com/your-org/small-repo \
  --dry-run \
  --debug
```

---

## 🔍 로그 확인

### 정상 작동 시 로그

```
[5/7] Computing embeddings...
INFO: Embedding 423 text(s)
INFO: Processing batch 1/9 (50 texts)
DEBUG: Waiting 1.0s before next batch...
INFO: Processing batch 2/9 (50 texts)
DEBUG: Waiting 1.0s before next batch...
...
INFO: Generated 423 embedding(s)
✓ Embeddings computed
```

---

### Rate Limit 발생 시 로그

```
INFO: Processing batch 3/9 (50 texts)
ERROR: HTTP 429: Rate limit exceeded
WARNING: Rate limit exceeded, waiting 5s before retry...
INFO: Processing batch 3/9 (50 texts)  ← 재시도
```

**해결책**: `BATCH_SIZE` 줄이거나 `BATCH_DELAY` 늘리기

---

## 📋 체크리스트

### 문제 해결 순서

- [ ] 1. `.env` 파일에 rate limit 설정 추가
- [ ] 2. `BATCH_SIZE=50`, `BATCH_DELAY=1.0`로 시작
- [ ] 3. Dry-run으로 테스트
- [ ] 4. 429 에러 발생 시 `BATCH_SIZE` 줄이기 (30 → 20 → 10)
- [ ] 5. 여전히 에러 발생 시 `BATCH_DELAY` 늘리기 (2.0 → 3.0 → 5.0)
- [ ] 6. 실제 위키 생성

---

## 💡 추가 팁

### 1. 작은 저장소로 먼저 테스트

```bash
# 작은 저장소로 설정 테스트
python deepwiki_single.py \
  --repo https://github.com/your-org/small-repo \
  --output ./test-wiki
```

---

### 2. API 서버 관리자에게 문의

Rate limit 정책을 확인하여 최적 설정 찾기:
- 분당 최대 요청 수는?
- 초당 최대 요청 수는?
- 배치당 최대 텍스트 수는?

---

### 3. 오프피크 시간 활용

```bash
# Cron으로 새벽에 실행
0 2 * * * cd /path/to/deepwiki-open && python deepwiki_single.py --repo URL
```

---

## 🚨 주의사항

### ⚠️ Dry-run도 API 호출함!

```bash
python deepwiki_single.py --dry-run  # ← API 호출은 실제로 발생!
```

Dry-run은 **파일 쓰기만** 안 하고, **Embedding API는 실제로 호출**됩니다!

---

### ⚠️ 설정 변경 후 재시작 필요

환경 변수 변경 후:
- 새 터미널 세션 시작, 또는
- `source .env` 실행

---

## 📚 관련 문서

- [DETAILED_WIKI_GUIDE.ko.md](DETAILED_WIKI_GUIDE.ko.md) - 상세 설정 가이드
- [wiki/USAGE_EXAMPLES.md](wiki/USAGE_EXAMPLES.md) - 사용 예제
- [.env.example](.env.example) - 환경 변수 템플릿

---

## 🤝 여전히 문제가 있나요?

1. 로그 파일 확인 (debug 모드)
2. API 서버 상태 확인
3. 네트워크 연결 확인
4. API 서버 관리자에게 rate limit 정책 문의

---

**업데이트**: 2024년 12월
**버전**: 2.0 (Rate Limit 해결)
**최적화 대상**: GPT-OSS 120B/130B + BGE-M3
