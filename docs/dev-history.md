# Soniox Captions OBS Plugin - Development History

점진적 개발 기록. C++ 비경험자가 OBS 플러그인을 처음부터 만들어가는 과정.

---

## Step 1: 껍데기 플러그인 (Load/Unload)

**목표**: OBS에 플러그인이 등록되고, Plugin Manager에서 보이는 것

### 무엇을 했나
1. obs-plugintemplate (GitHub 공식 템플릿) 복사
2. `buildspec.json` 수정 — 이름을 `soniox-caption-obs`로 변경
3. `src/plugin-main.c` — 최소 코드만 유지

### 핵심 코드
```c
#include <obs-module.h>

OBS_DECLARE_MODULE()  // OBS 플러그인 필수 매크로

bool obs_module_load(void) {     // OBS 시작 시 호출
    return true;                  // true = 로드 성공
}

void obs_module_unload(void) {}  // OBS 종료 시 호출
```

### 배운 것
- OBS 플러그인은 **3가지 필수 요소**만 있으면 됨: `OBS_DECLARE_MODULE`, `obs_module_load`, `obs_module_unload`
- macOS에서 플러그인은 `.plugin` 번들 (폴더) 형태
- 설치 경로: `~/Library/Application Support/obs-studio/plugins/`
- 제거: 해당 폴더 삭제하면 끝
- CMake `--preset macos`로 configure → `--build --preset macos`로 빌드

### 빌드 명령어
```bash
cmake --preset macos              # OBS SDK 자동 다운로드 + 프로젝트 설정
cmake --build --preset macos      # 컴파일

# 설치
cp -R build_macos/RelWithDebInfo/soniox-caption-obs.plugin \
  ~/Library/Application\ Support/obs-studio/plugins/
```

### 트러블슈팅
- **CMake 미설치**: `brew install cmake`로 해결
- **PLUGIN_BUILD_NUMBER 빈값 에러**: `cmake/.CMakeBuildNumber` 파일에 `1` 기록하여 해결
  - 원인: CI 환경이 아니면 빌드넘버가 설정되지 않는 템플릿 버그

### 결과
Plugin Manager에서 `soniox-caption-obs` 체크됨 확인 ✓

---

## Step 2: OBS Source 등록 (화면에 텍스트 표시)

**목표**: Sources > + 에서 "Soniox Captions" 선택 가능, 화면에 고정 텍스트 표시

### 무엇을 했나
1. `plugin-main.c` → `plugin-main.cpp`로 전환 (나중에 C++ 필요하므로 미리 전환)
2. `CMakeLists.txt`에서 소스 파일명 변경
3. `obs_source_info` 구조체로 소스 등록
4. OBS 내장 텍스트 소스(`text_ft2_source_v2`)를 내부적으로 사용하여 텍스트 렌더링

### 핵심 개념: OBS Source 등록 패턴
```
obs_source_info 구조체 정의
  ├── id           — 고유 식별자
  ├── type         — OBS_SOURCE_TYPE_INPUT (입력 소스)
  ├── output_flags — OBS_SOURCE_VIDEO (영상 출력)
  ├── get_name     — 메뉴에 보이는 이름
  ├── create       — 소스 생성 시 호출
  ├── destroy      — 소스 삭제 시 호출
  ├── get_width    — 가로 크기
  ├── get_height   — 세로 크기
  └── video_render — 매 프레임 렌더링
```

### 핵심 코드: 내장 텍스트 소스 빌려쓰기
```cpp
// 직접 폰트 렌더링하지 않고, OBS의 텍스트 소스를 내부적으로 생성
obs_data_t *text_settings = obs_data_create();
obs_data_set_string(text_settings, "text", "Soniox Captions Ready!");
obs_data_set_int(text_settings, "font_size", 48);

// obs_source_create_private = 사용자에게 보이지 않는 내부 소스
data->text_source = obs_source_create_private(
    "text_ft2_source_v2",   // macOS/Linux용 텍스트 렌더러
    "soniox_text",          // 내부 이름
    text_settings
);
```

### 배운 것
- OBS 소스 = 콜백 함수 모음. OBS가 적절한 시점에 호출해 줌
- `obs_source_create_private()` — OBS 내장 기능을 "빌려서" 사용 가능 (텍스트 렌더링을 직접 구현할 필요 없음)
- macOS: `text_ft2_source_v2`, Windows: `text_gdiplus` (OS별 텍스트 렌더러 다름)
- `.c` → `.cpp` 전환 시 `rm -rf build_macos && cmake --preset macos` 필요 (Xcode 캐시 문제)

### 트러블슈팅
- **.c → .cpp 전환 후 빌드 실패**: Xcode가 이전 .c 파일을 캐시. `build_macos/` 삭제 후 re-configure로 해결

### 결과
OBS 화면에 "Soniox Captions Ready!" 텍스트 표시됨 ✓

---

## Step 3: Properties UI (설정 화면)

**목표**: 소스 우클릭 > Properties에서 API Key, 언어, 폰트 크기 설정 가능

### 무엇을 했나
1. `get_properties` 콜백 — Properties UI 정의
2. `get_defaults` 콜백 — 기본값 설정
3. `update` 콜백 — 설정 변경 시 텍스트 업데이트

### 핵심 개념: OBS Properties API
```
obs_properties_create()           — Properties 패널 생성
  ├── obs_properties_add_text()   — 텍스트 입력란 (OBS_TEXT_PASSWORD = 마스킹)
  ├── obs_properties_add_list()   — 드롭다운 목록
  └── obs_properties_add_int_slider() — 숫자 슬라이더
```

### 핵심 코드
```cpp
// Properties UI 정의
static obs_properties_t *soniox_caption_get_properties(void *) {
    obs_properties_t *props = obs_properties_create();

    // API Key (비밀번호 타입 = 마스킹)
    obs_properties_add_text(props, "api_key", "Soniox API Key", OBS_TEXT_PASSWORD);

    // 언어 선택 드롭다운
    obs_property_t *lang = obs_properties_add_list(props, "language", "Language",
        OBS_COMBO_TYPE_LIST, OBS_COMBO_FORMAT_STRING);
    obs_property_list_add_string(lang, "Korean", "ko");
    obs_property_list_add_string(lang, "English", "en");
    // ...

    // 폰트 크기 슬라이더
    obs_properties_add_int_slider(props, "font_size", "Font Size", 12, 120, 2);

    return props;
}

// 설정 변경 시 호출됨
static void soniox_caption_update(void *private_data, obs_data_t *settings) {
    const char *api_key = obs_data_get_string(settings, "api_key");
    int font_size = (int)obs_data_get_int(settings, "font_size");
    // → 텍스트 소스 업데이트
}
```

### 배운 것
- OBS Properties = 선언적 UI. 위젯 타입과 키를 지정하면 OBS가 UI를 자동 생성
- `OBS_TEXT_PASSWORD` → 입력값이 마스킹되고 "Show" 버튼이 자동 추가됨
- `get_defaults` → 기본값을 설정하면 "Defaults" 버튼 클릭 시 복원됨
- `update` 콜백에서 `obs_data_get_string/int`로 값을 읽을 수 있음
- settings 데이터는 OBS가 자동으로 저장/복원해 줌 (Scene Collection에 포함)

### 결과
Properties 창에 API Key 입력란, Language 드롭다운, Font Size 슬라이더 표시됨 ✓

---

---

## Step 4: WebSocket 연결 (Soniox 서버)

**목표**: Properties에서 "Test Connection" 버튼으로 Soniox 연결 확인

### 무엇을 했나
1. CMakeLists.txt에 FetchContent로 IXWebSocket + nlohmann/json 추가
2. "Test Connection" 버튼 추가
3. WebSocket 연결 → config JSON 전송 → 응답 확인

### 핵심 코드: FetchContent로 라이브러리 추가
```cmake
include(FetchContent)
FetchContent_Declare(ixwebsocket
  GIT_REPOSITORY https://github.com/machinezone/IXWebSocket.git
  GIT_TAG v11.4.5)
FetchContent_MakeAvailable(ixwebsocket)
target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE ixwebsocket nlohmann_json::nlohmann_json)
```

### 배운 것
- FetchContent = CMake의 npm install. Git에서 라이브러리를 자동 다운로드+빌드
- IXWebSocket의 API: `setUrl()` → `setOnMessageCallback()` → `start()`
- OBS 플러그인 템플릿의 `-Werror`가 서드파티 코드와 충돌 → `CMAKE_COMPILE_WARNING_AS_ERROR OFF`
- IXWebSocket는 macOS SecureTransport(deprecated) 대신 **OpenSSL 사용 권장**

### 트러블슈팅
- **IXWebSocket GitHub URL**: `nicehero/IXWebSocket` (잘못) → `machinezone/IXWebSocket` (정답)
- **-Werror 충돌**: Xcode는 프로젝트 전체에 적용 → 전역 OFF
- **Universal 빌드 실패**: Homebrew OpenSSL이 arm64만 → 개발 중 arm64 전용으로 전환

---

## Step 5+6: 오디오 캡처 → 실시간 자막

**목표**: 마이크 음성을 Soniox에 보내고 화면에 자막 표시

### 무엇을 했나
1. Properties에 Audio Source 드롭다운 추가 (OBS 오디오 소스 열거)
2. "Start Caption" / "Stop Caption" 버튼
3. OBS 오디오 콜백으로 마이크 캡처 → float32→int16 변환 → 48kHz→16kHz 다운샘플
4. WebSocket으로 PCM 바이너리 전송
5. Soniox 토큰 파싱 (final/non-final, endpoint)
6. OBS 텍스트 소스에 자막 업데이트
7. 폰트 선택 기능 추가 (한국어/CJK 폰트 지원)

### 핵심 개념: OBS 오디오 캡처
```cpp
// OBS 오디오 소스에 콜백 등록
obs_source_add_audio_capture_callback(audio_src, audio_capture_callback, data);

// 콜백에서 오디오 변환 + 전송
static void audio_capture_callback(void *param, obs_source_t *,
    const struct audio_data *audio, bool muted) {
    // float32 48kHz → int16 16kHz (3:1 다운샘플)
    pcm16[i] = (int16_t)(src[i * 3] * 32767.0f);
    websocket->sendBinary(pcm_data);
}
```

### 핵심 개념: Soniox 토큰 파싱
```
토큰 종류:
- is_final: true  → 확정 텍스트 (final_buffer에 누적)
- is_final: false → 미확정 텍스트 (매번 교체)
- text: "<end>"   → 발화 완료 (버퍼 초기화)

화면 표시 = final_buffer + non_final (실시간 갱신)
```

### 배운 것
- OBS 오디오: float32 planar, 보통 48000Hz
- Soniox PCM 설정 필수 필드: `audio_format`, `sample_rate`, **`num_channels`** (이름 주의!)
- `obs_source_create_private()`: 폰트 설정은 `font` 오브젝트로 전달 (face, size, style)
- macOS 한국어 기본 폰트: "Apple SD Gothic Neo"
- 오디오 콜백은 OBS 스레드에서 실행 → WebSocket 스레드와 다름 → IXWebSocket의 sendBinary는 thread-safe

### 트러블슈팅
- **"Audio data channels must be specified"**: `num_audio_channels`(틀림) → **`num_channels`**(정답). Soniox 공식 문서에서 확인
- **Disconnected 즉시 표시**: 기존 WebSocket stop() 시 Close 콜백이 `captioning=false`로 리셋 → `stopping` 플래그 추가
- **한글 깨짐 (네모)**: text_ft2_source_v2에 CJK 폰트 미설정 → font face 드롭다운 추가

---

## Step 7: 번역 기능

**목표**: Soniox 내장 번역으로 원문+번역을 동시에 표시

### 무엇을 했나
1. "Enable Translation" 체크박스 + "Translate To" 드롭다운 추가
2. WebSocket config에 translation 설정 추가
3. 토큰 파싱에서 번역 토큰 분리 처리 (translation_status === "translation")
4. 화면에 원문 + 줄바꿈 + 번역 표시

### 핵심 코드
```cpp
// config에 번역 추가
if (data->translate) {
    config["translation"] = {
        {"type", "one_way"},
        {"target_language", data->target_lang}
    };
}

// 토큰 분리
bool is_trans = token.value("translation_status", "") == "translation";
if (is_trans) { /* 번역 버퍼에 누적 */ }
else { /* 원문 버퍼에 누적 */ }

// 화면 표시
display = original + "\n" + translation;
```

### 배운 것
- Soniox는 STT와 번역을 동시에 제공 (별도 번역 API 불필요)
- `translation_status: "translation"` 토큰으로 구분
- ELSTTv2의 JavaScript 로직이 C++로 거의 1:1 포팅 가능

---

## 전체 진행 상황

| Step | Description | Status | Date |
|------|-------------|--------|------|
| 1 | Skeleton plugin (load/unload) | ✅ Done | 2026-04-05 |
| 2 | Register text source in OBS | ✅ Done | 2026-04-05 |
| 3 | Properties UI (API key, language, font) | ✅ Done | 2026-04-05 |
| 4 | WebSocket connection to Soniox | ✅ Done | 2026-04-05 |
| 5+6 | Audio capture + token parsing + display | ✅ Done | 2026-04-05 |
| 7 | Translation support | ✅ Done | 2026-04-05 |
| 8 | Cross-platform build + GitHub release | ✅ Done | 2026-04-05 |
| 9 | Hotkey toggle (Start/Stop) | ✅ Done | 2026-04-05 |
| 10 | Auto-reconnect | ✅ Done | 2026-04-05 |
| 11 | Dock panel UI | ⏸️ Blocked | AGL framework 호환 문제 |

---

## Step 8: GitHub CI/CD + Release

**목표**: push 시 macOS/Windows/Linux 자동 빌드 + Release 생성

### 무엇을 했나
1. obs-plugintemplate 기본 GitHub Actions 활용 (push.yaml, build-project.yaml)
2. `.Brewfile`에 OpenSSL 추가 (CI에서 IXWebSocket TLS 빌드용)
3. macOS CI를 Universal → arm64 전용으로 변경 (Homebrew OpenSSL이 arm64 only)
4. `build-macos` 스크립트에서 `-arch x86_64` 제거
5. `git tag 0.1.0` → 자동 Release 생성 (draft → publish)
6. macOS .plugin zip을 수동으로 Release에 업로드

### 배운 것
- obs-plugintemplate에 CI/CD가 이미 포함되어 있음 (GitHub Actions)
- 태그(`git tag x.x.x`)를 push하면 자동으로 Release가 생성됨 (draft 상태)
- `gh release edit --draft=false`로 공개 전환
- Homebrew는 Universal 빌드를 지원하지 않음 → arm64 전용으로 전환
- macOS `.pkg` 생성에는 Apple Developer 인증서($99/년)가 필요 → zip 배포로 대체
- CI 빌드 스크립트(`build-macos`)에 아키텍처가 하드코딩되어 있을 수 있음 → 확인 필요

### 트러블슈팅
- **CI x86_64 링크 실패**: CMakePresets.json만 변경으로는 부족, `build-macos` 스크립트에 `-arch x86_64`가 하드코딩 → 제거
- **gh CLI 인증**: `gh auth login -p https -w` → 브라우저에서 device code 입력

---

## Step 9: 핫키 토글

**목표**: Properties 열지 않고 단축키로 자막 시작/중지

### 핵심 코드
```cpp
// 소스 생성 시 핫키 등록
data->hotkey_id = obs_hotkey_register_source(source, "soniox_toggle_caption",
    "Toggle Soniox Captions", hotkey_toggle_caption, data);
```

OBS Settings → Hotkeys에서 키를 할당하면 Properties 없이 토글 가능.

---

## Step 10: 자동 재연결

**목표**: 네트워크 끊김 시 자동 재시도

### 핵심 코드
```cpp
data->websocket->enableAutomaticReconnection();
data->websocket->setMinWaitBetweenReconnectionRetries(3000);   // 3초
data->websocket->setMaxWaitBetweenReconnectionRetries(30000);  // 최대 30초
```

IXWebSocket 내장 기능 — 별도 구현 불필요.

---

## Step 11: Dock 패널 (보류)

**목표**: OBS 하단/측면에 항상 보이는 Start/Stop 패널

**보류 사유**: OBS Frontend API + Qt 활성화 시 macOS 26.x에서 `AGL` 프레임워크 미발견 에러. OBS SDK 31.1.1이 최신 macOS와 호환되지 않음. OBS SDK 업데이트 후 재시도 예정.

---

## 전체 진행 상황

| Step | Description | Status | Date |
|------|-------------|--------|------|
| 1 | Skeleton plugin (load/unload) | ✅ Done | 2026-04-05 |
| 2 | Register text source in OBS | ✅ Done | 2026-04-05 |
| 3 | Properties UI (API key, language, font) | ✅ Done | 2026-04-05 |
| 4 | WebSocket connection to Soniox | ✅ Done | 2026-04-05 |
| 5+6 | Audio capture + token parsing + display | ✅ Done | 2026-04-05 |
| 7 | Translation support | ✅ Done | 2026-04-05 |
| 8 | CI/CD + GitHub Release v0.1.0 | ✅ Done | 2026-04-05 |
| 8.1 | Intel Mac CI + Release v0.1.1 | ✅ Done | 2026-04-07 |
| 8.2 | Button text toggle fix v0.1.2 | ✅ Done | 2026-04-07 |
| 9 | Hotkey toggle | ✅ Done | 2026-04-05 |
| 10 | Auto-reconnect | ✅ Done | 2026-04-05 |
| 11 | Dock panel UI | ⏸️ Blocked | AGL 호환 문제 |

---

## Step 8.1: Intel Mac (x86_64) CI 빌드 + Release v0.1.1

**목표**: Apple Silicon뿐 아니라 Intel Mac에서도 실행 가능한 빌드 제공

### 무엇을 했나
1. `push.yaml` — `branches` 섹션 제거 (태그 push만 빌드, main push 시 Actions 실행 안 함)
2. `build-project.yaml` — `macos-x86_64-build` job 추가 (Apple Silicon 런너에서 크로스컴파일)
3. `CMakePresets.json` — `macos-ci-x86_64` preset 추가 (Intel Homebrew OpenSSL 경로)
4. `build-macos` 스크립트 — `--arch` 옵션 추가, x86_64일 때 Intel Homebrew로 OpenSSL 설치
5. `package-macos` 스크립트 — `--arch` 옵션 추가, output 파일명에 아키텍처 반영
6. `build-plugin/action.yaml`, `package-plugin/action.yaml` — target에서 arch 추출하여 전달
7. `CMakeLists.txt` — `OPENSSL_ROOT_DIR` 가드 추가 (`NOT DEFINED CACHE{OPENSSL_ROOT_DIR}`)

### 핵심 문제: arm64 런너에서 x86_64 크로스컴파일 시 OpenSSL 링크 실패

```
ld: symbol(s) not found for architecture x86_64
  "_SSL_connect", "_SSL_read", "_SSL_write" ... (OpenSSL 심볼들)
```

**원인**: macOS-15 런너는 Apple Silicon(arm64). `brew --prefix openssl`이 `/opt/homebrew/opt/openssl@3` (arm64)을 반환. CMakeLists.txt에서 이 경로를 `FORCE`로 설정하여 preset의 x86_64 경로를 덮어씀.

**해결**:
1. CMakeLists.txt에 `NOT DEFINED CACHE{OPENSSL_ROOT_DIR}` 가드 추가 → preset 값 보존
2. `macos-ci-x86_64` preset에 `OPENSSL_ROOT_DIR: "/usr/local/opt/openssl@3"` 명시
3. `build-macos` 스크립트에서 x86_64 빌드 시 Intel Homebrew (`/usr/local/bin/brew`) 설치 후 OpenSSL 설치

```cmake
# Before (문제)
if(APPLE)
  execute_process(COMMAND brew --prefix openssl ...)
  set(OPENSSL_ROOT_DIR "${OPENSSL_ROOT_DIR}" CACHE PATH "" FORCE)

# After (수정)
if(APPLE AND NOT DEFINED CACHE{OPENSSL_ROOT_DIR})
  execute_process(COMMAND brew --prefix openssl ...)
  set(OPENSSL_ROOT_DIR "${OPENSSL_ROOT_DIR}" CACHE PATH "" FORCE)
```

### 배운 것
- GitHub Actions macos-15 런너 = Apple Silicon (arm64). Intel 빌드는 크로스컴파일
- Apple Silicon Mac에는 Homebrew가 2개 공존 가능: `/opt/homebrew/bin/brew` (arm64), `/usr/local/bin/brew` (x86_64)
- Intel Homebrew 설치: `arch -x86_64 /bin/bash -c "$(curl -fsSL .../install.sh)"`
- CMake `CACHE{VAR}` — preset에서 설정한 cache 변수가 존재하는지 확인 가능 (CMake 3.21+)
- `FORCE` 키워드를 무분별하게 사용하면 preset/CI 설정을 덮어씀 → 가드 조건 필수
- main push에서 빌드 실행 방지: `on.push.branches` 제거하고 `on.push.tags`만 유지

### CI 워크플로우 구조 (v0.1.1)

```
push.yaml (태그 push만 트리거)
  ├── build-project.yaml
  │   ├── check-event        — 이벤트 분석 (codesign, notarize, package 여부)
  │   ├── macos-build        — Apple Silicon (arm64) 빌드 + 코드사인
  │   ├── macos-x86_64-build — Intel (x86_64) 크로스컴파일 (코드사인 없음)
  │   ├── ubuntu-build       — Ubuntu x86_64 빌드
  │   └── windows-build      — Windows x64 빌드
  └── create-release         — draft Release 생성 + artifact 첨부
```

### 결과
- v0.1.1 릴리스: macOS arm64 + x86_64 + Windows x64 + Ubuntu x86_64 빌드 성공 ✓
- main push 시 Actions 실행 안 됨 ✓

---

## Step 8.2: 버튼 텍스트 토글 수정 (v0.1.2)

**목표**: Start/Stop 버튼 클릭 후 텍스트가 즉시 변경되도록 수정

### 문제
`on_start_stop_clicked` 콜백에서 `return true` → `RefreshProperties()` 호출 → 기존 properties 객체에서 UI 재빌드. 하지만 `get_properties()`는 다시 호출되지 않으므로 조건부 텍스트 설정이 반영되지 않음.

### 해결
콜백의 두 번째 파라미터에 `property` 이름을 부여하고, `obs_property_set_description()`으로 직접 버튼 텍스트를 변경.

```cpp
// Before: obs_property_t * (unnamed)
// After:  obs_property_t *property

if (data->captioning) {
    stop_captioning(data);
    obs_property_set_description(property, "Start Caption");
} else {
    start_captioning(data);
    obs_property_set_description(property, "Stop Caption");
}
```

### OBS 내부 동작
- `RefreshProperties()`: 기존 `obs_properties_t`에서 위젯 재빌드 (lightweight)
- `ReloadProperties()`: `get_properties()` 재호출 (heavyweight)
- 버튼 클릭 콜백의 `return true`는 **RefreshProperties만** 트리거

### 배운 것
- OBS 버튼 콜백에서 텍스트를 바꾸려면 `obs_property_set_description()` 사용 필수
- `get_properties()`에서 조건부로 텍스트를 설정하는 방식은 버튼 클릭 시 동작하지 않음
- 이 패턴은 모든 OBS 플러그인의 버튼 토글에 공통 적용 (Speechmatics, ElevenLabs 동일)

---

## 빌드 치트시트

```bash
# 빌드 (수정 후 매번)
cmake --build --preset macos

# 설치 (빌드 후 매번)
rm -rf ~/Library/Application\ Support/obs-studio/plugins/soniox-caption-obs.plugin
cp -R build_macos/RelWithDebInfo/soniox-caption-obs.plugin \
  ~/Library/Application\ Support/obs-studio/plugins/

# OBS 재시작하여 확인

# 소스 파일 추가/삭제 시 (가끔)
rm -rf build_macos && cmake --preset macos

# Release 만들기
git tag x.x.x && git push origin x.x.x
# → GitHub Actions가 자동 빌드 + Release 생성
```
