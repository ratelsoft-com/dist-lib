# RatelSoft.Utils.TestLibApp.Template

`dotnet new`로 생성하는 WPF 샘플 프로젝트 템플릿입니다.

- 목적: `RatelSoft.Utils.*`(Common/Wpf/UnifiedCamera/UnifiedIO) 패키지 설치/동작을 빠르게 검증
- 포함: `ViewModels/`, `Views/`, `Networking/` 등 실제 샘플 소스 + 메뉴(MainWindow)

## 설치

```powershell
dotnet new install RatelSoft.Utils.TestLibApp.Template
```

## 프로젝트 생성

```powershell
dotnet new ratel-testlibapp -n MyTestLibApp
cd MyTestLibApp
```

## 빌드/실행

```powershell
dotnet build
# 또는
dotnet run
```

## 패키지 버전/소스 설정

생성된 프로젝트의 `Directory.Build.props`에서 버전을 제어합니다.

- 기본값: `RatelSoftUtilsVersion` = `1.*`
- 사내/로컬 NuGet 피드를 쓴다면 `NuGet.Config`로 소스를 추가하세요.

## 벤더 SDK DLL 경로 설정(Basler/MVS)

Basler/MVS를 실제로 테스트할 때는 벤더 SDK DLL을 **실행 폴더(출력 폴더)에 직접 복사**하는 방식이 가장 단순합니다.

- Basler: `Basler.Pylon.dll`을 실행 파일(.exe) 옆에 복사
- MVS: `MvCameraControl.Net.dll`을 실행 파일(.exe) 옆에 복사

주의: 벤더 SDK에 따라 네이티브 런타임/드라이버 설치가 추가로 필요할 수 있습니다.

### (선택) 빌드 시 자동 복사
원하면 `Directory.Build.props`에서 managed DLL 경로를 지정해, 빌드 시 자동으로 출력 폴더로 복사되게 할 수 있습니다.

- `BaslerPylonManagedFile`
- `MvsManagedFile`

개발자 로컬 전용 설정이 필요하면 `TestLibApp.Local.props.template`을 복사해 `TestLibApp.Local.props`로 저장한 뒤 경로를 채우면 됩니다.

## 참고

- 이 템플릿은 생성된 프로젝트에서 기본적으로 NuGet 패키지 참조로 동작합니다.
- (library-utils 레포 내부에서 빌드하는 경우에는) 템플릿 content의 csproj가 자동으로 `ProjectReference`를 사용하도록 구성되어 있어,
  패키지 퍼블리시 없이도 샘플 빌드 검증이 가능합니다.
