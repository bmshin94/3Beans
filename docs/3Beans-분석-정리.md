# 3Beans 프로젝트 전수조사 분석 정리

> 작성: Claude Code (페르소나 "카리나") 와 함께한 대화 정리
> 작성일: 2026-09-28
> 대상 저장소: https://github.com/bmshin94/3Beans
> 원본(Upstream): https://github.com/Hydr8gon/3Beans

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [폴더 구조 전수조사](#2-폴더-구조-전수조사)
3. [핵심 코드 분석](#3-핵심-코드-분석)
4. [LLE vs HLE 쉽게 이해하기](#4-lle-vs-hle-쉽게-이해하기)
5. [어떤 용도로 쓰는가](#5-어떤-용도로-쓰는가)
6. [나에게 어떤 도움이 되는가](#6-나에게-어떤-도움이-되는가)
7. [설치 및 사용법](#7-설치-및-사용법)
8. [플러그인·스킬·MCP 여부](#8-플러그인스킬mcp-여부)
9. [API 토큰 필요 여부](#9-api-토큰-필요-여부)
10. [GitHub에서 유명한 이유](#10-github에서-유명한-이유)
11. [로컬 에이전트 구축에 도움이 될까](#11-로컬-에이전트-구축에-도움이-될까)
12. [React / PHP 로 만들 수 있을까](#12-react--php-로-만들-수-있을까)
13. [수익화 아이디어 (상세)](#13-수익화-아이디어-상세)
14. [법적·라이선스 주의사항](#14-법적라이선스-주의사항)
15. [참고 링크 모음](#15-참고-링크-모음)

---

## 1. 프로젝트 개요

**3Beans 는 닌텐도 3DS 를 "저수준(LLE, Low-Level Emulation)" 방식으로 에뮬레이션하는
독립 실행형 데스크톱 애플리케이션이다.**

| 항목 | 내용 |
|---|---|
| 정체 | 3DS 저수준 에뮬레이터 (네이티브 데스크톱 앱) |
| 원작자 | Hydr8gon (실명 Sean Maas) |
| 원본 저장소 | https://github.com/Hydr8gon/3Beans |
| 스타 / 포크 | ⭐ 662 / 🍴 12 (조사 시점) |
| 열린 이슈 | 19개 |
| 라이선스 | **GPL-3.0** |
| 언어 / 표준 | C++ (C++11) |
| 프로젝트 시작 | 2023-09-14 |
| 최신 업스트림 커밋 | 2026-08-05 (`Fix TEV previous buffer updating too soon`) |
| 규모 | 소스 파일 **93개**, 총 **34,792줄** |
| 지원 플랫폼 | Windows / macOS / Linux |
| 의존성 | wxWidgets 3.3.2+, PortAudio, libepoxy |

### 원작자가 밝힌 목표 (README 인용)

> "3Beans emulates the 3DS at a low level, which means that it runs the entire OS
> as if it were on real hardware. ... My goal for this project is to achieve
> viable speeds with full low-level emulation."

즉 **"실기와 동일하게 3DS OS 전체를 부팅시키면서도 실용적인 속도를 내는 것"** 이 목표다.

### 현재 완성도

- 홈 메뉴 부팅 ✅
- 일부 게임 실행 ✅
- 소프트웨어 렌더링 + OpenGL 하드웨어 렌더링 ✅
- CPU 는 **전부 인터프리터** (JIT 미구현) → 속도가 느린 주된 이유
- New 3DS 모드는 지원하지만 이슈가 더 많고 훨씬 느림

---

## 2. 폴더 구조 전수조사

```
3Beans/
├── src/
│   ├── core/                     에뮬레이터 두뇌 (하드웨어 시뮬레이션)
│   │   ├── core.cpp / core.h     전체 총괄 지휘자 + 이벤트 스케줄러
│   │   ├── settings.cpp / .h     3beans.ini 설정 저장·로드
│   │   ├── defines.h             비트 매크로, 로그 매크로, CpuId enum
│   │   ├── arm/                  CPU 에뮬레이션 (약 6,730줄)
│   │   │   ├── arm_interp*.cpp   ARM/Thumb 명령어 인터프리터
│   │   │   ├── vfp11_interp.cpp  부동소수점 유닛(VFP11)
│   │   │   ├── cp15.cpp          MMU / 캐시 제어 코프로세서
│   │   │   ├── interrupts.cpp    인터럽트 컨트롤러
│   │   │   └── timers.cpp        타이머
│   │   ├── memory/               메모리 맵 & DMA (약 4,300줄)
│   │   │   ├── memory*.cpp       주소 → 포인터 매핑, I/O 레지스터
│   │   │   ├── cdma.cpp          CDMA / XDMA 컨트롤러
│   │   │   └── ndma.cpp          NDMA 컨트롤러
│   │   ├── gpu/                  그래픽 (약 7,800줄) ← 가장 큼
│   │   │   ├── gpu.cpp           GPU 코어 + 스레드 처리
│   │   │   ├── gpu_cmd.cpp       GPU 커맨드 리스트 해석
│   │   │   ├── gpu_render_soft   소프트웨어 래스터라이저
│   │   │   ├── gpu_render_ogl    OpenGL 하드웨어 렌더러
│   │   │   ├── gpu_shader_interp 셰이더 인터프리터
│   │   │   ├── gpu_shader_glsl   셰이더 → GLSL JIT 변환
│   │   │   └── pdc.cpp           디스플레이 컨트롤러 (화면 출력)
│   │   ├── dsp/                  사운드 (약 4,700줄)
│   │   │   ├── teak_interp*.cpp  Teak DSP 프로세서 인터프리터
│   │   │   ├── dsp_lle.cpp       저수준 DSP (정확)
│   │   │   ├── dsp_hle.cpp       고수준 DSP (빠름)
│   │   │   └── csnd.cpp          CSND 사운드 하드웨어
│   │   ├── io/                   입출력 (약 2,100줄)
│   │   │   ├── cartridge.cpp     카트리지 리더 (NTR/CTR 프로토콜)
│   │   │   ├── sd_mmc.cpp        SD카드 / 내장 NAND
│   │   │   ├── i2c.cpp           MCU, 카메라 등 I2C 장치
│   │   │   ├── input.cpp         버튼 / 터치스크린 / 슬라이드패드
│   │   │   └── wifi.cpp          WiFi 칩 (실제 인터넷 연결 아님)
│   │   └── convert/              암호화 하드웨어 (약 1,300줄)
│   │       ├── aes.cpp / rsa.cpp / sha.cpp
│   │       └── y2r.cpp           YUV → RGB 변환 하드웨어
│   └── desktop/                  wxWidgets GUI (약 2,000줄)
│       ├── main.cpp / b3_app     앱 진입점, 설정 로딩, 키바인딩 기본값
│       ├── b3_frame.cpp          메뉴바 / 메인 윈도우
│       ├── b3_canvas_ogl         OpenGL 출력 캔버스
│       ├── b3_canvas_soft        소프트웨어 출력 캔버스
│       └── *_dialog.cpp          경로/입력/GPU/하드웨어 설정 창
├── .github/
│   ├── workflows/autobuild.yml   Win·Mac·Linux 자동 빌드 + 롤링 릴리즈
│   ├── pull_request_template.md  "PR 받지 않습니다" 안내
│   └── FUNDING.yml               Patreon: Hydr8gon / paypal.me/Hydr8gon
├── meta/                         macOS .app 번들, Flatpak manifest, .desktop
├── icon/                         windows.ico / linux.png / mac.icns
├── Makefile                      빌드 + install/uninstall/flatpak 타겟
├── LICENSE                       GPL-3.0 전문 (674줄)
├── README.md                     개요·설치·빌드·레퍼런스
└── CLAUDE.md                     ★ 이 포크에서 추가한 Claude 페르소나 지침
```

### 코드량 상위 파일

| 줄 수 | 파일 |
|---|---|
| 1,766 | `src/core/memory/memory_write.cpp` |
| 1,733 | `src/core/arm/arm_interp_alu.cpp` |
| 1,722 | `src/core/arm/arm_interp_transfer.cpp` |
| 1,569 | `src/core/memory/memory_read.cpp` |
| 1,504 | `src/core/gpu/gpu_render_ogl.cpp` |
| 1,349 | `src/core/arm/arm_interp_lookup.cpp` |
| 1,283 | `src/core/gpu/gpu_render_soft.cpp` |
| 1,278 | `src/core/gpu/gpu_cmd.cpp` |

---

## 3. 핵심 코드 분석

### 3.1 Core = 이벤트 드리븐 스케줄러

`src/core/core.h` 의 `Core` 클래스가 모든 하드웨어 객체를 소유한다.
그리고 `Task` enum 에 **52개의 이벤트**를 정의해두고, "몇 사이클 뒤에 무엇을 할지"를
정렬된 큐에 넣어 실행한다.

```cpp
struct Event {
    std::function<void()> *task;
    uint64_t cycles;
    bool operator<(const Event &event) const { return cycles < event.cycles; }
};

void Core::schedule(Task task, uint64_t cycles) {
    Event event(&tasks[task], globalCycles + cycles);
    auto it = std::upper_bound(events.cbegin(), events.cend(), event);
    events.insert(it, event);   // 시간순 정렬 삽입
}
```

초기 스케줄:

```cpp
schedule(RESET_CYCLES, 0x7FFFFFFFFFFFFFFF);  // 사이클 카운터 오버플로 방지
schedule(END_FRAME, 268111856 / 60);         // 1프레임 = 4,468,530 사이클
schedule(CSND_SAMPLE, 2048);                 // 오디오 샘플
```

- `268111856` = 3DS ARM11 실제 클럭 **268.11 MHz**
- 이를 60 으로 나눠 "1프레임 분량의 사이클 수"를 구한다 → 실기 타이밍 그대로 재현

`resetCycles()` 는 `globalCycles` 가 커져 오버플로되는 것을 막기 위해
모든 이벤트/CPU/타이머의 사이클을 일괄로 빼서 0 으로 되돌리는 영리한 처리다.

### 3.2 CPU — ARM11 쿼드코어 + ARM9

```cpp
enum CpuId { ARM11A, ARM11B, ARM11C, ARM11D, ARM9, MAX_CPUS, ARM11 = ARM11A };
```

- 실기 3DS = ARM11 MPCore (New 3DS 는 4코어) + 보조 ARM9
- 모두 **인터프리터** 방식. `arm_interp_lookup.cpp` 에 함수 포인터 테이블:

```cpp
static int (ArmInterp::*armInstrs[0x1000])(uint32_t);
static int (ArmInterp::*thumbInstrs[0x400])(uint16_t);
```

opcode 상위 비트로 인덱싱해 O(1) 디스패치한다.

### 3.3 템플릿으로 런루프를 특수화

```cpp
template <bool cores, bool dsp> static void runFrame(Core &core);

void Core::updateRunFunc() {
    bool dspOff = (dspCurrent == 1 || ((DspLle*)dsp)->teak.cycles == -1);
    if ((interrupts.cfg11MpBootcnt[0] | interrupts.cfg11MpBootcnt[1]) & BIT(4))
        runFunc = dspOff ? &ArmInterp::runFrame<true, false> : &ArmInterp::runFrame<true, true>;
    else
        runFunc = dspOff ? &ArmInterp::runFrame<false, false> : &ArmInterp::runFrame<false, true>;
    running.store(false);
}
```

"코어 2·3 이 켜졌는가", "DSP 를 돌려야 하는가"를 **컴파일 타임 상수로 박아넣은
4가지 버전**을 만들어두고 런타임에 함수 포인터만 교체한다.
→ 매 명령어마다 if 분기를 없애는 고성능 기법.

### 3.4 메모리 — 4KB 페이지 포인터 테이블

```cpp
struct MemMap { uint8_t *read, *write; uint32_t tag; };
MemMap memMap11[0x100000] = {};   // ARM11용, 4GB / 4KB = 1M 엔트리
MemMap memMap9[0x100000]  = {};   // ARM9용
```

실제 메모리 블록:

```cpp
uint8_t arm9Ram[0x180000];  // 1.5MB ARM9 내부 RAM
uint8_t vram[0x600000];     // 6MB VRAM
uint8_t dspWram[0x80000];   // 512KB DSP RAM
uint8_t axiWram[0x80000];   // 512KB AXI WRAM
uint8_t fcram[0x8000000];   // 128MB FCRAM (메인 메모리)
uint8_t boot11[0x10000];    // 64KB ARM11 부트롬
uint8_t boot9[0x10000];     // 64KB ARM9 부트롬
uint8_t *fcramExt;          // New 3DS 확장 128MB
uint8_t *vramExt;           // New 3DS 확장 4MB
```

읽기 경로가 인상적이다:

```cpp
template <typename T> FORCE_INLINE T Memory::read(CpuId id, uint32_t address) {
    if (uint8_t *data = (id == ARM9 ? memMap9 : memMap11)[address >> 12].read) {
        T value = 0;
        data += (address & 0xFFF);
        for (uint32_t i = 0; i < sizeof(T); i++)
            value |= data[i] << (i << 3);
        return value;
    }
    return readFallback<T>(id, address);   // I/O 레지스터 등은 느린 경로
}
```

**빠른 경로(일반 RAM)는 포인터 한 번, 느린 경로(I/O)는 별도 함수** 로 나눈 전형적인
hot/cold path 분리 패턴이다.

또한 `write()` 는 `map.tag++` 로 해당 페이지가 변경됐음을 표시한다
→ GPU 텍스처 캐시 무효화 등에 활용된다.

I/O 레지스터 정의는 매크로로 선언형처럼 만들었다:

```cpp
#define DEF_IO32(addr, func) \
    case addr + 0: case addr + 1: case addr + 2: case addr + 3: \
        base &= 0x3; size = 4; func; goto next;
```

### 3.5 GPU — 백엔드 교체 가능 (전략 패턴)

```
GpuRender  (추상 베이스)
 ├─ GpuRenderSoft   소프트웨어 래스터라이저 (정확, 느림)
 └─ GpuRenderOgl    OpenGL 하드웨어 렌더러 (빠름)

GpuShader  (추상 베이스)
 ├─ GpuShaderInterp 셰이더 명령어 인터프리터
 └─ GpuShaderGlsl   셰이더 → GLSL 로 JIT 변환
```

설정값 `gpuRenderer`, `gpuVtxShader`, `gpuFragShader` 로 런타임 교체.
`threadedGpu` 를 켜면 GPU 작업을 별도 스레드로 넘긴다:

```cpp
while (thread && taskStart.load() != taskEnd.load())
    std::this_thread::yield();
```

`std::atomic` 기반 락프리 워크 큐다.

3DS GPU(PICA200) 고유의 TEV(Texture Environment) 조합기도 enum 으로 충실히 모델링되어 있다
(`CombSrc`, `CombOper`, `CalcMode`, `TexWrap`, `CullMode`, `PrimMode`).

### 3.6 DSP — LLE / HLE 이중 구현

```cpp
class Dsp;                 // 추상 인터페이스
class DspLle : public Dsp; // Teak 프로세서를 명령어 단위로 에뮬레이션
class DspHle : public Dsp; // 상태 머신으로 펌웨어 동작만 흉내
```

`Core::initDsp()` 는 백엔드를 **핫스왑**한다. 주목할 점은
**교체 전에 기존 DSP 용으로 예약된 이벤트를 스케줄러에서 제거**한다는 것:

```cpp
for (int i = 0; i < events.size(); i++)
    for (int j = 0; j < sizeof(dspTasks) / sizeof(Task); j++)
        if (events[i].task == &tasks[dspTasks[j]])
            events.erase(events.begin() + i--);
delete dsp;
// ... 새 백엔드 생성 후 tasks[] 재정의
```

`DspHle` 의 `DspState` enum 은 핸드셰이크 → 초기화 → 수신 → 응답 → 실행 →
인터럽트 → 리셋 단계를 명시적 상태 머신으로 표현한다.

### 3.7 설정 시스템

```cpp
struct Setting { std::string name; void *value; bool isString; };
std::vector<Setting> settings = { Setting("fpsLimiter", &fpsLimiter, false), ... };
void Settings::add(std::vector<Setting> &extra);   // 런타임 확장
```

코어는 코어 설정만 알고, 데스크톱 프론트엔드가 `Settings::add()` 로
키바인딩·오디오 버퍼 같은 **플랫폼 전용 설정을 주입**한다.
저장 포맷은 단순 `key=value` 텍스트(`3beans.ini`).

설정 파일 탐색 순서: 작업 디렉터리 → OS 표준 설정 디렉터리(Windows/macOS 는
UserDataDir, Linux 는 XDG `~/.config/3beans`).

### 3.8 CI / 배포 파이프라인

`.github/workflows/autobuild.yml` 는 `main` 에 push 될 때마다:

1. **Windows**: MSYS2 UCRT64 + pacman 의존성 → `make` → `3beans.exe`
2. **macOS**: Nix 로 portaudio/libepoxy, wxWidgets 3.3.2 소스 빌드 → `.dmg`
3. **Linux**: Flatpak SDK → `3beans.flatpak`
4. **release**: 기존 `release` 태그 삭제 → 3개 아티팩트 zip → 새 릴리즈 생성

즉 **"항상 최신 커밋의 3개 OS 빌드가 릴리즈 페이지에 올라가 있는" 롤링 릴리즈** 구조.
1인 개발 프로젝트가 참고할 만한 최고의 템플릿 중 하나.

---

## 4. LLE vs HLE 쉽게 이해하기

에뮬레이터 = **"다른 기계인 척 하는 프로그램"**.
3DS 게임은 "3DS 부품"에게 말을 걸지만 PC 에는 그 부품이 없으므로,
소프트웨어로 가짜 부품을 만들어준다. 방법이 두 가지다.

### 방법 A. 통역사 방식 (HLE, High-Level Emulation)

게임이 OS 함수를 호출하면 그 호출을 가로채서 PC 방식으로 대신 처리한다.

- 게임: "저장해줘" → 통역사: "윈도우 파일로 저장해줄게"
- 장점: 빠르다
- 단점: 통역사가 모르는 호출이 나오면 게임이 안 돌아간다 → 게임별 대응 필요
- 예: Citra

### 방법 B. 복제 방식 (LLE, Low-Level Emulation) ← 3Beans

칩 하나하나의 동작을 그대로 프로그램으로 재현해서, **진짜 3DS 펌웨어를 부팅**시킨다.

- 게임: "저장해줘" → 가짜 3DS OS 가 받아서 → 가짜 SD 컨트롤러에 기록
- 장점: 실기와 거의 동일한 동작, 홈 메뉴/부팅 시퀀스까지 재현
- 단점: 느리고 구현 난도가 극도로 높다, 실기 덤프 파일이 필수

### 비교표

| 구분 | HLE (Citra 등) | **LLE (3Beans)** |
|---|---|---|
| 방식 | OS 함수 호출 대체 | 하드웨어 자체 재현 |
| 필요 파일 | 게임 롬만 | **boot9.bin, boot11.bin, nand.bin** |
| 속도 | 빠름 | 느림 |
| 정확도 | 게임별 패치 필요 | 실기와 거의 동일 |
| 홈 메뉴 | 안 나옴 | **나온다** |
| 부팅 시퀀스 | 없음 | 전원 ON 부터 재현 |
| 롬 형식 | 복호화된 덤프 | **암호화된 덤프** |

### 폴더 = 3DS 부품 공장 (비유)

| 폴더 | 실제 부품 | 쉬운 비유 |
|---|---|---|
| `arm/` | CPU 5개 | 두뇌 — 계산 |
| `memory/` | RAM 128MB | 책상 — 작업 공간 |
| `gpu/` | PICA200 그래픽칩 | 화가 — 화면 그리기 |
| `dsp/` | Teak 사운드칩 | 스피커 — 소리 |
| `io/` | 카트리지·터치·SD | 손과 입 — 입출력 |
| `convert/` | AES/RSA/SHA 칩 | 금고 — 복호화 |
| `desktop/` | — | 창틀 — PC 윈도우 |

### 스케줄러 = 연극 무대감독 (비유)

무대감독이 대본에 "4,468,530 클럭 뒤에 1프레임 종료", "2,048 클럭 뒤에 오디오 샘플",
"N 클럭 뒤에 타이머 인터럽트"를 적어두고, 가장 먼저 일어날 일부터 순서대로 실행한다.
덕분에 모든 부품이 실기와 같은 타이밍으로 맞물려 동작한다.

---

## 5. 어떤 용도로 쓰는가

1. **게임 보존(Preservation)** — 3DS 단종 + eShop 종료로 디지털 게임이 소실 위험.
   LLE 에뮬레이터는 "하드웨어 자체의 기록"이라는 점에서 보존 가치가 가장 크다.
2. **하드웨어 연구 / 리버스 엔지니어링** — 3DS 칩의 실제 동작을 코드로 문서화.
   특히 Teak DSP 는 공개 자료가 거의 없어 이 구현 자체가 레퍼런스가 된다.
3. **정확도 기준점(Reference)** — 다른 에뮬레이터의 구현이 맞는지 비교하는 기준.
4. **홈브류 개발/테스트** — 직접 만든 3DS 앱을 실기 없이 검증.
5. **학습용** — 컴퓨터 구조, 시스템 프로그래밍 교보재.

> 주의: 실기에서 추출한 부트롬·NAND 가 없으면 **실행 자체가 불가능**하다.

---

## 6. 나에게 어떤 도움이 되는가

솔직한 결론: **"실행해서 쓰는 도구"보다 "읽어서 배우는 교과서"로서의 가치가 훨씬 크다.**

### 6.1 C++ 실전 설계 교과서

한 저장소 안에 다음 패턴이 전부 살아있다.

- 이벤트 기반 스케줄러 (정렬 큐 + `std::function`)
- 함수 포인터 테이블 디스패치 (`armInstrs[0x1000]`)
- 템플릿 메타프로그래밍으로 런루프 특수화 (`runFrame<cores, dsp>`)
- 추상 클래스 + 전략 패턴 (렌더러 2종, 셰이더 2종, DSP 2종 런타임 교체)
- hot/cold path 분리 (`read()` vs `readFallback()`)
- `std::atomic` 락프리 워크 큐 (GPU 스레드)
- 매크로 DSL (`DEF_IO32` 로 I/O 맵을 선언형으로)
- 의존성 주입 (모든 컴포넌트가 `Core&` 를 생성자로 받음)

### 6.2 크로스플랫폼 배포 완전체 예제

`autobuild.yml` 하나로 Windows·macOS·Linux 3개 빌드 + 자동 릴리즈.
파일만 가져다 고쳐 쓰면 내 프로젝트에 바로 적용 가능하다.
`Makefile` 의 OS 분기, `meta/` 의 mac 번들·Flatpak manifest·`.desktop` 파일도 그대로 참고 가능.

### 6.3 저수준 컴퓨터 구조 학습

CPU 파이프라인, 레지스터 뱅킹, 인터럽트, DMA, MMU(CP15), 메모리 맵,
GPU 파이프라인, 오디오 DMA 까지 "컴퓨터가 실제로 도는 원리"를 코드로 볼 수 있다.

### 6.4 1인 개발 프로젝트 운영 사례

"PR 받지 않음" 명시 + 이슈/피드백만 수용 + Patreon 후원 + 롤링 릴리즈 + Discord 커뮤니티.
오픈소스를 지속 가능하게 운영하는 실제 모델 샘플.

---

## 7. 설치 및 사용법

### 7.1 방법 A — 빌드된 파일 받기 (권장)

1. https://github.com/Hydr8gon/3Beans/releases 접속
2. OS 에 맞는 zip 다운로드
   - `3beans-windows.zip` (exe)
   - `3beans-mac.zip` (dmg)
   - `3beans-linux.zip` (flatpak)
3. 실행

### 7.2 방법 B — 직접 빌드

**Windows (MSYS2)**

```bash
pacman -Syu mingw-w64-ucrt-x86_64-{gcc,pkg-config,wxwidgets3.3-msw,portaudio,libepoxy,jbigkit} make
cd 3Beans
make -j$(nproc)
```

**macOS**

```bash
brew install wxwidgets portaudio libepoxy
make -j$(sysctl -n hw.logicalcpu)
make install      # /Applications 에 .app 설치
```

**Linux**

```bash
# 배포판 패키지 매니저로 wxWidgets 3.3.2+, PortAudio, libepoxy 설치
make -j$(nproc)
sudo make install
# 또는 Flatpak 패키지 생성
make flatpak
```

> wxWidgets 는 **3.3.2 이상** 필요. 배포판 기본 저장소 버전이 낮으면 직접 빌드해야 한다.

기타 Make 타겟: `clean`, `uninstall`, `flatpak`, `flatpak-clean`
컴파일 옵션: `-O3 -flto -std=c++11 -DLOG_LEVEL=0` (LOG_LEVEL 을 올리면 디버그 로그 출력)

### 7.3 필수 덤프 파일 (이게 없으면 실행 불가)

| 파일 | 내용 | 획득 방법 |
|---|---|---|
| `boot9.bin` | ARM9 부트롬 64KB | **본인 소유 3DS** + GodMode9 로 추출 |
| `boot11.bin` | ARM11 부트롬 64KB | 동일 |
| `nand.bin` | 내장 스토리지 전체 이미지 (약 1GB) | 동일 |
| `sd.img` | SD 카드 이미지 (선택) | FAT 포맷 이미지 파일 생성 |

- GodMode9: https://github.com/d0k3/GodMode9
- Old 3DS 가 호환성·속도 면에서 유리. New 3DS 는 이슈가 많고 훨씬 느림.
- 경로 설정: `Settings → Path Settings` (설정은 `3beans.ini` 에 저장)
- **이 파일들은 배포가 불법이다. 반드시 본인 기기에서 직접 추출해야 한다.**

### 7.4 사용법

- `File → Insert Cart ROM` : 카트리지 롬 삽입
  - **암호화된(encrypted) 덤프**여야 한다. 고수준 에뮬레이터와 반대이므로 주의.
- `File → Eject Cart ROM` / `Quit`
- `System → Pause / Restart / Stop / Set Hardware`
- `Settings → FPS Limiter` / `Cart Auto-Boot` (홈 메뉴 건너뛰고 게임 직행)
- `Settings → DSP Backend → Interpreter | HLE`
- `Settings → GPU Settings` (렌더러, 정점/프래그먼트 셰이더, 스레드 GPU)
- `Settings → Path Settings` / `Input Bindings`

**기본 키 매핑** (`b3_app.cpp`)

| 3DS 버튼 | 기본 키 |
|---|---|
| A / B | `L` / `K` |
| X / Y | `O` / `I` |
| L / R | `Q` / `P` |
| Select / Start | `G` / `H` |
| 십자키 | 방향키 |
| 슬라이드패드 | `D` / `A` / `W` / `S` |
| 슬라이드패드 모드 | `Shift` |
| HOME | `Space` |

---

## 8. 플러그인·스킬·MCP 여부

### 결론: **셋 다 아니다.**

3Beans 는 **wxWidgets 로 만든 독립 실행형 네이티브 데스크톱 애플리케이션(C++)** 이다.

| 구분 | 정의 | 3Beans |
|---|---|---|
| 플러그인 | 호스트 앱에 끼워 넣는 확장 모듈 | ❌ |
| 스킬(Skill) | Claude 에게 주는 지침 마크다운 (`SKILL.md`) | ❌ |
| MCP | AI ↔ 외부 도구 연결 프로토콜 (JSON-RPC 서버) | ❌ |
| 3Beans | 실행 파일(exe/app/flatpak) | ✅ |

### 그럼 `CLAUDE.md` 는?

이 포크에서 **직접 추가한 파일**이다 (커밋 `086fe73 docs: created CLAUDE.md persona guide`,
PR #1 `feat/claude-guide` 로 머지됨). 원본 Hydr8gon 저장소에는 없다.

역할은 "Claude Code 가 이 저장소에서 작업할 때 지켜야 할 규칙"을 정의하는
**프로젝트 메모리(지침 파일)** 이다. 내용은 "카리나" 페르소나 정의
(한국어 응답, 친근한 말투, 이모지 사용 등).

→ 굳이 분류하면 플러그인/스킬/MCP 중 **"프로젝트 메모리"** 에 해당하며,
3Beans 프로그램 자체와는 아무 관련이 없다.

---

## 9. API 토큰 필요 여부

### 결론: **전혀 필요 없다.**

전수조사 결과:

- 네트워크 라이브러리 없음 (libcurl, OpenSSL 등 미사용)
- 외부 서버 통신 코드 없음
- 계정·로그인·결제 기능 없음
- 의존성은 `wxWidgets`(GUI), `PortAudio`(오디오), `libepoxy`(OpenGL) 3개뿐

`src/core/io/wifi.cpp` 가 있지만 이는 **3DS 의 WiFi 칩 하드웨어를 에뮬레이션**하는
코드이며, 실제 인터넷 통신을 하지 않는다.
`src/core/convert/` 의 AES/RSA/SHA 역시 3DS 내장 암호화 하드웨어 재현용이다.

**완전 오프라인 로컬 애플리케이션.**

> 단, 이 저장소를 Claude Code 로 작업할 때는 Claude 구독/API 가 필요하다.
> 그것은 3Beans 와 무관한 별개 사안이다.

---

## 10. GitHub에서 유명한 이유

⭐ **662 stars**, 🍴 12 forks, 19 open issues — 에뮬레이터 씬 기준 상당한 수치.

### 10.1 희소성 — 사실상 유일한 현역 3DS LLE 에뮬레이터

3DS 저수준 에뮬레이터는 역사상 **Corgi3DS**(PSI-Rockin, 개발 중단) 하나뿐이었다.
3Beans 는 현재 **활발히 개발되는 유일한 3DS LLE 프로젝트**다.

### 10.2 개발자의 기존 명성

Hydr8gon 은 **NooDS**(NDS 에뮬레이터), **sm64_switch** 등으로 이미 알려진 개발자.
새 프로젝트에 기존 팔로워가 자동 유입된다.

### 10.3 "불가능하다던 것"을 해냄

- 실제로 3DS 홈 메뉴가 부팅되는 결과물
- Teak DSP 를 명령어 단위로 구현 (Teakra 외에 레퍼런스가 거의 없는 영역)
- 실기 클럭 268.11MHz 를 숫자 그대로 사용하는 타이밍 정확도
- 소프트웨어/하드웨어 렌더러, 셰이더 인터프리터/GLSL JIT 를 모두 제공

### 10.4 시기적 타이밍

- 2023: 3DS eShop 완전 종료 → 게임 보존 이슈 급부상
- 2024: 닌텐도 소송으로 Yuzu 폐쇄, 그 여파로 Citra 도 개발 중단
- 그 공백기에 "새로운, 더 정확한 접근법"으로 등장

### 10.5 1인 개발자의 꾸준함과 운영 품질

- 2023-09 시작 → 2026-08 현재까지 지속 커밋 (3년간 약 3.5만 줄)
- 커밋 메시지가 간결하고 이슈 번호를 링크 (`Fixes #12`, `Fixes #3 and fixes #11`)
- 커밋마다 3개 OS 빌드를 자동 배포하는 롤링 릴리즈
- Discord 커뮤니티 + 블로그 개발기 + Patreon 후원
- "PR 은 받지 않습니다"라는 명확한 선언 (오히려 화제가 됨)

---

## 11. 로컬 에이전트 구축에 도움이 될까

### 결론: **직접적으로는 무관, 아키텍처 학습용으로는 매우 유용.**

AI/LLM 관련 코드는 0줄이다. 하지만 아래 패턴들은 에이전트 런타임 설계에 그대로 쓸 수 있다.

### 11.1 이벤트 스케줄러 → 에이전트 태스크 큐

```cpp
struct Event { std::function<void()> *task; uint64_t cycles; };
// cycles 를 timestamp 로 바꾸면 그대로 "예약 액션 큐"
```

`Task` enum + `tasks[MAX_TASKS]` 배열로 태스크 종류를 관리하는 방식은
에이전트의 **툴 레지스트리** 설계와 동일한 발상이다.

### 11.2 백엔드 핫스왑 (전략 패턴)

```
Dsp        → DspLle (정확/느림)   | DspHle (빠름/근사)
GpuRender  → GpuRenderSoft        | GpuRenderOgl
GpuShader  → GpuShaderInterp      | GpuShaderGlsl
```

설정값 하나로 런타임에 구현체를 교체하고, **교체 시 기존 예약 작업까지 정리**하는
`initDsp()` 로직은 에이전트에서 **"로컬 LLM ↔ 클라우드 API 런타임 전환"** 을
구현할 때 거의 그대로 응용할 수 있다.

### 11.3 스레드 안전한 백그라운드 워커

```cpp
std::atomic<bool> running{false};
while (thread && taskStart.load() != taskEnd.load())
    std::this_thread::yield();
```

논블로킹으로 워커에 작업을 넘기고 완료를 기다리는 구조 = 에이전트 백그라운드 작업 처리.

### 11.4 확장 가능한 설정 시스템

`Settings` 네임스페이스 + `std::vector<Setting>` 등록 + `Settings::add()` 로
런타임 확장 → 에이전트 config/플러그인 설정 시스템 설계 참고.

### 11.5 정리

| 목표 | 도움 정도 |
|---|---|
| LangChain/AutoGPT 류 프레임워크 구축 | ❌ 무관 |
| 에이전트 **런타임 아키텍처** 설계 학습 | ⭕⭕⭕ |
| 고성능 C++ 런타임/스케줄러 구현 | ⭕⭕⭕ |
| MCP 서버 구현 | ❌ 별도 학습 필요 |

---

## 12. React / PHP 로 만들 수 있을까

### 결론: **에뮬레이터 본체는 불가능, 주변 생태계는 충분히 가능.**

### 12.1 PHP — 불가능 (0%)

- PHP 는 요청 처리 후 종료되는 모델. 초당 2.68억 사이클 루프에 부적합
- 실시간 그래픽/오디오 출력 불가
- C++ 대비 성능 차이가 수천 배 → 60fps 목표에 근본적으로 도달 불가

### 12.2 React / JavaScript — "조건부 가능" (30%)

React 는 UI 라이브러리이므로 단독으로는 불가능하다. 현실적인 경로는 하나뿐:

```
[C++ 코어] --Emscripten--> [WebAssembly] --> [React 로 UI 감싸기]
```

이는 실제 업계 표준 방식이며 선례가 있다:
**EmulatorJS**, **melonDS Web**, **RetroArch Web**, **Dolphin Web**

하지만 3Beans 에는 다음 장벽이 있다.

| 장벽 | 내용 |
|---|---|
| 성능 | WASM 은 네이티브의 50~70%. 3Beans 는 **네이티브에서도 풀스피드가 아니다** → 실사용 불가 수준 예상 |
| 메모리 | FCRAM 128MB + 확장 128MB + VRAM 10MB → 브라우저 힙 압박 |
| 스레드 | `std::thread` 사용 → SharedArrayBuffer 필요 → COOP/COEP 헤더 설정 필수 |
| 그래픽 | libepoxy/OpenGL → WebGL2 로 전면 포팅 필요 |
| 파일 I/O | 1GB `nand.bin` 을 IndexedDB/OPFS 로 다뤄야 함 |
| 법률 | 부트롬·NAND 배포 불가 → 사용자가 직접 업로드해야 하는 UX 문제 |

### 12.3 React / PHP 로 "진짜 만들 수 있는 것"

1. **게임 호환성 DB 사이트** — PHP/Laravel + MySQL + React. 크라우드소싱 호환성 리포트
2. **웹 설정 에디터** — `3beans.ini` 를 GUI 로 편집 후 다운로드
3. **성능 리포트 대시보드** — 사용자 FPS 로그 수집 → 하드웨어별 통계 시각화
4. **세이브 파일 웹 에디터/컨버터** — 업로드 → 파싱 → 편집 → 다운로드 (PHP 로 충분)
5. **데스크톱 런처/프론트엔드** — React + Electron/Tauri, 3Beans 바이너리를
   자식 프로세스로 실행. 라이브러리 관리·커버아트·통계 제공

### 12.4 추천

**5번(런처)** 이 가장 현실적이다.
React 로 UI 를 만들고 Tauri 또는 Electron 으로 감싸서 3Beans 를 외부 프로세스로 실행.
기존 웹 개발 역량을 그대로 활용하면서 결과물은 진짜 데스크톱 앱이 되고,
**별도 프로세스 실행이므로 GPL 전염도 피할 수 있다.**

---

## 13. 수익화 아이디어 (상세)

> 전제: 3Beans 는 **GPL-3.0**. 소스를 사용·수정하면 소스 공개 및 GPL 승계 의무가 있다.
> 따라서 "소스 비공개 유료 판매"는 불가능하다. 또한 롬/부트롬 배포는 불법이다.
> 아래는 모두 이 선을 지킨 아이디어다.

### Tier S — 실현 가능성과 수익성이 모두 높은 것

#### S-1. 교육 콘텐츠 제작 (추천도 ★★★★★)

3.5만 줄 코드를 교육 자산으로 전환한다.

| 채널 | 콘텐츠 | 수익 구조 |
|---|---|---|
| YouTube | "에뮬레이터로 배우는 C++" 시리즈 | 광고 + 멤버십 |
| 블로그 | 코드 분석 연재 | 애드센스 + 협찬 |
| Inflearn / Udemy | "C++ 실전: 에뮬레이터로 배우는 시스템 프로그래밍" | 강의 판매 |
| 전자책 | 아키텍처 해설서 | 권당 판매 |

**근거**
- 국내에 "에뮬레이터 개발" 한국어 자료가 거의 없다
- CS 전공자, 게임 개발 지망생, 시스템 프로그래머 수요가 명확
- 코드를 **설명**하는 것은 GPL 위반이 아니다 (위험 0)
- 초기 투자 = 시간뿐

**커리큘럼 초안**

1. 에뮬레이터란 무엇인가 / HLE vs LLE
2. 이벤트 스케줄러 설계 (`core.cpp`)
3. CPU 인터프리터 만들기 (`arm/`)
4. 메모리 맵을 O(1) 로 (`memory/`)
5. GPU 렌더러 2종과 전략 패턴 (`gpu/`)
6. GitHub Actions 로 3개 OS 자동 배포 (`autobuild.yml`)

#### S-2. 데스크톱 런처 앱 (추천도 ★★★★★)

"3Beans 는 엔진, 내가 파는 것은 사용 경험."

```
[React + Tauri/Electron 런처]
 ├─ 게임 라이브러리 (커버아트 자동 수집)
 ├─ 플레이 시간 / 실적 통계
 ├─ 게임별 설정 프로파일 관리
 ├─ 세이브 백업 + 클라우드 동기화
 ├─ 여러 에뮬레이터 통합 관리
 └─ 3Beans 바이너리를 자식 프로세스로 실행
```

**핵심**: 3Beans 를 **별도 프로세스로 실행**하면 라이브러리 링크가 아니므로
GPL 전염을 피할 수 있다 → 런처는 독자 라이선스 가능.

**수익 모델**: 무료 + Pro 구독(클라우드 동기화 등) 또는 일회성 구매
**선례**: Playnite, LaunchBox(유료판 존재), Pegasus Frontend

기존 React 역량을 100% 활용 가능하며 포트폴리오 가치도 높다.

#### S-3. 후원 / 스폰서 모델 (추천도 ★★★★)

원작자가 이미 채택한 방식 (`FUNDING.yml`: Patreon, PayPal).

포크해서 독자 기능을 추가하고 후원을 받는 경로:

- 치트 엔진 / 액션 리플레이 코드 지원
- 넷플레이 (로컬 통신 온라인화)
- 세이브 스테이트 + 되감기(rewind)
- AI 텍스처 업스케일링
- **안드로이드 포팅** (수요가 가장 큼)

채널: GitHub Sponsors, Patreon, Ko-fi, Buy Me a Coffee
후원자 혜택: 얼리 빌드, 전용 Discord, 기능 우선순위 투표

**주의**: 소스는 GPL 로 계속 공개해야 한다.
"코드는 무료, 편의와 지속성에 후원"이 올바른 프레이밍이다.

### Tier A — 가능하지만 노력이 필요한 것

#### A-1. 주변 웹 서비스 (추천도 ★★★)

| 아이디어 | 스택 | 수익 |
|---|---|---|
| 호환성 데이터베이스 | PHP/Laravel + React | 광고 + 제휴 |
| 세이브 컨버터/에디터 | PHP | 무료 N회 + 구독 |
| 설정 프리셋 마켓플레이스 | React + API | 광고 + 프리미엄 |
| 벤치마크 리더보드 | React + DB | 하드웨어 제휴 링크 |

수익: 애드센스 + 쿠팡파트너스/아마존 제휴 (게임패드, PC 부품 추천)

#### A-2. 기술 컨설팅 / 프리랜싱 (추천도 ★★★)

이 코드베이스를 이해하고 기여할 수준이 되면:

- 레트로 게임 이식 외주 (인디 퍼블리셔 수요 존재)
- 임베디드/시스템 프로그래밍 포지션 이직
- 하드웨어 리버스 엔지니어링 용역

"에뮬레이터 개발 경험"은 게임/시스템 업계에서 강력한 시그널이다.

#### A-3. 하드웨어 번들 (추천도 ★★)

라즈베리파이/미니PC + 3Beans 사전 설치 + 케이스 + 컨트롤러 키트 판매.

**필수 조건**: GPL 이므로 사용한 소스/수정본을 함께 제공해야 한다.
롬·부트롬은 절대 포함 금지.

### 절대 하면 안 되는 것

| 금지 | 이유 |
|---|---|
| 소스 비공개 + 유료 판매 | GPL-3.0 위반 |
| boot9 / boot11 / nand 배포 | 저작권 침해 |
| 롬 포함 배포 | 명백한 불법 |
| 앱스토어 유료 등록 | GPL 과 스토어 약관 충돌 소지 |
| 원작자 크레딧 제거 | GPL 위반 + 커뮤니티 신뢰 상실 |

> 참고: Yuzu(스위치 에뮬레이터)는 닌텐도 소송으로 240만 달러 합의 후 폐쇄되었고,
> 그 여파로 Citra 도 함께 중단되었다. 선을 넘지 않는 것이 최우선이다.

### 권장 로드맵 (웹 개발자 기준)

```
1단계 (0~6개월, 투자 0원)
  → 블로그/유튜브로 코드 분석 연재
  → 목표: 개인 브랜딩 + 청중 확보

2단계 (6~18개월, React 역량 활용)
  → Tauri + React 런처 앱 개발
  → 무료 배포로 사용자 확보 → Pro 기능 유료화

3단계 (18개월~)
  → 온라인 강의 출시 + 기술 컨설팅
  → 부수입에서 본업 전환 가능성
```

**핵심 철학**

> 에뮬레이터 자체를 파는 것이 아니라,
> **에뮬레이터 주변의 편의·지식·경험을 판다.**

GPL 을 건드리지 않고, 법적 리스크도 없으며, 기존 웹 개발 역량을 그대로 활용할 수 있다.

---

## 14. 법적·라이선스 주의사항

### GPL-3.0 핵심 의무

1. 소스 코드를 사용·수정해 배포하면 **전체 소스를 GPL-3.0 으로 공개**해야 한다
2. 원저작권 표시와 라이선스 고지를 유지해야 한다
3. 변경 사항을 명시해야 한다
4. 정적/동적 링크 모두 GPL 전염 대상이다
   → **별도 프로세스 실행(exec)** 은 일반적으로 전염되지 않는다

### 저작권 관련

- `boot9.bin`, `boot11.bin`, `nand.bin` 은 **닌텐도 저작물**이다. 배포 불법
- 게임 롬 배포 불법. 본인 소유 카트리지의 개인 백업만 (국가별 법률 확인 필요)
- 에뮬레이터 **자체의 개발·배포는 합법**이다 (Sony v. Connectix, Sega v. Accolade 판례)
- 단, 우회 기술·암호키를 포함하면 DMCA 문제가 발생할 수 있다

---

## 15. 참고 링크 모음

### 저장소

- **이 저장소 (포크)**: https://github.com/bmshin94/3Beans
- **원본 저장소**: https://github.com/Hydr8gon/3Beans
- **릴리즈 (빌드 다운로드)**: https://github.com/Hydr8gon/3Beans/releases
- **이슈 트래커**: https://github.com/Hydr8gon/3Beans/issues

### 개발자 채널

- 개발자 블로그 (Hydra's Lair): https://hydr8gon.github.io
- Discord 서버: https://discord.gg/JbNz7y4
- Patreon: https://patreon.com/Hydr8gon
- PayPal: https://paypal.me/Hydr8gon

### 기술 레퍼런스 (README 기재)

- **GBATEK**: https://problemkaputt.de/gbatek.htm — 3DS 하드웨어 레퍼런스
- **3DBrew**: https://www.3dbrew.org — 고·저수준 문서 위키
- **libctru**: https://github.com/devkitPro/libctru — 홈브류 라이브러리
- **Teakra**: https://github.com/wwylele/teakra — Teak DSP 아키텍처 자료
- **Corgi3DS**: https://github.com/PSI-Rockin/Corgi3DS — 최초의 3DS LLE 에뮬레이터

### 도구 / 의존성

- **GodMode9** (덤프 추출): https://github.com/d0k3/GodMode9
- SD 이미지 샘플: https://kuribo64.net/get.php?id=mRJJ5GggXOPbKUMZ
- wxWidgets: https://www.wxwidgets.org
- PortAudio: https://www.portaudio.com
- libepoxy: https://github.com/anholt/libepoxy
- MSYS2 (Windows 빌드): https://www.msys2.org
- Homebrew (macOS): https://brew.sh

### 관련 프로젝트 (참고용)

- NooDS (같은 개발자의 NDS 에뮬레이터): https://github.com/Hydr8gon/NooDS
- EmulatorJS (WASM 포팅 선례): https://github.com/EmulatorJS/EmulatorJS
- Playnite (런처 선례): https://github.com/JosefNemec/Playnite

---

## 요약 한 장

| 질문 | 답 |
|---|---|
| 뭐하는 건가? | 3DS 를 칩 단위로 재현하는 저수준 에뮬레이터 (C++, 3.5만 줄) |
| 언제 쓰나? | 게임 보존 / 하드웨어 연구 / 정확도 기준점 / 홈브류 테스트 / 학습 |
| 내게 도움? | 실행용보다 **C++ 아키텍처 교과서 + CI 배포 템플릿**으로서의 가치가 큼 |
| 설치법 | 릴리즈 다운로드 또는 `make`. **단, 실기 덤프 파일 필수** |
| 플러그인/스킬/MCP? | 전부 아님. 독립 데스크톱 앱. (`CLAUDE.md` 만 이 포크의 추가물) |
| API 토큰? | 불필요. 완전 오프라인 |
| 왜 유명? | 사실상 유일한 현역 3DS LLE + 개발자 명성 + Citra 공백기 타이밍 + 3년 꾸준함 |
| 에이전트에 도움? | 직접 무관, 스케줄러·전략패턴·워커 구조 학습엔 매우 유용 |
| React/PHP 가능? | 본체 불가(PHP 0%, WASM 경유 30%). 런처·웹툴은 충분히 가능 |
| 수익화 | 교육 콘텐츠 → React 런처 앱 → 강의/컨설팅 3단계. GPL·저작권 선 준수 필수 |
