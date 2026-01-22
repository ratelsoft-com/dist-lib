# RatelSoft.Utils.UnifiedCamera

카메라 공통 모듈입니다.

## 벤더 DLL 제공(권장: 실행 폴더에 직접 복사)
Basler/MVS 같은 벤더 SDK는 네이티브 런타임/드라이버 의존 및 배포 제약 때문에, 이 패키지는 벤더 DLL을 포함하지 않습니다.

따라서 소비자(앱)에서는 **실행 파일(.exe) 출력 폴더에 벤더 managed DLL을 직접 복사**하는 방식이 가장 단순합니다.

- 예: `Basler.Pylon.dll`을 `MyApp.exe` 옆에 복사
- (필요 시) `MvCameraControl.Net.dll`도 동일하게 복사

주의: 벤더 SDK에 따라 네이티브 DLL/런타임 설치가 추가로 필요할 수 있습니다.

## (선택) 빌드 시 자동 복사: Directory.Build.props
원하면 `Directory.Build.props`로 DLL 전체 경로를 지정해, 빌드 시 자동으로 Reference를 추가하고 출력 폴더로 복사되게 할 수 있습니다.

예시(레포/솔루션 루트에 `Directory.Build.props` 생성):

```xml
<Project>
  <PropertyGroup>
    <!-- Basler Pylon managed assembly 전체 경로 -->
    <BaslerPylonManagedFile>C:\\Program Files\\Basler\\pylon\\Development\\Assemblies\\Basler.Pylon.dll</BaslerPylonManagedFile>

    <!-- MVS managed assembly 전체 경로 -->
    <MvsManagedFile>C:\\Program Files\\MVS\\Development\\DotNet\\MvCameraControl.Net.dll</MvsManagedFile>
  </PropertyGroup>
</Project>
```

레포에 포함된 샘플 DLL을 선택하고 싶다면(개발용):

```xml
<Project>
  <PropertyGroup>
    <BaslerPylonManagedFolder>7.3</BaslerPylonManagedFolder>
  </PropertyGroup>
</Project>
```

## 참고
- 벤더 SDK(Basler/MVS) DLL을 포함할 수 있어 `net8.0-windows` 타깃입니다.
- 로컬 테스트/배포 단순화를 위해 내부 의존성(`CameraService.Contracts`)을 별도 프로젝트/별도 DLL로 유지하지 않고,
  소스를 직접 포함해 `UnifiedCamera.dll` 단일 DLL로 빌드되도록 구성되어 있습니다.
