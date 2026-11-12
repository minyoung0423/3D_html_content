[README.md](https://github.com/user-attachments/files/34724025/README.md)
# Three.js 기반 3D 디지털 트윈 웹 애플리케이션 분석 가이드

본 문서는 첨부된 두 개의 단일 HTML 파일(`ThreeJS_korean_bus.html`, `smart_classroom_camera_stable.html`)에 대한 **실행 방법**과 **전체적인 코드 및 문법 구조**를 분석하여 정리한 기술 문서입니다.

---

## 1. 대상 파일 개요

| 구분 | `ThreeJS_korean_bus.html` | `smart_classroom_camera_stable.html` |
| :--- | :--- | :--- |
| **프로젝트명** | 한국형 저상 전기 시내버스 스마트팩토리 디지털 트윈 | 스마트 학교 교실 디지털 트윈 (카메라 안정화 버전) |
| **Three.js 버전** | `r180` (ES Modules) | `r164.1` (ES Modules) |
| **주요 목적** | 8단계 버스 조립 공정 시뮬레이션 및 부품 구조 학습 | 교실 내 IoT 센서(온도, $CO_2$, 조도) 및 환경 설비 시뮬레이션 |
| **특이사항** | 레이캐스팅 부품 선택, X-RAY 투시, 분해 보기(Explode) 지원 | Blender 변환용 메타데이터(`#sceneSpec`) 내장, CSV/JSON 데이터 내보내기 지원 |

---

## 2. 실행 방법 (How to Run)

두 파일 모두 외부 별도 번들러(Webpack, Vite 등) 없이 브라우저에서 직접 실행 가능한 **단일 HTML(Single-File)** 구조로 작성되어 있으나, **ES Modules (`<script type="module">`)** 및 **CDN 네트워크 로드**를 사용하므로 아래 환경을 권장합니다.

### 2.1. 인터넷 연결 필수
* `<script type="importmap">`을 통해 `cdn.jsdelivr.net`에서 Three.js 코어 및 `OrbitControls` 애드온을 실시간으로 불러옵니다.
* 최초 실행 시 반드시 인터넷에 연결되어 있어야 합니다.

### 2.2. 로컬 웹 서버를 통한 실행 (권장)
보안 정책(CORS) 및 모듈 로딩 안정성을 위해 파일을 더블클릭(`file://`)하는 것보다 로컬 서버(`http://`)로 실행하는 것이 가장 안전합니다.

1. **VS Code Live Server 이용**
   * VS Code에서 해당 HTML 파일을 열고 우측 하단의 **`Go Live`** 클릭
2. **Python 내장 서버 이용**
   ```bash
   # HTML 파일이 위치한 폴더에서 터미널 실행
   python -m http.server 8000
   ```
   * 브라우저 접속: `http://localhost:8000/ThreeJS_korean_bus.html` 또는 `http://localhost:8000/smart_classroom_camera_stable.html`
3. **GitHub Pages 배포**
   * 리포지토리 루트에 파일들을 업로드하고 `Settings > Pages`에서 배포 활성화 시 웹 URL로 즉시 접근 가능

---

## 3. 전체적인 코드 및 문법 구조

두 파일 모두 **HTML + CSS + JavaScript(ES6+)**가 하나의 파일 안에 통합된 **모놀리식(Monolithic) 아키텍처**를 따르며, 크게 **4개의 계층**으로 구성됩니다.

### 3.1. 문서 메타데이터 및 Import Map (`<head>`)
* **반응형 뷰포트 및 다크 테마 설정**: `<meta name="viewport">`, `<meta name="theme-color">`를 통해 모바일/데스크톱 화면 비율과 다크 모드 UI를 지정합니다.
* **Import Map 선언**:
  ```html
  <script type="importmap">
  {
    "imports": {
      "three": "https://cdn.jsdelivr.net/npm/three@0.180.0/build/three.module.js",
      "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.180.0/examples/jsm/"
    }
  }
  </script>
  ```
  * 별도의 `npm install` 없이 표준 ES `import` 문법으로 Three.js 모듈을 가져올 수 있도록 매핑합니다.
* *(교실 파일 전용)* `<script type="application/json" id="sceneSpec">`: 교실의 치수, 가구 좌표, 카메라 프리셋, Blender 변환 규칙을 JSON 데이터 블록으로 분리해 두어 코드 유지보수성과 데이터 추출성을 높였습니다.

### 3.2. UI 레이아웃 구조 (`<body>` & `<style>`)
* **CSS Custom Properties (`:root`)**: `--bg`, `--panel`, `--teal`, `--blue` 등 CSS 변수를 활용하여 일관된 다크 글래스모피즘(Glassmorphism, `backdrop-filter: blur()`) 디자인을 구현했습니다.
* **레이아웃 배치**:
  * **버스 공장 (`#app`)**: CSS Grid(`grid-template-columns: 292px 1fr 272px`)를 사용해 좌측(공정 제어), 중앙(3D 뷰포트), 우측(관제 KPI 및 부품 설명) 3단 패널로 분할했습니다.
  * **스마트 교실**: 전체 화면 3D 캔버스(`position: fixed; inset: 0`) 위에 상단 헤더(`.head`), 우측 플로팅 제어 패널(`.control-panel`), 하단 카메라 바(`.bottom`)를 `z-index`로 띄우는 오버레이 방식을 사용했습니다.
* **반응형 미디어 쿼리 (`@media`)**: 화면 너비(`1250px`, `990px`, `780px`, `690px`)에 따라 패널을 숨기거나 모바일 드로어(Drawer) 메뉴로 전환됩니다.

### 3.3. 초기화 및 방어적 에러 핸들링 (`<script>`)
* **로딩 인디케이터 및 타임아웃 감시**:
  * 모듈 로딩이 지연되거나 네트워크 오류로 Three.js가 실행되지 않을 때 빈 화면 대신 안내 UI(`#fallback` 또는 `showStartupError`)를 띄우는 안전장치가 구현되어 있습니다.
* **WebGL 컨텍스트 손실 복구** *(교실 파일)*:
  * `webglcontextlost` 및 `webglcontextrestored` 이벤트를 리스닝하여 GPU 과부하로 화면이 끊겼을 때 텍스처와 그림자 맵을 재복구합니다.

### 3.4. Three.js 3D 엔진 핵심 로직 (`<script type="module">`)

두 파일의 자바스크립트 로직은 아래의 **6단계 파이프라인**으로 동작합니다.

#### ① 씬(Scene), 카메라, 렌더러 및 컨트롤 설정
* `THREE.Scene`: 배경색과 안개(`THREE.Fog`)를 설정해 공간감을 부여합니다.
* `THREE.PerspectiveCamera`: 원근 카메라(FOV 42~43°)를 배치합니다.
* `THREE.WebGLRenderer`: `ACESFilmicToneMapping`과 `PCFSoftShadowMap`을 활성화하여 실사풍 명암과 부드러운 그림자를 렌더링합니다. 스크린샷 저장을 위해 `preserveDrawingBuffer: true` 옵션이 켜져 있습니다.
* `OrbitControls`: 마우스 드래그(회전), 휠(줌), 우클릭(이동) 및 `enableDamping`(관성 효과)을 제공합니다.

#### ② 절차적(Procedural) 3D 모델링 헬퍼 함수
외부 `.glb`나 `.obj` 3D 모델 파일을 로드하지 않고, **순수 코드로 기하학적 도형을 조합**하여 가볍고 빠른 로딩을 구현했습니다.
* **헬퍼 함수**: `bx()`(직육면체), `cyl()`(원기둥), `sph()`(구), `tube()`/`rod()`(곡선 및 파이프) 함수를 정의해 반복적인 `Mesh` 생성 코드를 단축했습니다.
* **동적 캔버스 텍스처 (`canvasLabel` / `texCanvas`)**: HTML5 `<canvas>` 2D API로 텍스트나 칠판 UI, 나무 바닥 결을 그린 뒤 `THREE.CanvasTexture`로 변환하여 3D 표면에 매핑합니다.
* **계층적 그룹화 (`THREE.Group`)**:
  * 버스: `frame`(섀시), `wheelSystem`(바퀴), `power`(배터리), `skeleton`(골격) 등을 `parts` 배열에 등록해 공정 단계별로 `visible`과 위치를 제어합니다.
  * 교실: `architecture`(벽체), `windows`(창문/블라인드), `deskGroup`(책상 24세트), `pupils`(학생 아바타) 등으로 그룹화되어 있습니다.

#### ③ 상태 관리 객체 (`state` / `sim`)
* 애플리케이션의 모든 동적 상태를 단일 객체(`state` 또는 `sim`)에서 중앙 관리합니다.
  * 버스: `{ stage, auto, speed, door, xray, explode, spin, selected, view }`
  * 교실: `{ occupancy, outdoorDaylightPercent, temperatureTargetC, ventilationPercent, roomTemp, co2, lux, run }`

#### ④ 인터랙션 및 Raycasting
* **이벤트 바인딩**: HTML 버튼, 슬라이더(`input`), 단축키(`Space`, 방향키) 이벤트를 상태 변경 함수(`setStage`, `readInputs`, `setView`)와 연결합니다.
* **3D 객체 피킹 (`THREE.Raycaster`)** *(버스 파일)*: 화면 클릭 좌표를 정규화 장치 좌표(NDC, $-1 \sim +1$)로 변환하여 광선을 쏘고, 충돌한 부품의 `userData.partId`를 읽어 우측 패널에 설명을 출력합니다.

#### ⑤ 실시간 애니메이션 루프 (`setAnimationLoop` / `requestAnimationFrame`)
매 프레임마다 경과 시간(`clock.getDelta()`)을 계산하여 다음 연산을 수행합니다.
* **선형 보간 (`lerp`) 애니메이션**:
  * 조립 부품의 이동 및 분해 보기 위치 전환
  * 버스 출입문 개폐 슬라이딩
  * 카메라 시점 프리셋 전환 시 부드러운 위치 및 타겟 이동
* **환경/기계 시뮬레이션**:
  * 버스 공장: 산업용 로봇 팔 회전 및 천장 크레인 왕복 운동
  * 스마트 교실: 지수 감쇠 수식을 활용한 실시간 온도 및 $CO_2$ 농도 변화 계산, 에어컨 기류 파티클 순환, 차트 갱신
* **카메라 시야 확보(Wall Cutaway)** *(교실 파일)*: 카메라가 교실 앞벽이나 창가 벽 바깥쪽으로 이동할 때 벽면 메시를 자동으로 숨겨 시야가 가려지는 현상을 방지합니다.

#### ⑥ 데이터 입출력 (Export)
* **화면 캡처**: `renderer.domElement.toDataURL()` 또는 `toBlob()`을 이용해 현재 3D 렌더링 화면을 PNG 이미지로 다운로드합니다.
* **데이터 추출** *(교실 파일)*: 현재 씬의 모든 3D 오브젝트 월드 좌표와 시뮬레이션 기록을 `JSON` 및 `CSV` 파일로 다운로드할 수 있습니다.
