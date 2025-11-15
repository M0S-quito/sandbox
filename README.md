# 🧰 Codespace 작업 공간 구축 가이드  
### Codespace를 여러 리포를 위한 범용 개발 환경으로 만드는 설정 완전판

---

## 🧩 이 세팅 전체가 뭘 하는 건가?

Codespace는 기본적으로 **제한된 권한의 `GITHUB_TOKEN`**을 자동으로 넣고 시작한다.  
이 토큰은 지금 보고 있는 단일 리포 정도만 접근 가능해서:

- push/pull 때 권한 부족 에러 남  
- organisation 리포 작업 불가  
- 여러 리포를 동시에 다루면 인증 충돌  
- Codespace 재생성하면 인증/토큰 전부 초기화  
- 커밋 사인(GPG/SSH) 꼬임  

이런 좆같은 문제들이 생긴다.

여기서 정리한 세팅은 이 기본 구조를 완전히 뒤집어서,

> **Codespace를 로컬 개발 환경처럼,  
> 내 GitHub 계정 전체 권한을 가진 개발 머신으로 만드는 과정이다.**

결과적으로:

- 모든 리포 push/pull 가능  
- 계정 권한 그대로 사용  
- PAT 기반 안정 인증  
- SSH signing 정상 작동  
- Codespace 재생성돼도 금방 복구  
- 여러 프로젝트 작업해도 안 꼬임  

이런 환경이 된다.

---

# 1. PAT 발급하기  
Codespace의 제한된 기본 토큰을 제거하고  
**내 GitHub 계정 권한을 그대로 쓸 수 있는 토큰(PAT)** 을 만드는 단계다.

```plaintext
GitHub → Settings → Developer settings → Tokens (Classic)
```

- `repo`, `org`, `codespace` 권한만 켜두기  
- fine‑grained는 리포 추가될 때마다 다시 설정해야 해서 불편  
- Classic PAT이 Codespace에서 가장 범용적이고 안정적임

> 결론: Codespace를 진짜 작업 환경으로 쓰려면 Classic PAT이 필수다.

---

# 2. Repo → Secrets에 `MY_PAT` 등록  
Codespace는 리포의 Codespace Secrets를 자동으로 환경 변수로 주입한다.

```plaintext
repo → Settings → Secrets and variables → Codespaces
```

- `MY_PAT`라는 이름으로 아까 만든 PAT 저장

이걸 쓰면 **PAT을 로컬에 저장하지 않고도 Codespace가 안전하게 PAT을 받아온다.**

Secrets → Codespace Env → `$MY_PAT`  
이 흐름으로 전달됨.

---

# 3. Codespace에서 기본 토큰 제거 + PAT 로그인

Codespace는 접속할 때마다 자동으로 `GITHUB_TOKEN`을 넣는다.  
이 토큰이 남아 있는 한, **내 PAT보다 우선순위가 더 높게 작동**함.  
그래서 맨 처음에 반드시 이걸 제거해야 한다.

```bash
unset GITHUB_TOKEN
printf '%s' "$MY_PAT" | gh auth login --with-token
gh auth status
```

### 각 줄의 역할  
- `unset GITHUB_TOKEN`  
  - Codespace 기본 토큰 삭제  
  - 이게 안되면 push/pull·signing 모두 기본 토큰으로 인증돼서 권한 꼬임

- `gh auth login --with-token`  
  - GitHub CLI(gh)를 **내 계정 = 내 PAT**으로 로그인

- `gh auth status`  
  - 진짜로 PAT 기반 로그인인지 최종 확인

이 순간부터 Codespace는 더 이상 “제한된 계정”이 아니라  
**“M0S-quito 계정 자체로 작동하는 머신”** 이 된다.

---

# 4. Git 인증을 GH CLI로 완전 강제하기  
Git은 기본적으로 OS credential helper, 기본 토큰 등  
여러 인증 소스를 섞어서 사용한다.

이걸 완전히 차단하고,

> **모든 Git 인증은 gh CLI가 담당한다.  
> gh는 이미 내 PAT으로 로그인해 있으므로 무조건 통일된다.**

이렇게 만드는 설정이다.

→ 작업공간의 루트(repo root)에서 실행:

```bash
gh auth setup-git
```

그러면 `.gitconfig`에 아래가 자동 삽입된다:

```ini
[credential]
  helper = !gh auth git-credential
```

### 이 설정의 의미  
- Git이 push/pull 요청  
→ gh가 credential 요청 받아 처리  
→ gh는 PAT 기반으로 인증  
→ 인증 충돌 없음  
→ Codespace 재생성돼도 명령 한 줄이면 복구  

쓰는 이유가 너무 명확하다.

만약 자동으로 안 들어갔다면 `.gitconfig`에 직접 추가하면 된다.

---

# 5. .gitignore로 Codespace 작업 구조 보호  
너는 Codespace 안에서 여러 프로젝트를 동시에 굴리기 때문에  
루트 repo에 프로젝트 폴더들을 그대로 두면 Git이 전부 추적해버린다.

그래서 프로젝트 폴더를 미리 ignore 시켜 둔다:

```gitignore
# projects
프로젝트이름/
```

이러면 Codespace 작업 환경이 더러워지지 않는다.

---

# ⚠️ 사용시 주의

### Codespace는 접속 시마다 `GITHUB_TOKEN`을 재발급한다.  
그래서 매 접속 후 아래 한 줄이 필수다.

```bash
unset GITHUB_TOKEN && gh auth setup-git
```

- 기본 토큰을 제거하고  
- gh 기반 인증 체계를 재등록하는 명령

이 한 줄이면 Codespace 초기화가 와도 바로 복구된다.

---

# 📌 전체 구조 요약

```
[기본 Codespace]
    ↓ (GITHUB_TOKEN = 제한적)
권한 부족 / 여러 리포 push 안 됨 / 서명 꼬임

[내 세팅 적용]
    ↓
GITHUB_TOKEN 제거
PAT 기반 gh 로그인
Git credential helper = gh로 통일
프로젝트 폴더는 ignore로 분리

[결과]
Codespace = 로컬 개발환경 수준의 GitHub 권한
push/pull 충돌 없음
SSH 서명/인증 정상
여러 리포 작업 가능
재생성되어도 한 줄로 복구
```