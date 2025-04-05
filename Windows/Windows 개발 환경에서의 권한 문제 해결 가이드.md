# Windows 개발 환경에서의 권한 문제 해결 가이드

## 1. Windows 권한 시스템 이해하기

### 1.1 Windows 권한 시스템 개요
Windows는 NTFS 파일 시스템을 기반으로 복잡한 권한 관리 시스템을 가지고 있습니다. 이 시스템은 보안을 강화하기 위해 설계되었지만, 개발 환경 구성 시 여러 제약을 가져올 수 있습니다.

### 1.2 주요 보호 폴더
Windows에서 특별히 보호되는 폴더들:
- `C:\Program Files\`
- `C:\Program Files (x86)\`
- `C:\Windows\`
- `C:\Users\[사용자명]\AppData\`

이 폴더들은 일반 사용자 권한으로는 쓰기 작업에 제한이 있으며, 관리자 권한으로도 UAC(User Account Control) 승인이 필요합니다.

### 1.3 UAC(User Account Control)
- Windows Vista 이후 도입된 보안 기능
- 관리자 계정이라도 중요 시스템 작업에는 명시적 승인 필요
- "관리자 권한으로 실행" 옵션을 사용해야 UAC 승인을 받을 수 있음

## 2. 개발 환경 설정 시 권한 문제 해결법

### 2.1 애플리케이션 서버(Tomcat, JBoss 등) 설치 위치
**✅ 권장 설치 경로:**
- `C:\Tomcat\` 또는 `D:\servers\Tomcat\`와 같이 루트 드라이브 바로 아래 설치
- `C:\dev\servers\`와 같이 개발용 폴더 구성

**❌ 피해야 할 설치 경로:**
- `C:\Program Files\Apache Tomcat\`
- `C:\Windows\` 하위 폴더

### 2.2 데이터베이스 서버 설정
- 데이터 파일 저장 위치를 `C:\Program Files\` 외부로 지정
- 로그 파일 위치도 쓰기 권한이 있는 경로로 설정
- 서비스 실행 계정에 충분한 권한 부여

### 2.3 IDE 및 개발 도구 설정
- 작업 공간(workspace)을 `C:\Users\[사용자명]\Documents\` 또는 별도 드라이브에 설정
- 빌드 출력 경로도 권한 문제가 없는 위치로 지정
- 가능하면 IDE를 관리자 권한으로 실행

## 3. 권한 문제 진단 및 해결 방법

### 3.1 권한 문제 식별 방법
다음과 같은 오류 메시지는 권한 문제를 나타냅니다:
- "Access denied"
- "권한이 거부되었습니다"
- "Cannot create/write to file"
- "[로그파일명] 파일을 만들 수 없습니다"
- "[컨텍스트명] 컨텍스트 로드에 실패했습니다"

### 3.2 즉각적인 해결 방법
1. **관리자 권한으로 실행**
    - 서버나 애플리케이션을 관리자 권한으로 실행

2. **설치 위치 변경**
    - 애플리케이션을 권한 제한이 적은 경로로 이동

3. **폴더 권한 변경**
   ```
   # 관리자 권한 명령 프롬프트에서
   icacls "C:\Program Files\Apache Tomcat" /grant Users:(OI)(CI)F /T
   ```

### 3.3 서비스로 등록하여 해결
서버를 Windows 서비스로 등록할 경우:
1. 서비스 실행 계정을 LocalSystem 또는 관리자 계정으로 설정
2. '로컬 서비스' 또는 '네트워크 서비스' 계정에 필요한 폴더 권한 부여

예시 (Tomcat 서비스 등록):
```batch
@echo off
set CATALINA_HOME=C:\Tomcat
%CATALINA_HOME%\bin\service.bat install TomcatService
```

## 4. 개발 조직을 위한 모범 사례

### 4.1 표준 개발 환경 구성
표준화된 개발 환경 구성을 문서화하고 팀원들에게 공유:
- 설치 경로 표준화: `C:\dev\` 또는 `D:\dev\`와 같은 별도 경로 사용
- 권한 설정 스크립트 공유
- 환경변수 설정 가이드 제공

### 4.2 권한 관련 문제 해결 문서화
- 발생한 권한 문제와 해결책을 내부 지식베이스에 기록
- 새로운 팀원 온보딩 과정에 권한 관련 교육 포함

### 4.3 가상화 환경 활용
- Docker나 VM을 활용하여 권한 문제를 우회
- 개발 환경의 컨테이너화로 일관성 있는 환경 제공

## 5. 실제 사례 및 해결책

### 5.1 Tomcat 로그 생성 실패 문제
**문제 상황:**
- Tomcat이 `C:\Program Files\Tomcat`에 설치됨
- 서버 시작 시 로그 파일 생성 실패
- "로그백을 만들 수 없습니다" 오류 발생

**해결책:**
1. Tomcat을 `C:\Tomcat`으로 이동
2. 또는 로그 디렉토리 위치를 변경:
   ```xml
   <!-- server.xml 또는 logging.properties에서 -->
   <Host ... appBase="webapps" 
         createDirs="true" 
         workDir="C:/logs/tomcat/work" 
         logsDir="C:/logs/tomcat/logs">
   ```

### 5.2 웹 애플리케이션 컨텍스트 로드 실패
**문제 상황:**
- "컨텍스트 [/애플리케이션명] 로드에 실패했습니다" 오류
- 임시 파일 생성 권한 문제

**해결책:**
1. Tomcat의 실행 계정 변경
2. 또는 임시 디렉토리 위치 변경:
   ```
   # catalina.bat 또는 setenv.bat에 추가
   set CATALINA_TMPDIR=C:\temp\tomcat
   ```

## 6. 추가 참고 자료

- [Windows 파일 시스템 권한 관리](https://docs.microsoft.com/ko-kr/windows-server/administration/windows-commands/icacls)
- [Tomcat 공식 설치 가이드](https://tomcat.apache.org/tomcat-9.0-doc/setup.html)
- [Windows에서 서비스 관리](https://docs.microsoft.com/ko-kr/windows-server/administration/windows-commands/sc-config)
