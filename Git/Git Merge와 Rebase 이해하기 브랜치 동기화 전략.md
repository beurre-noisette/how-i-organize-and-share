# Git Merge와 Rebase 이해하기: 브랜치 동기화 전략

## 1. 문제 상황

### 초기 상태

```
                        [PR 및 Merge]
                        ↗           ↘
[로컬 main] → [remote/refactor/android-login] → [remote/main]
     ↓                                              ↓
     └→ [로컬 feature/exception-handling]   [로컬 main을 pull로 업데이트]
        (아직 변경사항 없음)
```

### 상세 설명

1. 로컬 main에서 `feature/exception-handling` 브랜치 생성 (아직 작업 시작 안 함)
2. 로컬 main에서 `refactor/android-login` 브랜치 생성
3. `refactor/android-login` 브랜치에서 소셜 로그인 리팩토링 작업:
    - `deviceType` → `provider` 변경
    - Factory 패턴 도입
4. 리팩토링 브랜치가 PR을 통해 remote main에 병합됨
5. 로컬 main이 원격 main과 동기화됨 (`git pull origin main`)

### 해결해야 할 문제

**로컬의 `feature/exception-handling` 브랜치를 현재 main의 최신 코드(리팩토링된)와 어떻게 동기화할 것인가?**

## 2. 가능한 해결 방법들

### 방법 1: Git Merge 사용

```bash
git checkout feature/exception-handling
git merge main
```

### 방법 2: Git Rebase 사용

```bash
git checkout feature/exception-handling
git rebase main
```

### 방법 3: 브랜치 삭제 후 재생성

```bash
git checkout main
git branch -D feature/exception-handling
git checkout -b feature/exception-handling
```

## 3. 각 방법의 장단점과 수행 방법

### Merge 방식

#### 시각적 표현

```
      A---B---C (feature)
     /         \
D---E---F---G---H (main)
```

#### 장점
- **안전성**: 원본 커밋 히스토리가 변경되지 않음
- **추적성**: 브랜치가 언제, 어디서 분기되었는지 명확히 볼 수 있음
- **공유 브랜치에 적합**: 이미 원격에 push된 브랜치에도 안전하게 사용 가능

#### 단점
- **복잡한 히스토리**: 많은 병합이 있을 경우 히스토리가 복잡해짐
- **병합 커밋 생성**: 추가적인 병합 커밋이 생성됨

#### 수행 방법

```bash
git checkout feature/exception-handling  # 대상 브랜치로 이동
git merge main                          # main의 변경사항을 병합
# 충돌이 발생하면 해결 후
git add .                               # 충돌 해결 파일 추가
git commit -m "Merge main into feature" # 병합 커밋 생성
```

### Rebase 방식

#### 시각적 표현

```
              A'--B'--C' (feature)
             /
D---E---F---G (main)
```

#### 장점
- **깔끔한 히스토리**: 선형적인 커밋 히스토리 생성
- **불필요한 병합 커밋 없음**: 병합 커밋이 생성되지 않음
- **코드 리뷰 용이**: 변경 사항을 더 명확하게 파악 가능

#### 단점
- **원본 커밋 변경**: 기존 커밋 해시가 변경됨
- **공유 브랜치에 위험**: 이미 push된 브랜치에 사용 시 force push 필요
- **충돌 해결 복잡**: 여러 커밋에 걸쳐 충돌 발생 가능

#### 수행 방법

```bash
git checkout feature/exception-handling  # 대상 브랜치로 이동
git rebase main                         # main 위에 feature 커밋 재배치
# 충돌이 발생하면 해결 후
git add .                               # 충돌 해결 파일 추가
git rebase --continue                   # rebase 계속 진행
# 원격 브랜치가 있다면 강제 push 필요
git push -f origin feature/exception-handling
```

### 브랜치 삭제 후 재생성

#### 장점
- **가장 단순한 방법**: 복잡한 Git 명령어 불필요
- **충돌 가능성 없음**: 항상 깨끗한 시작점을 얻음
- **실수 방지**: 잘못된 베이스 코드에서 작업할 가능성 제거

#### 단점
- **이력 손실**: 브랜치 생성 시점 메타데이터 손실
- **기존 작업 손실**: 브랜치에 이미 작업이 있다면 사용 불가

#### 수행 방법

```bash
git checkout main                      # main 브랜치로 이동
git pull origin main                   # 최신 main 상태 확인
git branch -D feature/exception-handling # 기존 브랜치 삭제
git checkout -b feature/exception-handling # 새 브랜치 생성
```

## 4. 우리가 선택한 해결 방법과 이유

### 선택한 방법: 브랜치 삭제 후 재생성

이 방법을 선택한 이유:

1. **작업이 없었음**: feature/exception-handling 브랜치에 아직 작업을 시작하지 않았음
2. **단순함**: 가장 간단하고 직관적인 방법
3. **오류 가능성 최소화**: 복잡한 Git 작업 없이 깨끗한 시작점 확보
4. **결과 예측 가능**: 결과가 명확하고 예측 가능함

제시된 시나리오에서는 작업이 없는 브랜치였기 때문에, 복잡한 merge나 rebase 과정보다 간단하게 브랜치를 재생성하는 것이 가장 효율적인 선택이었습니다.

## 5. Git Merge와 Rebase의 심층 이해

### Merge와 Rebase의 근본적 차이

|특성|Merge|Rebase|
|---|---|---|
|히스토리|비선형적, 분기 보존|선형적, 분기 제거|
|커밋 변경|원본 커밋 유지|새 커밋 생성 (SHA 변경)|
|충돌 해결|한 번에 해결|커밋별 단계적 해결|
|적합한 상황|공유 브랜치, 팀 협업|개인 브랜치, 코드 정리|

### Rebase 작동 방식 설명

Rebase는 "현재 브랜치의 변경사항을 다른 브랜치 위에 다시 적용"하는 과정입니다.

```
# 시작 상태
      A---B---C (feature)
     /
D---E---F---G (main)

# git checkout feature; git rebase main의 내부 동작
1. 공통 조상 찾기 (E)
2. feature 브랜치의 변경사항 임시 저장 (A, B, C)
3. feature 브랜치를 main의 끝으로 이동
4. 저장된 변경사항을 하나씩 재적용 (A', B', C')
```

### 일반적인 사용 지침

1. **Merge 사용 권장 상황**:
    - 이미 공유된(push된) 브랜치
    - 브랜치 히스토리를 보존해야 할 때
    - 팀원들과 협업 중인 브랜치
2. **Rebase 사용 권장 상황**:
    - 로컬에서만 작업 중인 개인 브랜치
    - 최신 변경사항을 feature에 적용할 때
    - 깔끔한 커밋 히스토리를 원할 때
3. **브랜치 재생성 권장 상황**:
    - 작업을 시작하지 않은 브랜치
    - 완전히 새로운 시작점이 필요할 때
    - 이전 커밋 히스토리가 중요하지 않을 때

### 주의사항

1. **Golden Rule of Rebase**:
   > "이미 공개 저장소에 push한 커밋은 절대 rebase하지 마라."
2. **Force Push 위험성**:
    - 팀 협업 시 다른 개발자의 작업에 영향을 줄 수 있음
    - 사용 전 항상 팀과 협의 필요
3. **Merge vs Rebase는 의견 차이**:
    - 팀/프로젝트마다 선호하는 전략이 다를 수 있음
    - 일관된 방식으로 사용하는 것이 중요