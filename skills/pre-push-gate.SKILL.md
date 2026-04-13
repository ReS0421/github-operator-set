---
name: pre-push-gate
description: Remote publication safety gate for push, PR, and lightweight preflight checks. Use before git push or PR creation, or when the user asks for a pre-push review. Deterministic-first, changed-target-first, severity-based blocking. Keep this as a standalone skill; do not treat it as an internal team-orchestrator phase.
---

# pre-push-gate

## Purpose

`pre-push-gate`는 git push / PR 직전, 또는 수동 preflight 요청 시 실행하는 **독립 safety gate**다.

목표:
- remote 반영 직전의 저비용 치명 이슈 차단
- deterministic-first 검사를 우선 실행
- 필요한 경우에만 reviewer를 붙여 비용 통제
- 최종적으로 명확한 verdict 반환

이 스킬은 구현/리뷰 스킬이 아니다.
**코드를 설계하거나 리팩터링하는 역할이 아니라, 지금 반영하려는 변경을 내보내도 되는지 판단하는 gate**다.

---

## Positioning

- standalone skill로 유지한다
- GitHub 팀 내 publication gate specialist로 둔다
- 상위 진입점은 `github-ops`다
- `team-orchestrator` 내부 phase로 간주하지 않는다
- 필요 시 openclaw가 외부에서 호출한다
- 수동 preflight / push 직전 / PR 직전 흐름에서 공용으로 쓴다
- 기존 `github-publish-checklist`, `github-doc-consistency`의 audit 성격 일부를 profile/checkset으로 흡수한다

---

## Trigger

다음 상황에서 사용:
- 사용자가 `push 해줘`, `push 전에 점검해줘`, `PR 올리기 전에 봐줘`라고 할 때
- 코드 변경 후 remote 반영 직전 최종 safety check가 필요할 때
- `pre-push` / `preflight` / `push gate` 류 요청이 있을 때

기본 비적용:
- 일반 구현 작업 중간
- 장기 아키텍처 리뷰
- 전체 repo full audit

---

## Core Principles

1. **Deterministic-first**
   - lint / test / build / grep류 검사를 먼저
   - AI는 후순위

2. **Changed-target-first**
   - 무엇을 내보낼지 먼저 정의한다
   - 기본 입력은 `change_target` + `input_ref`

3. **Severity-based gate**
   - Critical/High는 block
   - Medium은 warn 중심

4. **Explainable verdict**
   - 왜 막혔는지, 어디가 문제인지, 무엇을 해야 하는지 명시

5. **Standalone reuse**
   - coding-team 외 흐름에도 재사용 가능해야 함

---

## Input Model

### change_target
반드시 아래 중 하나를 먼저 고른다.

v1 기본 우선순위는 **`staged` 우선**이다.
즉, 명시적 다른 의도가 없으면 commit 전 staged 변경을 먼저 본다.

- `staged`
  - 기본값. 아직 commit 전 staged 변경 검토
- `head_vs_upstream`
  - commit 이후 push 직전, 현재 HEAD와 upstream 차이 검토
- `branch_vs_base`
  - PR 직전 브랜치 diff 검토
- `deploy_commit`
  - 특정 commit/release payload의 lightweight gate

### input_ref
판정 기준점을 문자열로 고정한다.
예:
- `git_index@2026-04-09T16:00:00+09:00`
- `HEAD..origin/main`
- `merge-base(main, HEAD)..HEAD`
- `deploy:abc1234`

입력 기준점을 못 고정하면 애매한 판정이 되므로, 먼저 input을 명확히 한다.

---

## Standard Flow

### Step 0. Preconditions
최소 입력:
- `repo_root`
- `current_branch`
- `intent` (`push|pr|deploy|preflight`)
- `change_target`
- `input_ref`

확인:
- git repo인지
- change_target을 정했는지
- input_ref를 고정했는지
- branch / base / upstream 정보가 필요한 경우 확보됐는지

자동 추론 원칙:
- `repo_root`, `current_branch`는 가능한 한 자동 추론
- `change_target`은 명시 의도가 없으면 `staged` 우선
- `input_ref`는 선택된 target에 맞게 계산
- 그래도 애매하면 중단하고 사용자 확인

실패 시 중단하고 필요한 입력을 명확히 요청/정리한다.

### Step 1. Diff Classification
입력을 분류한다.

기본 플래그:
- docs_only
- code_changed
- infra_touched
- auth_touched
- db_touched
- dependency_changed
- large_diff
- deploy_sensitive

주의:
- 경로명만으로 과하게 판정하지 않는다
- generated/lockfile/fixture는 보정 고려

### Step 2. Hard Block Scan
먼저 저비용 고위험 검사.

최소 범위:
- merge conflict marker
- private key header
- known secret/token patterns
- obvious hardcoded credential assignment
- protected branch direct push 여부

원칙:
- secret / private key / conflict marker → 즉시 block
- protected branch direct push는 기본적으로 `NEEDS_CONFIRMATION`
- 단, 정책상 direct push 금지 저장소라면 `BLOCKED`로 승격 가능

### Step 3. Fast Path Check
조건 충족 시 무거운 검사 생략 가능.

fast path 후보:
- pure docs-only
- markdown note/docs

fast path 제외 예시:
- `.github/workflows/**`
- deployment yaml
- 실행 스크립트 변경
- runtime behavior를 바꾸는 config 문서

### Step 4. Direct Validation
결정론적 validator를 repo 설정에 맞춰 실행.

#### JS/TS
- `package.json` scripts 확인
- `build`, `test`, `lint`, `typecheck` 중 존재하는 것만 실행
- monorepo면 changed scope 우선
- 없으면 `skip_with_warning`

#### Python
- `pyproject.toml`, `setup.py`, `requirements.txt` 확인
- `ruff` 우선, 없으면 `flake8`
- `pytest` 가능 시 실행
- 없으면 `skip_with_warning`

#### Go
- `go.mod` 존재 시 `go test ./...`, `go vet ./...`

공통:
- silent skip 금지
- timeout 필요
- 실패 시 verdict에 명시

validator 상태는 아래 셋 중 하나로 기록한다.
- `run` — 실제 실행함
- `skip_with_warning` — 실행 조건이 모자라 건너뜀
- `not_applicable` — 현재 change_target / 언어 / repo 구조상 해당 없음

### Step 5. Conditional Risk Review
필요한 경우에만 reviewer를 붙인다.

v1 기본 모델:
- unified `pre-push-risk-reviewer`
- routing tag만 구분: `security`, `infra`, `db`, `large_diff`

최소 호출 규칙:
- `auth_touched=true` → `security`
- `infra_touched=true` 또는 `deploy_sensitive=true` → `infra`
- `db_touched=true` → `db`
- `large_diff=true` → `large_diff`

AI reviewer 규칙:
- evidence 기반 finding만 허용
- AI 단독 block 금지
- deterministic failure를 보강하는 보조 신호로 사용 가능

### Step 6. Verdict Synthesis
최종 verdict는 아래 중 하나:
- `PASS`
- `PASS_WITH_WARNINGS`
- `BLOCKED`
- `NEEDS_CONFIRMATION`

---

## Blocked Classes

`BLOCKED` 또는 실질적 중단 상황은 아래 성격으로 나눈다.

### mechanical
예:
- secret
- merge conflict
- lint/test/build 실패

처리:
- 로컬 수정
- gate 재실행

### contract_risk
예:
- auth bypass 의심
- migration mismatch
- remote publication risk가 reviewer evidence로 확인됨

처리:
- targeted review 또는 구현/review 단계로 되돌림

### confirmation
예:
- protected branch direct push
- 민감 환경 반영

처리:
- 사용자 확인 후 재개

---

## False Positive Mitigation

- `.env.example`, `tests/`, `__tests__/`, `fixtures/`, docs 예시는 기본 warn 우선
- auth/security touched는 경로명 단독 판정 금지
- large diff는 generated/lockfile/rename-heavy 보정 고려
- docs-only fast path는 실행 경로 변경 파일을 제외

---

## Profiles

`pre-push-gate`는 단일 체크리스트가 아니라 **profile-driven gate**로 운용한다.

### `default`
- 일반 push / PR 직전 기본 gate
- diff classification → hard block scan → direct validation → verdict

### `public_repo`
공개 레포 반영 전 강화 gate. 기존 `github-publish-checklist`의 성격을 흡수한다.

추가 확인 항목:
- file exposure / private docs 혼입
- secret / credential / session artifact 노출
- `.gitignore` 기본 안전성
- README / LICENSE / reproducibility 최소 기준
- public suitability / personal info / local path independence

### `doc_consistency`
문서 정합성 중심 gate. 기존 `github-doc-consistency`의 성격을 흡수한다.

추가 확인 항목:
- placeholder mismatch
- env var / path naming consistency
- code ↔ docs consistency
- metadata consistency (project name, license, version)

### `pr_preflight`
PR 생성 직전 lightweight validation.

추가 확인 항목:
- branch diff / staged diff 기준 확인
- lint / test / typecheck의 실행 가능 범위 확인
- obvious contract risk / protected branch 흐름 확인

---

## Transition Notes

- 신규 진입점은 `pre-push-gate`를 사용한다.
- `github-publish-checklist`, `github-doc-consistency`는 즉시 삭제보다 deprecated 전환을 우선한다.
- 다른 문서에서 두 스킬을 직접 호출 중이면 우선 `pre-push-gate` profile 참조로 치환한다.

---

## Output Contract

최종 응답은 가능하면 아래 구조를 따른다.

```yaml
verdict: PASS|PASS_WITH_WARNINGS|BLOCKED|NEEDS_CONFIRMATION
blocked_class: mechanical|contract_risk|confirmation|null
intent: push|pr|deploy|preflight
input_ref: <string>
summary: <one-line>
validators_run: []
reviewer_routes: []
blocking_findings: []
warnings: []
next_step: <string>
rerun_condition: <string>
```

핵심은:
- 무엇을 기준으로 판단했는지
- 무엇이 문제인지
- 다음 행동이 무엇인지
를 분명히 남기는 것이다.

---

## Non-Goals

이 스킬이 하지 않는 것:
- 전체 코드베이스 장기 품질 개선
- 대규모 아키텍처 리디자인
- 제품 스펙 검토
- full security audit

그건 별도 reviewer / audit 흐름의 책임이다.

---

## Implementation Notes for openclaw

- 이 스킬은 필요 시 shell/lint/test와 reviewer를 조합해 실행한다
- push 전에 항상 mandatory로 고정할지 여부는 호출자 정책의 책임이다
- 즉, 이 skill은 **gate implementation**, mandatory enforcement는 **caller policy**다
