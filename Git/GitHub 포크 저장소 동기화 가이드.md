## 1. Git의 'upstream' 개념 이해하기

Git에서 원격 저장소(remote repository)는 네트워크나 인터넷을 통해 접근할 수 있는 프로젝트의 버전이다. 

Git을 사용할 때는 일반적으로 다음과 같은 원격 저장소 이름을 사용한다.

- **origin**: 내가 클론한 원격 저장소의 기본 이름
- **upstream**: 내 포크(fork)의 원본이 되는 저장소

예를 들어:

- `https://github.com/beurre-noisette/Server` (origin)
- `https://github.com/CokeZet/CokeZet_Server` (upstream)

'upstream'은 Git의 표준 용어는 아니지만, 포크된 저장소의 원본을 참조하는 관례적인 이름이라고 한다.

## 2. 포크 저장소 동기화 전체 과정

### 2.1 최초 설정 (한 번만 필요)

1. **포크한 저장소 클론하기**

    ```bash
    git clone https://github.com/YOUR_USERNAME/YOUR_FORK.git
    cd YOUR_FORK
    ```

2. **원본 저장소를 'upstream'으로 추가하기**

    ```bash
    git remote add upstream https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git
    ```

3. **원격 저장소 설정 확인하기**

    ```bash
    git remote -v
    ```

결과:

```
origin    https://github.com/YOUR_USERNAME/YOUR_FORK.git (fetch)
origin    https://github.com/YOUR_USERNAME/YOUR_FORK.git (push)
upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (fetch)
upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (push)
```


### 2.2 동기화 과정 (업데이트가 필요할 때마다 반복)

1. **원본 저장소의 최신 변경사항 가져오기**

    ```bash
    git fetch upstream
    ```

2. **로컬 메인 브랜치로 이동하기**

    ```bash
    git checkout main    # 또는 master 등 메인 브랜치 이름 사용
    ```

3. **원본 저장소의 변경사항 병합하기**

    ```bash
    git merge upstream/main
    ```

    - 병합 충돌이 발생하면 해결 필요
    - 병합 커밋 메시지 작성 필요
4. **포크 저장소에 푸시하기**

    ```bash
    git push origin main
    ```


## 3. 주요 명령어 상세 설명

### git remote add

```bash
git remote add <이름> <URL>
```

이 명령어는 지정한 URL의 원격 저장소를 특정 이름으로 추가하고, `upstream`을 추가할 때 사용한다.

### git fetch

```bash
git fetch <원격 저장소 이름>
```

원격 저장소의 모든 브랜치와 데이터를 가져오지만, 로컬 브랜치에 자동으로 병합하지는 않는다. 원본 저장소의 변경사항을 확인하기 위해 사용한다.

### git merge

```bash
git merge <브랜치 이름>
```

지정한 브랜치의 변경사항을 현재 브랜치에 병합한다. `upstream/main`의 변경사항을 로컬 `main` 브랜치에 병합할 때 사용한다.

## 4. 인텔리제이에서 작업하기

### 4.1 터미널 열기

1. 메뉴에서 `View` → `Tool Windows` → `Terminal` 선택
2. 프로젝트 루트 디렉토리에서 터미널이 열립니다

### 4.2 GUI로 푸시하기

1. 메뉴에서 `Git` → `Push` 선택
2. 푸시할 변경사항 확인 후 `Push` 버튼 클릭

### 4.3 병합 충돌 해결하기

1. 인텔리제이가 자동으로 충돌 해결 도구를 표시
2. 각 충돌에 대해 `Accept Yours`, `Accept Theirs` 또는 수동 편집 선택
3. 모든 충돌 해결 후 `Git` → `Commit` 선택하여 병합 완료

## 5. 자주 발생하는 문제와 해결 방법

### 5.1 병합 충돌

- **문제**: 원본 저장소와 포크 저장소에서 같은 파일의 같은 부분이 다르게 수정됨
- **해결**: 충돌된 파일을 수동으로 편집하여 어떤 변경사항을 유지할지 결정

### 5.2 권한 문제

- **문제**: `fetch upstream` 시 권한 오류 발생
- **해결**: 원본 저장소가 비공개인 경우, 접근 권한을 요청하거나 원본 소유자에게 문의

### 5.3 브랜치 이름 불일치

- **문제**: 원본 저장소와 포크 저장소의 기본 브랜치 이름이 다름
- **해결**: 정확한 브랜치 이름 확인 후 명령어 수정

    ```bash
    git checkout main    # 또는 mastergit merge upstream/main    # 또는 upstream/master
    ```


## 6. 용어 정리

- **Fork(포크)**: 다른 사람의 GitHub 저장소를 자신의 GitHub 계정으로 복사하는 것
- **Clone(클론)**: 원격 저장소를 로컬 컴퓨터로 복사하는 것
- **Remote(원격 저장소)**: 인터넷이나 네트워크에 호스팅된 프로젝트의 버전
- **Upstream**: 포크의 원본 저장소
- **Origin**: 클론한 원격 저장소의 기본 이름
- **Fetch**: 원격 저장소의 데이터를 가져오는 것
- **Merge**: 한 브랜치의 변경사항을 다른 브랜치에 통합하는 것
- **Push**: 로컬 변경사항을 원격 저장소에 업로드하는 것

## 7. 참고 자료

- [GitHub 공식 문서: 포크 동기화](https://docs.github.com/en/github/collaborating-with-pull-requests/working-with-forks/syncing-a-fork)
- [Pro Git Book: 원격 저장소 관리](https://git-scm.com/book/ko/v2/Git%EC%9D%98-%EA%B8%B0%EC%B4%88-%EB%A6%AC%EB%AA%A8%ED%8A%B8-%EC%A0%80%EC%9E%A5%EC%86%8C)
- [IntelliJ IDEA 문서: Git 사용하기](https://www.jetbrains.com/help/idea/using-git-integration.html)