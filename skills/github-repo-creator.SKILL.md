---
name: github-repo-creator
description: "GitHub 레포 풀 셋업 — 로컬 프로젝트 디렉토리 기반으로 README/gitignore/LICENSE 생성, 문서 정합성 감사, gh repo create, branch protection, 초기 push까지 일괄 처리. Trigger: ReS가 '레포 만들어줘' 또는 '깃허브에 올리자'라고 할 때."
---

# GitHub Repo Creator Skill

## Quick Facts
- 트리거: 레포 만들어줘, 깃허브에 올리자 요청 시
- 입력: 로컬 프로젝트 경로, 레포 이름, 설명, 공개/비공개 여부
- 출력: GitHub 레포 (README/gitignore/LICENSE 포함, branch protection, 초기 push)
- 관련 스킬: github-ops, pre-push-gate
- 실행 방식: 메인세션 직접

> 로컬 프로젝트 디렉토리 → GitHub 레포 풀 셋업 스킬.
> CASE A 전용: 로컬 디렉토리가 이미 존재하는 상태에서 GitHub 레포를 생성하고 연결.

## 전제 조건

- `gh` CLI 설치 및 인증 완료 (`gh auth status`)
- 로컬 프로젝트 디렉토리 존재 (`~/projects/{name}/`)
- git 설치

---

## STEP 1: INTAKE — 정보 수집

### 1-1. 프로젝트 경로 확인
```bash
# 디렉토리 존재 여부
ls -la ~/projects/{name}/ 2>/dev/null || echo "NOT FOUND"
```

ReS에게 확인할 항목:
1. **프로젝트 경로** — `~/projects/{name}/` 기본값, 다른 경로면 명시
2. **레포 이름** — 디렉토리명 기본값. 변경 원하면 말해줘
3. **레포 설명** — 1줄 (GitHub description에 표시됨)
4. **public / private** — 선택 (public이면 STEP 3 GATE 실행)

> 이 4가지를 한 번에 물어본다. 개별로 나눠 묻지 않는다.

---

## STEP 2: SCAN — 코드 스캔

### 2-1. 언어/스택 자동 감지
```bash
PROJECT_DIR=~/projects/{name}

# 언어/스택 감지
echo "=== 파일 확장자 분포 ==="
find "$PROJECT_DIR" -not -path '*/.git/*' -type f | \
  sed 's/.*\.//' | sort | uniq -c | sort -rn | head -20

echo "=== 스택 파일 존재 여부 ==="
for f in package.json requirements.txt pyproject.toml Cargo.toml go.mod build.gradle pom.xml; do
  [ -f "$PROJECT_DIR/$f" ] && echo "FOUND: $f"
done
```

### 2-2. 민감 파일 스캔
```bash
PROJECT_DIR=~/projects/{name}

echo "=== 민감 파일 패턴 탐지 ==="
find "$PROJECT_DIR" -not -path '*/.git/*' \( \
  -name "*.env" -o -name ".env" -o -name ".env.*" \
  -o -name "*secret*" -o -name "*credential*" -o -name "*token*" \
  -o -name "*.pem" -o -name "*.key" -o -name "id_rsa" \
\) 2>/dev/null

echo "=== 코드 내 시크릿 패턴 ==="
grep -rn \
  -e 'sk-[a-zA-Z0-9]\{20,\}' \
  -e 'github_pat_example_pattern' \
  -e 'AKIA[A-Z0-9]\{16\}' \
  -e 'xox[baprs]-[a-zA-Z0-9-]\+' \
  --include="*.py" --include="*.js" --include="*.ts" \
  --include="*.sh" --include="*.env" \
  "$PROJECT_DIR" 2>/dev/null | grep -v '.git/'
```

**민감 파일 발견 시 → 즉시 중단.**
ReS에게 처리 방법 확인 후 재진행:
- `.gitignore`에 추가만 할지
- 파일 삭제/이동 후 진행할지
- git history에 이미 있으면 history 정리 필요 여부

---

## STEP 3: GATE — public 레포 추가 감사 (public 선택 시만)

`pre-push-gate` 실행 (`profile: public_repo`):

```
Task: pre-push-gate 실행
프로파일: public_repo
대상 경로: ~/projects/{name}/
```

**판정:**
- PASS → STEP 4 진행
- FAIL → 중단. 체크리스트 항목 수정 후 재시작

---

## STEP 4: SCAFFOLD — 파일 생성

### 4-1. .gitignore 생성/보완

감지된 스택 기반으로 `.gitignore` 생성. 이미 존재하면 누락 항목만 추가.

**스택별 기본 항목:**

| 스택 | 추가 항목 |
|---|---|
| Node.js | `node_modules/`, `dist/`, `.env`, `*.log` |
| Python | `__pycache__/`, `*.pyc`, `.venv/`, `venv/`, `dist/`, `*.egg-info/`, `.env` |
| Rust | `target/` |
| Go | `bin/`, `*.exe` |
| Java | `target/`, `*.class`, `*.jar`, `.gradle/` |
| 공통 | `.DS_Store`, `.idea/`, `.vscode/`, `*.swp`, `*.log` |

### 4-2. README.md 초안 생성

```markdown
# {프로젝트명}

{레포 설명 — STEP 1에서 입력받은 1줄}

## Tech Stack

{감지된 스택 배지 또는 목록}

## Getting Started

### Prerequisites

{의존성 목록}

### Installation

```bash
git clone https://github.com/{username}/{repo-name}.git
cd {repo-name}
# 설치 명령 (스택에 따라 자동 생성)
```

### Usage

```bash
# 실행 명령
```

## Environment Variables

| Variable | Description | Required |
|---|---|---|
| `EXAMPLE_VAR` | Description | Yes |

> Copy `.env.example` to `.env` and fill in the values.

## License

{선택된 라이선스}
```

### 4-3. LICENSE 선택

ReS에게 선택 요청:
1. **MIT** — 가장 자유로운 오픈소스
2. **Apache 2.0** — 특허 보호 포함
3. **GPL 3.0** — 카피레프트
4. **없음** — 기본 저작권만

### 4-4. ReS 확인 게이트

생성된 파일 목록과 README 초안을 보여주고 ReS 확인 받기:

```
생성된 파일:
- .gitignore (스택: {감지된 스택})
- README.md (초안)
- LICENSE ({선택된 라이선스})

README 초안:
{내용 표시}

수정할 부분 있으면 말해줘. OK면 다음 단계 진행할게.
```

> **이 게이트를 통과해야 STEP 5로 진행.** 파일 생성 후 바로 넘어가지 않는다.

---

## STEP 4.5: DOC CONSISTENCY CHECK

`pre-push-gate` 실행 (`profile: doc_consistency`):

```
Task: pre-push-gate 실행
프로파일: doc_consistency
대상 경로: ~/projects/{name}/
```

**판정:**
- PASS → STEP 5 진행
- WARN → 리포트 보여주고 ReS 선택 (수정 후 재검사 또는 그냥 진행)
- FAIL → 중단. 수정 후 재검사

---

## STEP 5: CREATE — 레포 생성

### 5-1. gh repo create 실행
```bash
REPO_NAME="{레포 이름}"
DESCRIPTION="{레포 설명}"
VISIBILITY="--public" # 또는 --private

gh repo create "$REPO_NAME" \
  $VISIBILITY \
  --description "$DESCRIPTION" \
  --source=~/projects/{name} \
  --remote=origin
```

### 5-2. Topics 설정
감지된 스택 기반으로 topics 자동 제안:
```bash
gh repo edit --add-topic "{topic1}" --add-topic "{topic2}"
```

ReS에게 topics 확인 후 설정.

### 5-3. Branch Protection 설정 여부 확인

ReS에게 질문:
> main 브랜치 보호 설정할까? (PR 없이 직접 push 차단)
> - 혼자 쓰는 레포라면 불필요
> - 협업하거나 CI/CD 붙일 예정이면 권장

선택 시:
```bash
gh api repos/{owner}/{repo}/branches/main/protection \
  --method PUT \
  --field required_status_checks=null \
  --field enforce_admins=false \
  --field required_pull_request_reviews=null \
  --field restrictions=null
```

---

## STEP 6: CONNECT & PUSH — 연결 및 초기 push

```bash
PROJECT_DIR=~/projects/{name}
cd "$PROJECT_DIR"

# git 초기화 (필요 시)
if [ ! -d .git ]; then
  git init
  git branch -M main
fi

# remote 연결 (gh repo create --source 사용 시 이미 설정됨, 확인만)
git remote -v

# 스테이징 & 커밋
git add -A
git status

# 커밋 메시지 확인 후 실행
git commit -m "init: initial commit"

# push
git push -u origin main
```

> `git add -A` 후 `git status`를 출력해서 ReS가 staged 파일 확인할 수 있게 한다.
> 확인 없이 바로 push하지 않는다.

---

## STEP 7: REPORT — 완료 리포트

```
## 🎉 레포 생성 완료

**레포 URL:** https://github.com/{owner}/{repo-name}
**Visibility:** public / private
**Branch protection:** 설정됨 / 미설정

### 생성된 파일
- .gitignore
- README.md
- LICENSE ({종류})

### Doc Consistency
- 결과: PASS / WARN (WARN 항목 있으면 목록)

### 권장 다음 액션
- [ ] README 상세 보완 (설치/사용법 실제 명령 채우기)
- [ ] .env.example 추가 (환경변수 있는 경우)
- [ ] GitHub Actions 워크플로우 설정
- [ ] Collaborator 추가 (협업 시)
- [ ] Notion/포트폴리오 등록
```

---

## 전체 플로우 요약

```
STEP 1: INTAKE       → 경로/이름/설명/visibility 수집 (한 번에)
STEP 2: SCAN         → 스택 감지 + 민감 파일 체크
STEP 3: GATE         → [public만] publish-checklist 실행
STEP 4: SCAFFOLD     → .gitignore / README / LICENSE 생성
                     → ✋ ReS 확인 게이트
STEP 4.5: DOC CHECK  → doc-consistency 실행
STEP 5: CREATE       → gh repo create + topics + branch protection
STEP 6: PUSH         → git add / commit / push
                     → ✋ staged 파일 확인 후 push
STEP 7: REPORT       → 완료 리포트
```

## 중단 조건

아래 상황에서는 **즉시 중단하고 ReS에게 보고**:
- 민감 파일 발견 (STEP 2)
- publish-checklist FAIL (STEP 3)
- doc-consistency FAIL (STEP 4.5)
- `gh repo create` 실패 (이름 중복, 권한 오류 등)
- `git push` 실패 (conflict, 인증 오류 등)

## 주의사항

- **ReS 확인 게이트(STEP 4, STEP 6)는 건너뛰지 않는다.**
- 각 STEP은 순서대로 실행. 앞 STEP FAIL 시 다음 STEP 진행 불가.
- `gh` CLI가 없거나 인증 안 된 경우 먼저 해결 안내.
