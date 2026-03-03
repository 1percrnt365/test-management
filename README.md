# test-management

> A Claude Code skill for systematic test file management across any project.

---

## English

### Overview

As projects grow, test files accumulate — scattered in the wrong directories, pointing to deleted modules, or duplicated across features. This skill provides a **6-step workflow** to scan, classify, archive, and organize test files without losing history.

**Core principle: Never delete tests. Classify and move them.**

### Supported Languages / Frameworks

| Language | Test Framework | Archive Strategy |
|----------|---------------|-----------------|
| Python | pytest | `git mv` → `tests/archive/YYYY-MM/` |
| Rust | cargo test | `git mv` (integration) / `#[ignore]` (inline) |
| JavaScript / TypeScript | Jest / Vitest | `git mv` → `tests/archive/` |

### When to Use

- Test files are scattered across the project root
- 10+ test files have accumulated and you're losing track
- Test suite takes more than 5 minutes to run
- You're unsure which tests are still valid
- After AI-generated workflows (bkit, superpowers TDD) leave behind unorganized tests
- CI/CD is running unnecessary or stale tests

### 6-Step Workflow

```
Step 1: Scan     → Find all test files, count them, check distribution
Step 2: Classify → Label each as Active / Stale / Orphan / Duplicate
Step 3: Structure → Set up tests/unit, integration, e2e, archive folders
Step 4: Move     → git mv stale files into archive/YYYY-MM/ with ARCHIVE_LOG.md
Step 5: Tag      → Add markers (pytest: @pytest.mark.critical, Rust: #[ignore])
Step 6: CI/CD    → Wire up fast/slow job split in GitHub Actions
```

### Quick Reference

| Task | Python | Rust | JS/TS |
|------|--------|------|-------|
| Run core tests only | `pytest -m "critical"` | `cargo test --lib` | `npx vitest --testPathPattern="unit"` |
| Include slow tests | `pytest` | `cargo test -- --include-ignored` | `npx vitest` |
| List all tests | `pytest --co -q` | `cargo test -- --list` | `npx vitest --reporter=verbose` |
| Coverage | `pytest --cov` | `cargo tarpaulin` | `npx vitest --coverage` |
| Archive a file | `git mv <file> tests/archive/$(date +%Y-%m)/` | same / `#[ignore]` | `git mv <file> tests/archive/` |
| Exclude archive from CI | `--ignore=tests/archive` | automatic (Cargo) | `--testPathIgnorePatterns="archive"` |

### Installation

```bash
mkdir -p ~/.claude/skills/test-management
curl -o ~/.claude/skills/test-management/SKILL.md \
  https://raw.githubusercontent.com/1percrnt365/test-management/main/SKILL.md
```

Then invoke in Claude Code:

```
/test-management
```

### Maintenance Routine

| Frequency | Task | Time |
|-----------|------|------|
| Every commit | pre-commit: run critical tests | auto |
| Every Friday | Check for flaky tests, add regression tests for new bugs | 15 min |
| 1st of every month | Rescan, archive 30-day-old stale tests, check coverage | 30 min |
| After each release | Full test run (including slow), identify orphans | 30 min |

---

## 한국어

### 개요

프로젝트가 진행될수록 테스트 파일이 쌓입니다 — 잘못된 폴더에 방치되거나, 삭제된 모듈을 가리키거나, 기능별로 중복됩니다. 이 스킬은 히스토리를 잃지 않으면서 테스트 파일을 스캔·분류·아카이빙·정리하는 **6단계 워크플로우**를 제공합니다.

**핵심 원칙: 테스트는 삭제하지 않는다. 분류하고 이동한다.**

### 지원 언어 / 프레임워크

| 언어 | 테스트 프레임워크 | 아카이브 전략 |
|------|-----------------|--------------|
| Python | pytest | `git mv` → `tests/archive/YYYY-MM/` |
| Rust | cargo test | `git mv` (integration) / `#[ignore]` (인라인) |
| JavaScript / TypeScript | Jest / Vitest | `git mv` → `tests/archive/` |

### 사용 시점

- 테스트 파일이 프로젝트 루트에 산발적으로 흩어져 있을 때
- 테스트 파일이 10개 이상 누적되어 파악이 안 될 때
- 테스트 스위트 실행 시간이 5분을 넘길 때
- 어떤 테스트가 아직 유효한지 모를 때
- AI 생성 워크플로우(bkit, superpowers TDD 등) 이후 정리가 필요할 때
- CI/CD에서 불필요하거나 오래된 테스트가 실행될 때

### 6단계 워크플로우

```
1단계: 스캔     → 모든 테스트 파일 탐색, 개수 확인, 분포 파악
2단계: 분류     → Active / Stale / Orphan / Duplicate 레이블 부여
3단계: 구조화   → tests/unit, integration, e2e, archive 폴더 정리
4단계: 이동     → git mv로 오래된 파일을 archive/YYYY-MM/으로 이동 + ARCHIVE_LOG.md 기록
5단계: 태깅     → 마커 추가 (pytest: @pytest.mark.critical, Rust: #[ignore])
6단계: CI/CD   → GitHub Actions에서 빠른/느린 job 분리 설정
```

### 판단 기준 플로우

```
테스트 파일 발견
    │
    ├─ 대상 코드가 존재하는가?
    │   ├─ No  → Archive (Orphan)
    │   └─ Yes ↓
    │
    ├─ 실행하면 통과하는가?
    │   ├─ No  → 수정 가능? → Yes: 수정 / No: Archive (Stale)
    │   └─ Yes ↓
    │
    ├─ 30일 내 수정된 적 있는가?
    │   ├─ No  → 대상 코드도 안 바뀜? → Active 유지
    │   │        대상 코드는 바뀜?    → 검증 후 업데이트
    │   └─ Yes → Active 유지
    │
    └─ 중복 테스트인가?
        ├─ Yes → 최신 것 유지, 나머지 Archive
        └─ No  → Active 유지
```

### 설치 방법

```bash
mkdir -p ~/.claude/skills/test-management
curl -o ~/.claude/skills/test-management/SKILL.md \
  https://raw.githubusercontent.com/1percrnt365/test-management/main/SKILL.md
```

Claude Code에서 호출:

```
/test-management
```

### 유지보수 루틴

| 주기 | 작업 | 시간 |
|------|------|------|
| 매 커밋 | pre-commit: critical 테스트 자동 실행 | 자동 |
| 매주 금요일 | flaky 테스트 정리, 새 버그 회귀 테스트 추가 | 15분 |
| 매월 1일 | 재스캔, 30일+ 오래된 테스트 아카이브, 커버리지 확인 | 30분 |
| 매 릴리스 후 | 전체 테스트 실행(slow 포함), orphan 식별 | 30분 |

---

## License

MIT
