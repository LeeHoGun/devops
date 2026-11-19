# 12주차: 깃허브 액션 (GitHub Actions) CI

## 1. 실습 목표
- 테스트 코드와 워크플로를 작성해 push/PR 시 린트·테스트·이미지 빌드를 자동 실행
- main 머지 시 GHCR(ghcr.io)에 이미지를 자동 게시하고 쿠버네티스에 반영
- 브랜치 보호 규칙으로 검사를 통과하지 못한 코드가 main에 합쳐지지 않도록 설정

## 2. 진행 내용

### 실습 1-1. 테스트 코드 작성
- `guestbook/tests/test_app.py` 추가 (unittest)
  - `test_health`: `/healthz`가 200과 `status: ok`를 반환하는지 확인
  - `test_add_and_read`: 글 등록 후 302 응답, 조회 결과 글 1개·내용 `hello` 확인
- 푸시 전 컨테이너(python:3.12-slim)에서 `ruff check` + `unittest` 실행 → 2개 테스트 통과

### 실습 1-2. 워크플로 작성 (`.github/workflows/build.yml`)
- 트리거: main push(guestbook/**, build.yml 변경 시), main 대상 PR, 수동 실행(workflow_dispatch)
- 단계: 체크아웃 → Python 3.12 준비 → 린트(ruff) → 테스트 → QEMU/Buildx → ghcr.io 로그인 → 태그 결정 → 빌드 & 푸시
- PR에서는 빌드만, main에서만 이미지 푸시 (태그: 커밋 해시 7자리 + latest)
- `feature/ci-setup` 브랜치 → PR → 스쿼시 머지

### 실습 1-3. 결과 확인
- Actions 탭에서 PR 실행(✓)과 main 실행(✓) 확인
- GHCR에 `ghcr.io/leehogun/guestbook:4af9057`, `latest` 게시 확인
- `kubectl set image deployment/guestbook web=ghcr.io/leehogun/guestbook:4af9057`로 반영 → 브라우저에서 제목이 `new`로 바뀐 것 확인

### 실습 2. 실패 → 수정 → 성공 / 브랜치 보호
- 일부러 오류를 넣어 빨간 X 확인 후, 로그를 아래에서 위로 읽어 원인 파악·수정 → 초록 ✓
- 룰세트로 main 보호: PR 필수 + `build` 검사 통과 필수
- `gh api .../rules/branches/main` 결과 `pull_request`, `required_status_checks` 확인
- main에 직접 push 시 `remote rejected` 확인
- 깨진 코드 PR은 머지 버튼 비활성화, 수정 후 ✓ 되면 머지 가능

## 3. 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| PR에 `no checks reported` | build.yml에서 `pull_request:`가 `push:` 안으로 들여쓰기됨, `env:` 위치 오류, step 분리(`- uses:`), `linux.amd64` 오타 | 들여쓰기·오타 수정 후 다시 push |
| main 빌드 & 푸시에서 `denied: permission_denied: write_package` | 6주차에 CLI로 만든 guestbook 패키지가 devops 저장소와 연결되지 않음 | Package settings → Manage Actions access → devops 추가, Role: Write → Re-run |
| `minikube start` 실패 (`PROVIDER_DOCKER_VERSION_EXIT_1`) | WSL에서 Docker 데몬에 접속 불가 | Docker Desktop 실행 후 다시 시작 |
| `gh pr check --watch` 오류 | 명령어 오타 | `gh pr checks --watch` |

## 4. 배운 점
- CI는 빌드 자체보다 **main 브랜치를 지키는 것**이 핵심 (브랜치 보호 + 상태 검사)
- 단계의 성공/실패는 출력 내용이 아니라 **종료 코드(0 = 성공)** 로 판단
- 문법 검사 → 린트 → 테스트 순으로 싸고 빠른 검사부터 실행해 실패를 일찍 발견
- `latest`만 쓰면 이전 버전 추적이 어려우므로 커밋 해시 태그 사용
- 실패 로그는 맨 아래에서 위로 올라가며 첫 번째 오류 줄과 파일:줄 번호를 찾기
- 다음 주: ArgoCD로 `kubectl set image` 수동 반영을 자동화(CD/GitOps)
