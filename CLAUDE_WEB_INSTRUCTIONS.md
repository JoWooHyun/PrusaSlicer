# 커스텀 슬라이서 개발 프로젝트 - 웹 클로드 분석 지침

## 프로젝트 개요

PrusaSlicer(v2.9.4) 소스코드를 기반으로 커스텀 슬라이서를 개발하는 프로젝트.
**특수한 프린팅 방식**: FDM처럼 XYZ 축 이동을 사용하지만, 필라멘트 대신 고점도 레진을 도포하는 독자적인 방식.

### 실제 사용 워크플로우
```
[PC (Windows)]                    [USB]              [프린터 (Raspberry Pi)]
커스텀 슬라이서에서              USB에 저장           vgui가 파일을 읽고
3D 모델을 슬라이싱    →  슬라이싱 결과 파일  →   Moonraker로 모터 제어
                                                  + 프로젝터로 LED 조사
```
**슬라이서는 PC 전용 데스크탑 앱**이며, 프린터와 직접 통신하지 않는다.
슬라이싱 결과물을 USB에 담아 프린터에 물리적으로 전달하는 오프라인 방식.

### 프린팅 2단계 프로세스 (하이브리드 방식)
이 프린터는 FDM과 SLA를 결합한 독자적인 방식:
1. **레진 도포 (FDM식)**: XYZ축 이동으로 고점도 레진을 출력 위치에 뿌림 → **GCode 필요**
2. **LED 경화 (SLA식)**: 도포된 레진의 특정 부분에 LED를 조사하여 경화 → **레이어 이미지(PNG) 필요**
→ 따라서 출력 파일에는 **GCode(모터 경로) + 레이어별 이미지(LED 패턴) 모두 포함**되어야 함.

### 하드웨어 환경
- **보드**: BigTreeTech Manta M8P 2.0
- **컴퓨트**: Raspberry Pi CM4
- **펌웨어**: Klipper + Moonraker
- **프린터 GUI**: PySide6 기반 자체 Python GUI (웹 GUI 미사용)
  - 참조: https://github.com/JoWooHyun/vgui
  - vgui는 프린터 측에서 실행되며, USB의 슬라이싱 파일을 읽어 프린팅을 실행
- **현재 사용 슬라이서**: 치투슬라이서 (교체 대상)

### 개발 목표
PrusaSlicer를 기반으로 전체적인 커스터마이징:
1. 슬라이싱 엔진 수정 (레진 도포 방식에 최적화)
2. GCode 생성 커스터마이징 (Klipper 호환, 레진 도포 경로)
3. 레이어별 LED 조사 이미지(PNG) 생성 기능 추가
4. 출력 파일 포맷 정의 (GCode + 레이어 이미지를 포함하는 패키지)
5. UI/UX 개선
6. 독자 기능 추가

### 출력 파일 형식 (미확정, 설계 필요)
하이브리드 방식이므로 기존 슬라이서 포맷과 다른 새로운 형식이 필요할 수 있음:
- GCode: 레진 도포를 위한 XYZ 이동 경로 (Klipper 호환)
- 레이어 이미지: LED 경화를 위한 레이어별 PNG (SLA 방식 참고)
- 메타데이터: 레이어 높이, 노출 시간, 레진 도포 파라미터 등
- 치투슬라이서의 ZIP(run.gcode + 레이어 PNG) 형식을 참고할 수 있음

---

## PrusaSlicer 아키텍처 요약

### 디렉토리 구조
```
PrusaSlicer/
├── src/
│   ├── libslic3r/          # ★ 핵심 슬라이싱 엔진 (C++ 라이브러리)
│   │   ├── GCode/          # GCode 생성/처리
│   │   ├── Fill/           # 인필 패턴 (8+ 종류)
│   │   ├── Support/        # 서포트 생성 (Tree/Organic/Traditional)
│   │   ├── SLA/            # SLA(레진) 프린터 처리
│   │   ├── Arachne/        # 벽(Perimeter) 생성 알고리즘
│   │   ├── Format/         # 파일 형식 (STL,OBJ,3MF,STEP 등)
│   │   ├── Geometry/       # 기하학 연산
│   │   ├── Algorithm/      # 경로 정렬, 영역 확장 등
│   │   ├── Feature/        # FuzzySkin, Interlocking 등
│   │   ├── CSGMesh/        # 구성적 고체 기하학
│   │   ├── Execution/      # 병렬 처리 (TBB)
│   │   ├── Optimize/       # 수치 최적화
│   │   └── Utils/          # 유틸리티
│   ├── slic3r/
│   │   └── GUI/            # wxWidgets 기반 GUI
│   ├── CLI/                # 명령줄 인터페이스
│   ├── clipper/            # 다각형 연산
│   ├── libvgcode/          # GCode 벡터 처리
│   └── libseqarrange/      # 객체 배치 알고리즘
├── deps/                   # 외부 의존성 빌드 (28개 라이브러리)
├── bundled_deps/           # 번들 의존성
├── resources/
│   └── profiles/           # 프린터 프리셋 (36개)
├── tests/                  # 단위 테스트 (Catch2)
└── CMakeLists.txt          # 빌드 설정 (C++17, CMake 3.13+)
```

### 슬라이싱 파이프라인 (핵심)

```
STL/OBJ/3MF 로드
    ↓
Model → TriangleMesh 변환
    ↓
[posSlice] TriangleMeshSlicer::slice_mesh_ex()
    → 3D 메시를 Z 평면으로 절단 → ExPolygons 생성
    ↓
[posPerimeters] PerimeterGenerator::process()
    → Arachne(SkeletalTrapezoidation) 알고리즘으로 벽 생성
    ↓
[posPrepareInfill] 서피스 감지, 브릿지 감지, 쉘 결합
    ↓
[posInfill] Fill::fill_surface()
    → FillRectilinear/Honeycomb/Gyroid 등 패턴 적용
    ↓
[posIroning] 상단 면 아이로닝
    ↓
[posSupportMaterial] TreeSupport/SupportMaterial
    → 서포트 구조 생성
    ↓
[posEstimateCurledExtrusions] 말림 추정
    ↓
Print::_make_wipe_tower() / _make_skirt() / _make_brim()
    ↓
GCodeGenerator::do_export()
    → GCodeWriter로 명령어 작성
    → CoolingBuffer, SpiralVase, PressureEqualizer 등 후처리
    → 최종 GCode 파일 출력
```

### 핵심 클래스 관계

```
Print (프린트 작업 관리)
 ├─ PrintObject[] (프린트 객체)
 │   ├─ Layer[] (레이어)
 │   │   └─ LayerRegion[] (영역)
 │   │       ├─ perimeters: ExtrusionEntityCollection
 │   │       ├─ fills: ExtrusionEntityCollection
 │   │       └─ slices: SurfaceCollection
 │   ├─ SupportLayer[] (서포트 레이어)
 │   └─ PrintObjectRegions → PrintRegion[]
 ├─ GCodeGenerator (GCode 생성)
 │   ├─ GCodeWriter (명령어 작성)
 │   ├─ CoolingBuffer (냉각 제어)
 │   ├─ Seams::Placer (이음새 위치)
 │   └─ AvoidCrossingPerimeters (경로 최적화)
 └─ PrintConfig (300+ 설정 파라미터)
```

### GCode Flavor 설정
PrusaSlicer는 `gcfKlipper`를 포함한 다양한 GCode flavor를 지원:
```cpp
enum class GCodeFlavor {
    gcfRepRapSprinter, gcfRepRapFirmware, gcfRepetier,
    gcfTeacup, gcfMakerWare, gcfMarlinLegacy, gcfMarlinFirmware,
    gcfKlipper,  // ★ Klipper 지원
    gcfLerdge, gcfSmoothie, gcfMach3, gcfSailfish, ...
};
```

### 주요 외부 의존성
| 라이브러리 | 용도 |
|-----------|------|
| **Boost** (1.83+) | 시스템, 파일, 스레드, 로깅 |
| **Eigen3** (3.3.7+) | 선형대수/행렬 |
| **TBB** | 병렬 처리 |
| **CGAL** | 계산 기하학 |
| **OpenVDB** (5.0+) | 3D 복셀 처리 |
| **wxWidgets** (3.2+) | GUI |
| **NLopt** | 수치 최적화 |
| **LibBGCode** | BGCode 포맷 |

---

## vgui 프로젝트 참조 (프린터 측 소프트웨어)

vgui는 **프린터 측 라즈베리파이에서 실행**되는 Python/PySide6 기반 GUI.
슬라이서와는 별개의 프로그램이며, USB를 통해 전달받은 파일을 읽어 프린팅을 실행.

**vgui의 역할:**
- USB에서 슬라이싱 파일 읽기
- GCode를 Moonraker API로 전달 → Klipper가 모터 제어 (레진 도포)
- 레이어 이미지를 프로젝터에 표시 → LED 경화
- 현재 ZIP 형식(run.gcode + 레이어 PNG + 미리보기) 사용

**커스텀 슬라이서의 출력 포맷은 vgui가 읽을 수 있어야 함.**
→ vgui의 파일 파싱 로직을 확인하고 호환되는 형식으로 출력하거나, 양쪽을 함께 수정해야 함.

---

## 분석 시 주의사항

1. **이 프로젝트는 일반 FDM도 SLA도 아닌 하이브리드 방식**
   - FDM 방식의 XYZ 이동으로 고점도 레진을 도포 (필라멘트 X, 레진 O)
   - SLA 방식처럼 LED를 조사하여 도포된 레진을 경화
   - 따라서 **FDM의 GCode 경로 생성 + SLA의 레이어 이미지 생성** 두 가지 모두 필요

2. **슬라이서는 PC 전용 데스크탑 앱 (오프라인 방식)**
   - PC에서 슬라이싱 → USB에 파일 저장 → 프린터에 USB 삽입 → vgui가 읽어서 실행
   - 슬라이서와 프린터 간 네트워크 연동 없음
   - 슬라이서 출력 포맷은 vgui가 파싱할 수 있어야 함

3. **Klipper 호환 필수**
   - GCode flavor는 `gcfKlipper` 기반
   - vgui가 GCode를 Moonraker API로 전달

4. **GUI는 wxWidgets 기반이지만 변경 가능성 있음**
   - 슬라이서 GUI(PC): wxWidgets (PrusaSlicer 기본)
   - 프린터 GUI(라즈베리파이): PySide6 (vgui, 별도 프로젝트)
   - 이 둘은 별개의 앱

5. **코드 규모가 매우 큼**
   - libslic3r만 수백 개 소스 파일
   - 한 번에 전체를 다룰 수 없으므로 모듈별 분석 필요

---

## 웹 클로드에서의 분석 방법 제안

### Phase 1: 핵심 파이프라인 이해
아래 파일들을 순서대로 분석:
1. `src/libslic3r/Print.hpp` + `Print.cpp` → 전체 프린트 프로세스
2. `src/libslic3r/PrintObject.cpp` → 객체별 슬라이싱
3. `src/libslic3r/PrintObjectSlice.cpp` → 메시→레이어 변환
4. `src/libslic3r/Layer.hpp` + `Layer.cpp` → 레이어 구조
5. `src/libslic3r/GCode.hpp` + `GCode.cpp` → GCode 생성

### Phase 2: 인필/서포트/벽 생성
6. `src/libslic3r/Fill/FillBase.hpp` → 인필 패턴 기본 구조
7. `src/libslic3r/PerimeterGenerator.hpp` → 벽 생성
8. `src/libslic3r/Support/SupportMaterial.hpp` → 서포트

### Phase 3: GCode 상세
9. `src/libslic3r/GCode/GCodeWriter.hpp` → GCode 명령어 작성
10. `src/libslic3r/GCode/CoolingBuffer.hpp` → 냉각 제어
11. `src/libslic3r/PrintConfig.hpp` → 300+ 설정 파라미터

### Phase 4: 설정/프리셋 시스템
12. `src/libslic3r/Preset.hpp` → 프리셋 관리
13. `src/libslic3r/PresetBundle.hpp` → 프리셋 번들
14. `resources/profiles/` → 기존 프린터 프로파일 참조

### Phase 5: 수정 포인트 식별
- GCode 생성 커스터마이징 위치
- 새로운 프린터 프로파일 추가 방법
- 출력 포맷 (ZIP) 추가 방법
- UI 수정 진입점

---

## 대화 시작 프롬프트 (웹 클로드에 복사해서 사용)

아래 내용을 웹 클로드의 "프로젝트 지침" 또는 첫 메시지로 사용하세요:

---

```
나는 PrusaSlicer(v2.9.4, C++17)를 기반으로 커스텀 슬라이서를 개발하고 있다.

## 내 프린터의 특수성
- FDM처럼 XYZ축 이동을 하지만, 필라멘트 대신 고점도 레진을 도포하는 방식
- 도포된 레진에 LED를 조사하여 경화하는 하이브리드 방식 (FDM 이동 + SLA 경화)
- 따라서 슬라이서는 **GCode(도포 경로) + 레이어 이미지(LED 패턴)** 둘 다 생성해야 함

## 사용 워크플로우 (중요!)
1. PC(Windows)에서 이 커스텀 슬라이서로 3D 모델을 슬라이싱
2. 슬라이싱 결과 파일을 USB에 저장
3. USB를 프린터(Raspberry Pi)에 삽입
4. 프린터의 vgui(PySide6 GUI)가 파일을 읽어서 프린팅 실행
   - GCode → Moonraker API → Klipper로 모터 제어 (레진 도포)
   - 레이어 이미지 → 프로젝터로 LED 조사 (레진 경화)
※ 슬라이서는 PC 전용 데스크탑 앱이며, 프린터와 직접 네트워크 통신하지 않음 (오프라인 USB 전달)

## 하드웨어
- 보드: BigTreeTech Manta M8P 2.0
- 컴퓨트: Raspberry Pi CM4
- 펌웨어: Klipper + Moonraker
- 프린터 GUI: vgui (PySide6, https://github.com/JoWooHyun/vgui)
- 현재 슬라이서: 치투슬라이서 (교체 예정)

## 프로젝트 목표
1. 슬라이싱 엔진을 레진 도포 방식에 맞게 수정
2. Klipper 호환 GCode 생성 (레진 도포 경로용)
3. 레이어별 LED 조사 이미지(PNG) 생성 기능 추가
4. 출력 파일 포맷 설계 (GCode + 레이어 이미지를 포함하는 패키지, vgui가 읽을 수 있는 형식)
5. UI/UX 전면 커스터마이징
6. 독자적 기능 추가

## PrusaSlicer 핵심 구조 요약

### 슬라이싱 파이프라인
STL 로드 → TriangleMeshSlicer::slice_mesh_ex() → ExPolygons
→ PerimeterGenerator (벽) → Fill (인필) → Support (서포트)
→ GCodeGenerator::do_export() → GCode 출력

### 핵심 파일
- src/libslic3r/Print.hpp,cpp - 프린트 프로세스 관리
- src/libslic3r/PrintObject.cpp - 객체별 슬라이싱 (posSlice→posPerimeters→posInfill→posSupportMaterial)
- src/libslic3r/GCode.hpp,cpp - GCode 생성 엔진
- src/libslic3r/GCode/GCodeWriter.hpp,cpp - GCode 명령어 작성
- src/libslic3r/Fill/ - 인필 패턴 (Rectilinear, Honeycomb, Gyroid 등)
- src/libslic3r/Support/ - 서포트 (Tree, Organic, Traditional)
- src/libslic3r/Arachne/ - 벽 생성 (SkeletalTrapezoidation)
- src/libslic3r/PrintConfig.hpp - 300+ 설정 파라미터
- src/libslic3r/SLA/ - SLA 레진 처리 (레이어 이미지 생성 참고!)
- src/libslic3r/Format/ - 파일 형식 (출력 포맷 추가 시 참고)
  - SL1.hpp,cpp / SL1_SVG.hpp,cpp - SLA 출력 포맷 (ZIP + PNG 레이어, 가장 가까운 참고 대상)

### 특히 참고할 SLA 관련 코드
- src/libslic3r/SLA/RasterBase.hpp,cpp - 레이어를 래스터 이미지로 변환
- src/libslic3r/Format/SL1.hpp,cpp - ZIP 아카이브(GCode + 레이어 PNG) 출력
이 코드들이 우리 프린터의 "레이어 이미지 생성" 부분에 가장 가까운 참고 대상

### GCode Flavor
gcfKlipper 지원 내장. PrintConfig에서 gcode_flavor 설정.

### 빌드
C++17, CMake 3.13+, 주요 의존성: Boost, Eigen3, TBB, CGAL, OpenVDB, wxWidgets

## 요청사항
파일을 업로드하면 해당 파일을 분석하고, 내 프린터 방식에 맞게 수정할 포인트를 찾아줘.
질문할 때는 구체적인 파일명과 함수명을 알려주면 된다.
```
