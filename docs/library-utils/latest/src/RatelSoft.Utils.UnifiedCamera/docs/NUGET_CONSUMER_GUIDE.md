# UnifiedCamera - NuGet 소비자 가이드

이 문서는 `RatelSoft.Utils.UnifiedCamera`를 NuGet으로 설치한 **소비자 프로젝트**에서 벤더 SDK(Basler/MVS) 어셈블리를 연결하는 방법을 설명합니다.

## 1) 왜 별도 설정이 필요한가?
Basler/MVS 같은 벤더 SDK는 보통
- 버전별 호환성이 완벽하지 않거나,
- 네이티브 런타임/드라이버 설치가 필요하고,
- 라이선스 때문에 NuGet 패키지에 DLL을 포함하기 어려운 경우가 많습니다.

따라서 이 패키지는 기본적으로 벤더 SDK DLL을 **포함하지 않고**, 소비자가 설치한 SDK 경로를 설정하도록 설계되어 있습니다.

## 2) 빠른 시작(권장): 실행 폴더에 DLL 직접 복사
소비자 앱은 Basler/MVS 어셈블리를 직접 호출하지 않는 경우가 많고, 실제로 중요한 것은 **런타임에 DLL이 로딩 가능하냐** 입니다.

가장 단순한 방법은 아래처럼 **실행 파일(.exe) 출력 폴더에 벤더 managed DLL을 직접 복사**하는 것입니다.

- Basler 사용 시: `Basler.Pylon.dll`을 `MyApp.exe` 옆에 복사
- MVS 사용 시: `MvCameraControl.Net.dll`을 `MyApp.exe` 옆에 복사

주의:
- 벤더 SDK에 따라 네이티브 DLL/런타임/드라이버 설치가 추가로 필요할 수 있습니다.
- 어떤 카메라 타입을 실제로 사용할 때 해당 SDK가 필요합니다.

## 3) (선택) 빌드 시 자동 복사: Directory.Build.props
원하면 소비자 솔루션 루트에 `Directory.Build.props`를 만들고 아래 속성을 설정해, 빌드 시 자동으로 Reference를 추가하고 출력 폴더로 복사되게 할 수 있습니다.

- `BaslerPylonManagedFile`: `Basler.Pylon.dll`의 전체 경로
- `MvsManagedFile`: `MvCameraControl.Net.dll`의 전체 경로

예시:

```xml
<Project>
  <PropertyGroup>
    <BaslerPylonManagedFile>C:\Program Files\Basler\pylon\Development\Assemblies\Basler.Pylon.dll</BaslerPylonManagedFile>
    <MvsManagedFile>C:\Program Files\MVS\Development\DotNet\MvCameraControl.Net.dll</MvsManagedFile>
  </PropertyGroup>
</Project>
```

## 4) 문서 폴더(RatelLibDoc)
패키지는 `RatelLibDoc` 문서/템플릿을 포함합니다.

- 기본 동작: 빌드 시 출력 폴더에 `RatelLibDoc\...`를 복사합니다.
- 끄고 싶으면:

```xml
<Project>
  <PropertyGroup>
    <RatelLibDocCopyToOutput>false</RatelLibDocCopyToOutput>
  </PropertyGroup>
</Project>
```

## 5) 주의사항
- `Directory.Build.props` 자동화는 **managed DLL** 경로만 다룹니다. 벤더 SDK에 따라 네이티브 파일이 추가로 필요할 수 있습니다.
- 수동 복사 방식을 선택하더라도, 필요한 런타임 구성(설치/드라이버/환경변수)은 벤더 정책을 따릅니다.
