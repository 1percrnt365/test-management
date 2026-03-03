---
name: test-management
description: Use when test files are accumulating across projects, test suites are growing unwieldy, flaky tests appear, test execution is slow, user asks about test cleanup or archival strategy, or when organizing tests after skill-generated workflows like bkit or superpowers TDD cycles
---

# Test File Management

## Overview

프로젝트가 진행되며 쌓이는 테스트 파일을 체계적으로 정리·분류·아카이빙하는 실용적 관리 전략.

**핵심 원칙:** 테스트는 삭제하지 않는다. 분류하고 이동한다.

## When to Use

**사용 시점:**
- 테스트 파일이 프로젝트 루트에 산발적으로 흩어져 있을 때
- `test_*.py`, `*_test.rs`, `*.test.ts` 등 테스트 파일이 10개 이상 누적되었을 때
- 테스트 실행 시간이 5분을 넘길 때
- 어떤 테스트가 현재 유효한지 파악이 안 될 때
- skill 워크플로우(bkit, superpowers TDD 등) 후 정리가 필요할 때
- CI/CD 파이프라인에서 불필요한 테스트가 실행될 때

**사용하지 않을 때:**
- 테스트 코드 자체를 작성할 때 → superpowers:test-driven-development 사용
- 테스트 프레임워크 설정 → 프로젝트별 문서 참조

## 1단계: 현황 스캔

프로젝트의 테스트 파일 현황을 먼저 파악한다.

```bash
# 테스트 파일 수 확인 (venv/node_modules/target 제외)
find . \( -name "test_*.py" -o -name "*_test.py" -o -name "*_test.rs" \
  -o -name "*.test.ts" -o -name "*.test.js" \) \
  | grep -v -E "(venv|\.venv|node_modules|site-packages|__pycache__|/target/)" \
  | wc -l

# 위치별 분포 확인
find . \( -name "test_*.py" -o -name "*_test.rs" \) \
  -not -path "*/venv/*" -not -path "*/.venv/*" -not -path "*/target/*" \
  | sed 's|/[^/]*$||' | sort | uniq -c | sort -rn

# 최근 수정 기준 정렬 (오래된 것부터)
find . \( -name "test_*.py" -o -name "*_test.rs" \) \
  -not -path "*/venv/*" -not -path "*/target/*" \
  -printf "%T@ %p\n" 2>/dev/null | sort -n | head -20
```

**Windows/Git Bash 대안:**
```bash
find . \( -name "test_*.py" -o -name "*_test.rs" \) \
  -not -path "*/venv/*" -not -path "*/.venv/*" \
  -not -path "*/node_modules/*" -not -path "*/target/*" | while read f; do
  echo "$(stat -c %Y "$f" 2>/dev/null || echo 0) $f"
done | sort -n | head -20
```

## 2단계: 분류 기준

| 분류 | 기준 | 조치 |
|------|------|------|
| **Active** | 현재 코드와 연관, 통과함 | `tests/` 유지 |
| **Stale** | 30일+ 미수정, 대상 코드 변경됨 | 검증 후 업데이트 or Archive |
| **Orphan** | 대상 모듈/함수가 삭제됨 | `tests/archive/` 이동 |
| **Root-scattered** | 프로젝트 루트에 방치 | 적절한 `tests/` 하위로 이동 |
| **Duplicate** | 동일 기능 중복 테스트 | 최신 것만 유지, 나머지 Archive |
| **Skill-generated** | bkit/superpowers가 생성 | 태그 부여 후 분류 |

## 3단계: 폴더 구조 정리

**Python:**
```
project/
├── tests/
│   ├── unit/                 # 단위 테스트 (빠른 것)
│   ├── integration/          # 통합 테스트
│   ├── e2e/                  # E2E (느린 것)
│   └── archive/              # 비활성 테스트 보관
│       ├── ARCHIVE_LOG.md    # 이동 사유 기록
│       └── 2026-02/          # 월별 정리
└── conftest.py               # 공유 fixture
```

**Rust:**
```
project/
├── src/
│   ├── lib.rs                # #[cfg(test)] mod tests 인라인
│   ├── module_a.rs           # 모듈별 인라인 unit test
│   └── module_b.rs
├── tests/                    # Integration tests (각 파일이 독립 바이너리)
│   ├── integration_api.rs
│   ├── integration_db.rs
│   └── archive/              # 비활성 테스트 보관
│       └── ARCHIVE_LOG.md
└── benches/                  # 벤치마크 (criterion)
```

**JS/TS:**
```
project/
├── src/
│   └── __tests__/            # 또는 src 옆 tests/
├── tests/
│   ├── unit/
│   ├── e2e/
│   └── archive/
│       └── ARCHIVE_LOG.md
└── vitest.config.ts
```

**ARCHIVE_LOG.md 형식:**
```markdown
# Test Archive Log

## 2026-02-27
- `test_old_feature.py` → 대상 모듈 `old_feature.py` 삭제됨
- `test_prototype_v1.py` → v2로 대체됨, `tests/unit/test_prototype.py` 활성
```

## 4단계: 이동 실행

```bash
# archive 디렉토리 생성
mkdir -p tests/archive/$(date +%Y-%m)

# 파일 이동 (삭제 아님)
mv test_old_feature.py tests/archive/$(date +%Y-%m)/

# Git 추적 유지하며 이동
git mv test_old_feature.py tests/archive/$(date +%Y-%m)/
```

**Rust 아카이빙 특수사항:**
```bash
# Integration test (tests/ 내 독립 .rs 파일) → 다른 언어와 동일하게 이동
git mv tests/old_integration.rs tests/archive/$(date +%Y-%m)/

# 인라인 unit test (src/ 내 #[cfg(test)]) → 이동 불가, #[ignore] 처리
# 대상 모듈 내에서:
#[test]
#[ignore] // archived: 2026-02 - 대상 함수 deprecated
fn test_old_logic() { ... }
```

**주의사항:**
- `git mv` 사용으로 히스토리 보존
- 이동 전 반드시 `git status` 확인
- 커밋 메시지: `chore: archive stale tests (test-management cleanup)`
- Rust `tests/archive/` 하위 `.rs`는 Cargo가 test target으로 인식하지 않음 (안전)

## 5단계: 태그 기반 선택 실행

**pytest (Python):**
```python
# conftest.py 또는 각 테스트 파일에 마커 추가
import pytest

# 빠른 핵심 테스트
@pytest.mark.critical
def test_core_trading_logic():
    ...

# 느린 통합 테스트
@pytest.mark.slow
def test_full_evolution_pipeline():
    ...

# Skill이 생성한 테스트
@pytest.mark.generated
def test_from_tdd_cycle():
    ...
```

```ini
# pytest.ini 또는 pyproject.toml
[tool.pytest.ini_options]
markers = [
    "critical: 핵심 기능 (항상 실행)",
    "slow: 5초+ 소요 테스트",
    "generated: skill 워크플로우가 생성한 테스트",
    "smoke: 배포 후 최소 검증",
]
```

**실행 명령:**
```bash
# 핵심만 빠르게
pytest -m "critical and not slow"

# CI 전체
pytest -m "not slow"

# 전체 + 느린 것 포함
pytest -m ""

# archive 제외 (기본값)
pytest --ignore=tests/archive
```

**Rust (cargo test):**
```rust
// src/lib.rs 또는 각 모듈 내 인라인 테스트
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_core_logic() {
        // 단위 테스트 (cargo test 기본 실행)
    }

    #[test]
    #[ignore] // 느린 테스트 → cargo test -- --ignored 로만 실행
    fn test_heavy_computation() {
        // 5초+ 소요 테스트
    }
}
```

```rust
// tests/integration_api.rs (독립 integration test)
#[test]
fn test_api_full_flow() {
    // 각 파일이 별도 바이너리로 컴파일됨
}
```

**Rust 실행 명령:**
```bash
# 핵심만 빠르게 (ignore 제외)
cargo test

# 느린 테스트 포함
cargo test -- --include-ignored

# 특정 모듈만
cargo test module_a::tests

# 특정 integration test 파일만
cargo test --test integration_api

# integration tests만 (tests/ 디렉토리)
cargo test --tests

# 인라인 unit tests만 (src/ 내 #[cfg(test)])
cargo test --lib

# 테스트 목록 확인 (실행 안 함)
cargo test -- --list

# archive 디렉토리 내 .rs는 Cargo가 자동 무시 (tests/ 직속만 컴파일)
```

```toml
# Cargo.toml - 테스트 바이너리 명시 (선택)
[[test]]
name = "integration_api"
path = "tests/integration_api.rs"

# archive 내 파일은 [[test]]에 등록하지 않으면 자동 제외
```

**Jest/Vitest (JS/TS):**
```bash
# 특정 태그 실행
npx vitest --testPathPattern="unit"

# 느린 테스트 제외
npx vitest --testPathIgnorePatterns="e2e|archive"
```

## 6단계: CI/CD 연동

**GitHub Actions - Python:**
```yaml
name: Tests (Python)
on: [push, pull_request]
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -r requirements.txt
      - run: pytest tests/unit -m "critical" --tb=short

  integration:
    runs-on: ubuntu-latest
    needs: unit
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -r requirements.txt
      - run: pytest tests/integration --tb=short -x
```

**GitHub Actions - Rust:**
```yaml
name: Tests (Rust)
on: [push, pull_request]
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --lib --no-fail-fast

  integration:
    runs-on: ubuntu-latest
    needs: unit
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --tests --no-fail-fast

  full:
    runs-on: ubuntu-latest
    needs: integration
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo test -- --include-ignored
```

## 정기 클린업 루틴

**매주 금요일 (15분):**
1. 수집되는 테스트 수 확인
   - Python: `pytest --co -q`
   - Rust: `cargo test -- --list 2>/dev/null | grep -c "test$"`
   - JS/TS: `npx vitest --run --reporter=verbose 2>&1 | tail -5`
2. 실패하는 테스트가 있으면 즉시 수정 or Archive
3. flaky 테스트 태그 후 이슈 등록
   - Python: `@pytest.mark.flaky`
   - Rust: `#[ignore]` + `// FIXME: flaky` 주석

**매월 1일 (30분):**
1. 현황 스캔 (1단계) 재실행
2. 30일+ 미수정 테스트 리뷰
3. Archive 대상 이동 + ARCHIVE_LOG.md 업데이트
4. 커버리지 확인
   - Python: `pytest --cov --cov-report=term-missing`
   - Rust: `cargo install cargo-tarpaulin && cargo tarpaulin --out Html`
   - JS/TS: `npx vitest --coverage`

**매 릴리스 후:**
1. 전체 테스트 실행 (slow 포함)
2. Orphan 테스트 식별 → Archive
3. 커밋: `chore: post-release test cleanup`

## 판단 기준 플로우

```
테스트 파일 발견
    │
    ├─ 대상 코드가 존재하는가?
    │   ├─ No → Archive (Orphan)
    │   └─ Yes ↓
    │
    ├─ 실행하면 통과하는가?
    │   ├─ No → 수정 가능? → Yes: 수정 / No: Archive (Stale)
    │   └─ Yes ↓
    │
    ├─ 30일 내 수정된 적 있는가?
    │   ├─ No → 대상 코드도 안 바뀜? → Active 유지
    │   │       대상 코드는 바뀜? → 검증 후 업데이트
    │   └─ Yes → Active 유지
    │
    └─ 중복 테스트인가?
        ├─ Yes → 최신 것 유지, 나머지 Archive
        └─ No → Active 유지
```

## Common Mistakes

| 실수 | 해결 |
|------|------|
| 테스트 파일 삭제 | **절대 삭제 금지.** `tests/archive/`로 이동 |
| 루트에 `test_*.py` 방치 | 즉시 `tests/` 하위로 분류 |
| Archive 사유 미기록 | ARCHIVE_LOG.md에 반드시 기록 |
| venv/target 내 테스트 혼동 | `--ignore` 패턴으로 항상 제외 |
| 태그 없이 전체 실행 | critical/slow 마커 또는 `#[ignore]`로 분리 |
| CI에서 archive 실행 | `--ignore=tests/archive` 필수 |
| Rust archive .rs가 컴파일됨 | `tests/archive/`는 Cargo가 자동 무시. 단, `[[test]]`에 등록된 경우 제거 필요 |
| Rust 인라인 테스트 아카이빙 | `#[cfg(test)]` 블록을 통째로 주석 or 삭제 대신 `#[ignore]` 사용 |

## Quick Reference

| 상황 | Python | Rust | JS/TS |
|------|--------|------|-------|
| 현황 스캔 | `find . -name "test_*.py" \| grep -v venv \| wc -l` | `find . -name "*_test.rs" \| grep -v target \| wc -l` | `find . -name "*.test.ts" \| grep -v node_modules \| wc -l` |
| 핵심만 실행 | `pytest -m "critical and not slow"` | `cargo test --lib` | `npx vitest --testPathPattern="unit"` |
| 느린 것 포함 | `pytest` | `cargo test -- --include-ignored` | `npx vitest` |
| 테스트 목록 | `pytest --co -q` | `cargo test -- --list` | `npx vitest --run --reporter=verbose` |
| 커버리지 | `pytest --cov` | `cargo tarpaulin` | `npx vitest --coverage` |
| 아카이브 이동 | `git mv <file> tests/archive/$(date +%Y-%m)/` | 동일 (integration) / `#[ignore]` (인라인) | `git mv <file> tests/archive/` |
| archive 제외 | `--ignore=tests/archive` | 자동 (Cargo) | `--testPathIgnorePatterns="archive"` |
