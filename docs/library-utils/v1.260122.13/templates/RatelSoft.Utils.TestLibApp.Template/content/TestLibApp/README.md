# TestLibApp (Template)

이 프로젝트는 `RatelSoft.Utils.*` NuGet 패키지들을 빠르게 검증하기 위한 WPF 테스트 앱 샘플입니다.

## 생성 방법

1) 템플릿 설치

- `dotnet new install RatelSoft.Utils.TestLibApp.Template`

2) 프로젝트 생성

- `dotnet new ratel-testlibapp -n MyTestLibApp`

3) 빌드/실행

- `dotnet build`
- `dotnet run`

## 벤더 SDK DLL 경로 설정

이 샘플은 벤더 SDK DLL을 패키지에 포함하지 않습니다.

### 권장(가장 단순): 실행 폴더에 직접 복사
- Basler 사용 시: `Basler.Pylon.dll`을 실행 파일(.exe) 옆에 복사
- MVS 사용 시: `MvCameraControl.Net.dll`을 실행 파일(.exe) 옆에 복사

주의: 벤더 SDK에 따라 네이티브 런타임/드라이버 설치가 추가로 필요할 수 있습니다.

### (선택) 빌드 시 자동 복사: Directory.Build.props
원하면 생성된 `Directory.Build.props`에서 아래 속성으로 managed DLL 전체 경로를 지정해, 빌드 시 자동으로 출력 폴더로 복사되게 할 수 있습니다.

- `BaslerPylonManagedFile`
- `MvsManagedFile`
