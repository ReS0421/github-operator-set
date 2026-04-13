---
name: github-ops
description: "GitHub 팀 상위 진입점 — GitHub 관련 요청을 intake하고, repo bootstrap / pre-push gate / issue automation / direct gh operations 중 적절한 specialist로 라우팅한다. Trigger: GitHub 관련 작업 전반, 적절한 하위 스킬 선택이 필요한 경우."
---

# GitHub Ops

GitHub 관련 요청의 단일 진입점이다.

## Purpose

`github-ops`는 GitHub 작업을 직접 다 수행하는 스킬이 아니다.
역할은 다음 세 가지다:

1. 요청 의도 분류
2. 적절한 specialist 선택
3. 공통 전제조건/입력 정리

즉, GitHub 팀의 intake + router + lightweight orchestrator 역할이다.

---

## Use When

다음 상황에서 사용:
- 사용자가 "깃허브 쪽 처리해줘"처럼 넓게 요청할 때
- 어떤 GitHub 스킬로 가야 할지 애매할 때
- repo 생성 / push 전 점검 / issue 자동 처리 / PR 운영 중 무엇이 맞는지 판단이 필요할 때

기본 비적용:
- 이미 하위 specialist가 명확히 지정된 경우
- 단순 local git 작업만 필요한 경우
- GitHub가 아닌 다른 forge(GitLab 등) 작업

---

## Specialist Map

### 1. `pre-push-gate`
사용:
- push 전에 점검해줘
- PR 올리기 전에 봐줘
- preflight / pre-push review 요청

역할:
- remote publication 직전 verdict 반환
- deterministic-first gate
- `default` / `public_repo` / `doc_consistency` / `pr_preflight` profile 지원

### 2. `github-repo-creator`
사용:
- 레포 만들어줘
- 깃허브에 올리자
- 로컬 프로젝트를 GitHub 레포로 공개/연결하고 싶을 때

역할:
- 로컬 프로젝트 → GitHub 레포 bootstrap flow

### 3. `gh-issues`
사용:
- 이슈들 가져와서 처리해줘
- bug label 이슈 자동으로 고치자
- 리뷰 코멘트까지 추적하자

역할:
- issue fetch → implementation orchestration → PR → review follow-up

### 4. `github`
사용:
- PR 상태 확인
- CI 로그 보기
- issue/PR 생성, 댓글, merge
- gh CLI 기반 직접 운영 작업

역할:
- direct GitHub operations via gh CLI

---

## Routing Rules

### Route to `github-repo-creator`
- 새 레포 생성
- 공개/비공개 레포 초기 세팅
- README/.gitignore/LICENSE scaffold 포함 요청

### Route to `pre-push-gate`
- push / PR / release 직전 검토 요청
- "올리기 전에 봐줘" 류 요청
- publication safety 판단 필요

### Route to `gh-issues`
- issue batch 처리
- issue 기반 수정 자동화
- review comment follow-up automation

### Route to `github`
- PR/issue/CI/run/log/review/merge 조회 및 조작
- 특정 GitHub object에 대한 직접 operation

---

## Guardrails

1. GitHub 관련이라고 모두 한 스킬에서 처리하지 않는다.
2. specialist가 명확하면 그쪽으로 바로 보낸다.
3. local git 작업과 GitHub remote 작업을 구분한다.
4. publication 직전 판단은 `pre-push-gate`가 담당한다.
5. repo bootstrap은 `github-repo-creator`, long-running issue automation은 `gh-issues`가 담당한다.
6. `github-ops`는 팀장이지 만능 구현체가 아니다. 상세 실행 로직을 내부에 중복 정의하지 않는다.

---

## Naming Notes

- 사용자-facing 명칭은 `github-ops`로 통일한다.
- leaf specialist는 내부 역할 명칭을 유지한다.
- deprecated 예정인 audit 스킬(`github-publish-checklist`, `github-doc-consistency`)은 신규 진입점으로 사용하지 않는다.

---

## Transition Plan

단계적 전환을 권장한다:
1. `github-ops` 신설
2. Skills Registry / Org Chart 갱신
3. `pre-push-gate`에 profile 개념 추가
4. `github-repo-creator`가 `pre-push-gate` profile을 참조하도록 변경
5. `github-publish-checklist`, `github-doc-consistency`는 deprecated 표기 후 점진 퇴역

완전 삭제는 참조가 모두 정리된 뒤 진행한다.
