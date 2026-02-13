# RatelSoft Dist Library

RatelSoft 배포 라이브러리 문서/가이드 저장소입니다.  
NuGet 패키지와 소비자 문서를 함께 관리하며, 패키지 메타데이터의 Repository URL도 이 저장소를 기준으로 연결합니다.

## Package Feed

- GitHub Packages (NuGet): `https://nuget.pkg.github.com/ratelsoft-com/index.json`
- 패키지 소스 키 예시: `ratelsoft`
- 인증 방식: package-read 토큰 기반

## NuGet.Config
- 프로젝트에 NuGet.Config 파일을 만들어서 사용
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
    <add key="ratelsoft" value="https://nuget.pkg.github.com/ratelsoft-com/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <ratelsoft>
      <add key="Username" value="ratelsoft" />
      <add key="ClearTextPassword" value="<package-read-token>" />
    </ratelsoft>
  </packageSourceCredentials>
</configuration>
```

## 라이브러리 목록

아래는 현재 배포/연동 대상의 핵심 라이브러리입니다.

- `RatelSoft.Common`
  - 공통 유틸리티, 로깅, 설정/네트워킹 기반 기능
- `RatelSoft.Lib`
  - 상위 비즈니스 공용 라이브러리, Script/Flow/인증/런타임 공통 기능
- `RatelSoft.Types`
  - 공통 타입/계약(DTO, Enum 등) 성격의 기반 타입 모음
- `RatelSoft.Vision`
  - 검사/비전 알고리즘 및 처리 파이프라인
- `RatelSoft.Vision.Wpf`
  - Vision 영역의 WPF UI/Viewer/도구
- `RatelSoft.WPF` (`RatelLib.WPF`)
  - WPF 기반 공용 구성 요소
- `RatelSoft.Utils.Wpf`
  - WPF 유틸리티/바인딩/헬퍼 기능
- `RatelSoft.Utils.UnifiedIO`
  - 장비/신호 I/O 통합 추상화
- `RatelSoft.Utils.UnifiedCamera`
  - 카메라 통합 제어 추상화
- `RatelSoft.Utils.UnifiedMotion`
  - 모션 제어 통합 추상화(가상/Ajin/PowerPMac 등)
- `RatelSoft.Utils.OpenCvAdapter`
  - OpenCV 연동 보조/어댑터 계층
- `RatelSoft.Utils.TestLibApp.Template`
  - 라이브러리 소비자용 테스트 앱 템플릿

## 라이브러리 관계

대표 의존 관계(단순화):

```text
RatelSoft.Common
  ├─ RatelSoft.Lib
  │   ├─ RatelSoft.Utils.Wpf
  │   ├─ RatelSoft.WPF
  │   └─ RatelSoft.Vision
  │       └─ RatelSoft.Vision.Wpf
  ├─ RatelSoft.Utils.OpenCvAdapter
  ├─ RatelSoft.Utils.UnifiedIO
  └─ RatelSoft.Utils.UnifiedCamera

RatelSoft.Utils.UnifiedMotion
  └─ RatelSoft.Common

RatelSoft.Types
  └─ RatelSoft.Vision
```

## 문서 링크

`docs/` 폴더의 주요 문서:

- 실전 시작 가이드: [`docs/README_PRACTICAL.md`](./docs/README_PRACTICAL.md)
- Script v3 매뉴얼: [`docs/SCRIPT_V3_MANUAL.md`](./docs/SCRIPT_V3_MANUAL.md)
- Script 언어 방향(JS 전환): [`docs/SCRIPT_V3_LANGUAGE_DIRECTION.md`](./docs/SCRIPT_V3_LANGUAGE_DIRECTION.md)
- Script JS 빠른 시작: [`docs/SCRIPT_JS_QUICKSTART.md`](./docs/SCRIPT_JS_QUICKSTART.md)
- Unified 사용자 디바이스 확장 가이드: [`docs/UNIFIED_CUSTOM_DEVICE_GUIDE.md`](./docs/UNIFIED_CUSTOM_DEVICE_GUIDE.md)
- 통합 코드북: [`docs/RatelSoft_CODEBOOK.md`](./docs/RatelSoft_CODEBOOK.md)
- RatelViewer 코드북: [`docs/RatelSoft_CODEBOOK_RatelViewer.md`](./docs/RatelSoft_CODEBOOK_RatelViewer.md)
- EsViewer 실전 코드북: [`docs/RatelSoft_CODEBOOK_EsViewer_PRACTICAL.md`](./docs/RatelSoft_CODEBOOK_EsViewer_PRACTICAL.md)
- 타입 인덱스: [`docs/RatelSoft_TYPE_INDEX.md`](./docs/RatelSoft_TYPE_INDEX.md)
- AI 소비자 가이드: [`docs/RatelSoft_AI_CONSUMER_GUIDE.md`](./docs/RatelSoft_AI_CONSUMER_GUIDE.md)
- 로깅 시스템: [`docs/logging-system.md`](./docs/logging-system.md)
- 다국어/리소스: [`docs/Localization.md`](./docs/Localization.md)
- 문서 허브: [`docs/README.md`](./docs/README.md)

