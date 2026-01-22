# RatelSoft.Utils.Common

UI/플랫폼 무관 공통 유틸 패키지입니다.

## 포함 모듈
- Configuration: `Ratel.Configuration` 네임스페이스의 `ConfigManager<T>`, `DataStore<T>`
- Networking: `RatelSoft.Utils.Common.Networking` 네임스페이스
- Logging(Core): `RatelSoft.Utils.Common.Logging` 네임스페이스 (LibLog)

## DataStore - Mat(OpenCv) 의존성
`DataStore`는 기본 JSON 옵션에 Mat 컨버터를 포함하지 않습니다.
Mat을 JSON으로 저장하려면 `RatelSoft.Utils.OpenCvAdapter`를 참조한 뒤 Json 옵션에 컨버터를 추가하세요.
