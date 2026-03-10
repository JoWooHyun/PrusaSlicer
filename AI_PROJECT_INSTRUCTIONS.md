# 하이브리드 멀티레진 3D 프린터 커스텀 슬라이서 - AI 어시스턴트 지침

> 이 문서를 Claude 웹/ChatGPT 등의 "프로젝트 지침" 또는 시스템 프롬프트에 넣으세요.

---

## 프로젝트 한 줄 요약

PrusaSlicer(v2.9.4, C++17, AGPLv3)를 포크하여 **Top-Down 하이브리드 멀티레진 3D 프린터** 전용 커스텀 슬라이서를 개발한다.

---

## 1. 프린터 방식 (반드시 이해해야 함)

이 프린터는 **일반 FDM도 SLA도 아닌 하이브리드 방식**이다.

### 핵심 개념
- **도포 경로(GCode) = 재료 공급만** 담당 (정밀도 낮아도 됨)
- **마스크 이미지(PNG) = 최종 형상** 결정 (정밀도 핵심)
- 따라서 **레진 슬라이서가 몸통**이고, FDM 도포 기능은 추가 기능이다

### 레이어 공정 시퀀스 (5단계, 모든 레이어 동일)
```
for each layer:
    1. T0 deposition     # Tool0 노즐로 모델레진 도포 (GCode)
    2. T1 deposition     # Tool1 노즐로 템프레진 도포 (GCode)
    3. Blade leveling    # 블레이드로 레진 평탄화 (GCode)
    4. LED exposure      # DLP 마스크 이미지 조사 + 경화 대기 (PNG + G4)
    5. Z lift            # 다음 레이어 높이로 이동 (GCode)
```

### 멀티 레진 구조
```
Object (STL 파일)
  → Material (재료 ID)
    → Tool (도포 노즐)
      → Nozzle (물리적 하드웨어)

예시 (치과 모델):
  잇몸_모델.stl  → Material 0 (모델레진) → Tool 0 → Nozzle 0
  치아_21.stl    → Material 1 (템프레진) → Tool 1 → Nozzle 1
  치아_22.stl    → Material 1 (템프레진) → Tool 1 → Nozzle 1
```

### 마스크 이미지
- 재료 구분 없이 **전체 합쳐서 1장** PNG
- 흰색 = LED 켜짐 = 경화, 검은색 = LED 꺼짐 = 미경화
- 어디에 어떤 레진을 뿌릴지는 **GCode가 담당**

### 가정 (이상적 시나리오)
- 두 레진의 경화 시간 동일
- 블레이드 평탄화 시 레진 혼합 없음
- 도포 영역 정확히 분리됨
- 물리적 문제는 하드웨어/재료가 해결. 소프트웨어는 이상적으로 동작한다고 전제

---

## 2. 사용 워크플로우

```
[외부 CAD]           [PC]                      [USB]          [프린터]
exocad 등에서     커스텀 슬라이서에서        USB에 저장      vgui가 파일을
STL 파일 준비 → 로드 + Material 지정  →  ZIP 파일  →  읽어서 프린팅
(좌표 확정)       슬라이스                               실행
```

- **슬라이서는 PC 전용 데스크탑 앱** (프린터와 직접 통신 안 함, 오프라인 USB 전달)
- STL은 외부 CAD(exocad 등)에서 좌표가 이미 확정된 상태로 들어옴
- 슬라이서는 **STL을 수정하지 않고** 그대로 슬라이스
- STL 파일들은 서로 겹칠 수 있음 (예: 잇몸 구멍 안에 치아)
- Boolean 연산 등 복잡한 메시 연산은 피한다

---

## 3. 하드웨어

| 구분 | 사양 |
|------|------|
| 보드 | BigTreeTech Manta M8P 2.0 |
| 컴퓨트 | Raspberry Pi CM4 |
| 펌웨어 | Klipper + Moonraker |
| 프린터 GUI | vgui (PySide6, 자체 개발) |
| 프로젝터 | DLP LED 프로젝터 (마스크 경화) |
| 도포 | 공기압 디스펜서 + 듀얼 노즐 (T0, T1) |
| 블레이드 | 레진 평탄화용 (편도/왕복) |
| 현재 슬라이서 | 치투슬라이서 (교체 예정) |

---

## 4. 슬라이서 출력 포맷

```
print_job.zip
├── run.gcode              ← 도포/블레이드/Z이동 (Klipper)
├── layers/
│   ├── 0000.png           ← 레이어 마스크 (흰=경화, 검=미경화)
│   ├── 0001.png
│   └── ...
├── preview.png            ← 미리보기 썸네일
├── flat_field_mask.png    ← Flat Field 보정 (선택)
└── manifest.json          ← 메타데이터
```

### manifest.json 주요 필드
```json
{
  "layer_height": 0.05,
  "total_layers": 1200,
  "exposure_time": 2.5,
  "first_layer_exposure_time": 8.0,
  "resolution_x": 2560,
  "resolution_y": 1440,
  "pixel_size_um": 50,
  "tools": [
    {
      "id": 0, "name": "Model Resin",
      "nozzle_offset_x": 0.0, "nozzle_offset_y": 0.0,
      "nozzle_diameter": 0.4,
      "dispense_pressure": 300,
      "dispense_start_delay_ms": 50,
      "dispense_stop_delay_ms": 30
    },
    {
      "id": 1, "name": "Temp Resin",
      "nozzle_offset_x": 35.0, "nozzle_offset_y": 0.0,
      "nozzle_diameter": 0.4,
      "dispense_pressure": 280,
      "dispense_start_delay_ms": 60,
      "dispense_stop_delay_ms": 40
    }
  ],
  "blade": {
    "speed": 10.0,
    "direction": "one_way",
    "pass_count": 1,
    "height": 0.0
  }
}
```

---

## 5. 하드웨어 제어 파라미터

### 듀얼 노즐 오프셋
두 노즐은 물리적으로 다른 위치. 슬라이서가 GCode 좌표 보정 필요.
- `nozzle_offset_x/y/z` (T1 기준, T0은 원점)

### 디스펜서 제어 (공기압)
FDM의 E축(스테퍼)이 아닌 공기압 디스펜서. 시작/정지 지연 발생.
- `resin_dispense_start_delay` (ms)
- `resin_dispense_stop_delay` (ms)
- `resin_pressure` (kPa)
- `resin_nozzle_diameter` (mm)

### 블레이드 제어
프린터 핵심 메커니즘. 편도/왕복 설정 가능.
- `blade_speed` (mm/s)
- `blade_direction` (one_way / round_trip)
- `blade_pass_count` (회)
- `blade_height` (mm)

### LED/프로젝터
- `exposure_time` / `first_layer_exposure_time` (초)
- `resolution_x/y` (px)
- `pixel_size` (um)
- Flat Field 보정: 프로젝터 밝기 불균일 보정 (후순위)

---

## 6. PrusaSlicer 아키텍처 핵심

### 디렉토리 구조
```
PrusaSlicer/
├── src/libslic3r/          # 핵심 슬라이싱 엔진
│   ├── GCode/              # GCode 생성
│   ├── Fill/               # 인필 패턴 → 도포 경로로 활용 가능
│   ├── SLA/                # ★ 레이어 이미지 생성 (RasterBase)
│   ├── Format/             # ★ 출력 포맷 (SL1 = ZIP+PNG)
│   ├── Support/            # 서포트
│   ├── Arachne/            # 벽 생성 (도포에는 불필요)
│   └── Config.hpp          # PrinterTechnology enum
├── src/slic3r/GUI/         # wxWidgets GUI
├── deps/                   # 외부 의존성 28개
└── resources/profiles/     # 프린터 프리셋
```

### 핵심 파일 (코드 분석 우선순위)
| 순서 | 파일 | 역할 |
|------|------|------|
| 1 | Config.hpp | PrinterTechnology enum (ptFFF, ptSLA → ptHybrid 추가) |
| 2 | PrintConfig.hpp | 300+ 파라미터 정의 방법 |
| 3 | Print.cpp | 전체 슬라이싱 파이프라인 |
| 4 | SLA/RasterBase.* | ExPolygon → PNG 래스터화 (마스크 생성) |
| 5 | Format/SL1.* | ZIP 패키징 (출력 포맷 참고) |
| 6 | GCode.cpp + GCodeWriter.* | GCode 생성/작성 |
| 7 | PrintObject.cpp | 객체별 슬라이싱 흐름 |
| 8 | PrintObjectSlice.cpp | 겹침 처리 로직 |

### 슬라이싱 파이프라인
```
STL/3MF 로드 → Z 슬라이싱 → ExPolygons
  → 마스크 이미지 (PNG, 합쳐서 1장)   ← SLA의 RasterBase 참고
  → Tool별 도포 경로 (GCode)          ← FDM의 Fill/GCode 참고
  → 블레이드/Z이동 GCode
  → ZIP 패키징                         ← SL1Archive 참고
```

### PrusaSlicer에서 활용할 부분
**SLA에서 (몸통):** 3D 뷰어, Z 슬라이싱, RasterBase(PNG), SLA 서포트, SL1 ZIP, Zipper
**FDM에서 (추가):** 멀티 익스트루더→멀티 Tool, 인필→도포 경로, GCodeWriter(gcfKlipper)

### 제거할 FDM/Prusa 전용 기능
- Prusa 계정 로그인, PrusaConnect, Configuration Sources, 프리셋 자동 업데이트
- 온도 제어, 필라멘트 설정, 냉각 팬, Extrusion Multiplier, 리트랙션
- 벽(Perimeter) 정밀 생성 (Arachne), 아이로닝, 와이프 타워
- 기존 프린터 프로파일 36개

### 주요 외부 의존성
| 라이브러리 | 용도 |
|-----------|------|
| Clipper | 2D 폴리곤 Boolean (union, diff) |
| CGAL | 3D 메시 Boolean, self-intersection |
| Eigen3 | 행렬/벡터 연산 |
| TBB | 병렬 처리 |
| Boost | 파일/로깅/스레드 |
| wxWidgets | GUI |
| libpng/zlib | PNG/ZIP |

---

## 7. 참고 오픈소스

| 프로젝트 | 역할 | GitHub |
|---------|------|--------|
| LumenSlicer | PrusaSlicer 포크 레진 슬라이서 | Open-Resin-Alliance/LumenSlicer |
| Odyssey | SL1 파일 처리 + Klipper 통신 | Open-Resin-Alliance/Odyssey |
| Orion | Flutter 프린터 GUI | Open-Resin-Alliance/Orion |

차이: 이들은 순수 mSLA(도포 없음). 출력 포맷/이미지 생성/Klipper 통신 구조만 참고.

---

## 8. vgui (프린터 측)

- PySide6 기반 Python GUI, Raspberry Pi에서 실행
- USB에서 ZIP 읽기 → GCode를 Moonraker로 전달 → 마스크 PNG를 프로젝터에 표시
- 슬라이서 출력 포맷이 바뀌면 vgui의 **파일 파싱 부분만** 수정 (핵심 로직 유지)
- vgui는 PrusaSlicer 코드를 사용하지 않으므로 AGPLv3가 전파되지 않음 (별도 프로그램)

---

## 9. 개발 Phase

| Phase | 내용 |
|-------|------|
| -1 | PrusaSlicer 사용법 학습 (FDM/SLA 모드, 멀티 익스트루더) |
| 0 | 빌드 환경 구축 (C++17, CMake, deps) |
| 1 | 불필요 기능 제거 + 브랜딩 |
| 2 | 멀티 레진 Tool 매핑 (Object→Material→Tool→Nozzle) |
| 3 | 마스크 이미지 PNG 생성 (RasterBase) |
| 4 | 도포 GCode 생성 (노즐 오프셋 + 디스펜서 + 블레이드) |
| 5 | ZIP 패키징 + manifest.json (SL1Archive) |
| 6 | vgui 연동 + 실 프린트 테스트 |

---

## 10. 라이선스

- PrusaSlicer는 **AGPLv3**
- 포크한 코드도 AGPLv3 (파생저작물)
- **내부용 사용: 소스 공개 의무 없음**
- **외부 배포/판매: 수정한 전체 소스코드 공개 필요** (신규 코드 포함)
- vgui는 별도 프로그램이므로 AGPLv3 전파 안 됨

---

## 질문/분석 요청 시 참고

- 파일을 업로드하면 해당 파일을 분석하고, 이 프린터 방식에 맞게 수정 포인트를 찾아줘
- 질문할 때는 구체적인 파일명과 함수명을 알려주면 됨
- **레진 슬라이서가 몸통**이라는 관점에서 답변해줘
- FDM 전용 기능(온도, 냉각, 리트랙션 등)은 이 프로젝트에 불필요
- 도포 경로는 대략적 공급 수준이며, 정밀한 벽(Perimeter) 생성은 불필요
