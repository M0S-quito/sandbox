# codespcae 작업 공간

### codesapce에서 여러 작업이 가능한 유틸 작업 환경 만들기
##### 01. PAT발급하기
```plaintext
github -> setting -> Developer settings -> Tokens(Classic)
```
- *repo설정*, *org설정*, *codespace설정* 만 켜두기
> 원래는 `fine-grainde`로 해도 되긴 하는데 이러면 새로운 리포가 들어 오면 다시 설정해 줘야 해서 불편 조금 위험해도 클래식으로 설정

##### 02. 내가 작업 공간으로 할 리포에들어가 *secrets* 환경 변수 설정
```plaintext
repo -> setting -> Secrets and variables -> codespace
```
- `MY_PAT`라는 환경 변수 설정

##### 03. codespace에서 내가 발급한 토큰으로 변경
```bash
unset GITHUB_TOKEN
printf '%s' "$MY_PAT" | gh auth login --with-token
gh auth status
```
-  마지막 줄에서 내가 설정한 토큰에 권한이 확인 되면 성공!

##### 04. git이 pull/push할때 CLI를 통해 안전하게 인증하도록 변경
-> ***내가 작업 환경으로 사용할 root가 될 리포에서 진행***
```bash
gh auth setup-git
```
- 첫번째 줄이 *.gitconfig*에 `helper = !gh auth git-credential`구문을 추가 시켜줌
- 해봤는데 안된다면 *.gitconfig*에다가 직접 아래 구문 넣어 보기
```.gitconfig
[credential]
  helper = !gh auth git-credential
```

##### 05. .gitignore로 프로젝트들이 작업 환경과 같이 git에 올라가지 않게 하기
```.gitignore
# projects
리포 이름/
```
