# 🏁 레트로 2D 탑다운 레이싱 게임 (`drive.html`) 전면 재설계 계획서

본 문서는 단일 정적 파일(`drive.html`)로 실행되는 고성능 레트로 2D 탑다운 레이싱 게임(‘Steel Racer’ 스타일)의 아키텍처 및 상세 구현 명세서입니다. 모바일 터치 환경과 데스크톱 키보드 환경의 파편화 방어, 크로스 브라우저 예외 차단, 델타타임 물리 동기화 및 심미적 완성도를 보장하는 완전 무결한 단일 파일 구조를 목표로 합니다.

---

## 1. 프로젝트 개요 및 핵심 목표

- **대상 파일**: [drive.html](file:///c:/Users/Voltron/Desktop/VScode/new folder/drive.html)
- **실행 환경**: 외부 라이브러리(CDN, 엔진) 없는 순수 **Vanilla HTML5 / CSS3 / JavaScript** 단일 정적 파일
- **지원 디바이스**: 
  - 모바일: iOS Safari, Android Chrome (세로/가로 뷰포트 완벽 대응)
  - 태블릿: iPadOS Safari, Android Tablet
  - 데스크톱: Windows, macOS (Chrome, Edge, Safari, Firefox)
- **스타일 콘셉트**: 1980~90년대 네오 아케이드 레이싱 감성의 고품질 2D 탑다운 레이싱 (Steel Racer 스타일)

---

## 2. 오류 방어 설계 요약 (Error Defense & Robustness Architecture)

모바일 브라우저와 저사양/고주사율 기기에서 발생하는 치명적인 런타임 오류 및 조작 버그를 원천 차단하는 5대 방어 메커니즘을 정의합니다.

```
+--------------------------------------------------------------------------+
|                        Unified Viewport Manager                          |
|   visualViewport + dvh + 16:9 Letterbox/Pillarbox CSS/Canvas Scaling    |
+------------------------------------+-------------------------------------+
                                     |
+------------------------------------+-------------------------------------+
|                 Input Abstraction Layer (IAL)                            |
|   - Desktop Keyboard (WASD, Arrows, Space)                              |
|   - Mobile Multi-touch Virtual Pad (Pointer Events + Id Tracking)       |
|   - Blur/VisibilityChange: Auto-Reset inputState                        |
+------------------------------------+-------------------------------------+
                                     |
+------------------------------------+-------------------------------------+
|                     Deterministic Physics Loop                           |
|   - requestAnimationFrame(timestamp)                                    |
|   - Delta Time Cap (dt = min(rawDt, 0.1s))                               |
|   - Friction, Acceleration, Angular Velocity, Off-road Penalty          |
+------------------------------------+-------------------------------------+
                                     |
+------------------------------------+-------------------------------------+
|                     Procedural Canvas 2D Engine                          |
|   - Zero External Asset Dependency (100% Vector/Path Rendering)         |
|   - Track, Curb, Off-road Grass, Checkpoints, Particle Skidmarks        |
+--------------------------------------------------------------------------+
```

### 2.1. 고정 시간 물리 루프 및 델타 타임 캡핑 (Fixed Timestep & Frame-Spike Defense)
- **문제점**: 
  - 60Hz, 120Hz(ProMotion), 144Hz 등 디바이스 주사율에 따라 가변 `dt`를 직접 쓰면 부동소수점 오차가 누적되어 코너링 반경과 랩타임이 달라지는 불공정함(Non-deterministic) 발생.
  - 브라우저 탭 전환, 백그라운드 복귀 시 누적된 거대한 시간으로 인해 차량이 벽을 뚫고 날아가는 터널링(Tunnelling) 현상.
- **방어 로직 (Fixed Timestep Accumulator)**:
  ```javascript
  const FIXED_STEP = 1 / 60; // 60Hz 고정 물리 연산 간격 (약 0.01667초)
  const MAX_ACCUMULATOR = 0.1; // 최대 100ms 캡핑 (프레임 스파이크 방어)
  let accumulator = 0;
  let lastTime = performance.now();

  function gameLoop(currentTime) {
    let frameTime = (currentTime - lastTime) / 1000;
    lastTime = currentTime;

    // 비정상적 시간 스파이크 클램핑
    if (frameTime > MAX_ACCUMULATOR || frameTime <= 0 || isNaN(frameTime)) {
      frameTime = MAX_ACCUMULATOR;
    }
    accumulator += frameTime;

    // 고정 시간 단위로 결정론적 물리 업데이트 (주사율 독립성 보장)
    while (accumulator >= FIXED_STEP) {
      updatePhysics(FIXED_STEP);
      accumulator -= FIXED_STEP;
    }

    // 남은 시간 비율(alpha)을 활용한 부드러운 화면 렌더링
    const renderAlpha = accumulator / FIXED_STEP;
    render(renderAlpha);

    requestAnimationFrame(gameLoop);
  }
  ```

### 2.2. 포인터 고착 방지 (Pointer Sticking Defense) & 멀티터치 보장
- **문제점**:
  - 버튼을 누른 채 화면 밖으로 손가락이 미끄러지면 `pointerup`이 유실되어 키가 계속 눌린 상태로 유지됨.
  - 조향(좌/우)을 누른 상태에서 가속/브레이크를 동시에 누르면 터치 ID 충돌로 입력이 씹히는 문제.
- **방어 로직**:
  - `pointerdown`, `pointerup`, `pointercancel`, `pointerleave` 4개 이벤트를 모두 바인딩.
  - 각 버튼 엘리먼트마다 활성화된 `pointerId`를 추적(`activePointerId`), 동일 포인터의 해제/이탈 시 즉시 해제.
  - `window.addEventListener('pointerup', ...)` 전역 리스너를 보조 안전망으로 등록.

### 2.3. 브라우저 제스처 및 스크롤 원천 차단
- **문제점**:
  - iOS Safari 풀-투-리프레시(새로고침), 하단 주소창 팝업, 바운스 오버스크롤.
  - 화면 연속 탭 시 확대(`pinch-to-zoom`, `double-tap zoom`), 길게 누를 시 텍스트/이미지 선택 메뉴 팝업.
- **방어 로직**:
  - CSS 전역:
    ```css
    html, body {
      touch-action: none;
      user-select: none;
      -webkit-user-select: none;
      overscroll-behavior: none;
      overflow: hidden;
      width: 100%;
      height: 100%;
      height: 100dvh;
    }
    ```
  - JavaScript 이벤트 레벨에서 `touchstart`, `touchmove`, `contextmenu` 등의 기본 동작 방어(`preventDefault({ passive: false })`).

### 2.4. 뷰포트 종횡비 유지 및 캔버스 자동 스케일링 (Letterbox / Pillarbox)
- **기준 가상 해상도**: `960 x 540` (16:9 와이드스크린 최적 비율)
- **반응형 계산**:
  - `window.visualViewport`의 `width`와 `height`를 감지하여 16:9 비율 유지 스케일 팩터 계산.
  - 레터박스(상/하 여백) 또는 필러박스(좌/우 여백) 영역을 검정 백그라운드로 처리하고 캔버스를 화면 정중앙에 배치(`transform: translate(-50%, -50%)`).
  - 모바일 가상 조작 패드는 캔버스 비율 내부 또는 하단 레터박스 영역에 자연스럽게 오버레이.

### 2.5. 윈도우 포커스 아웃 시 입력 강제 리셋 (Blur Safe-guard)
- **문제점**: 데스크톱에서 가속 키(W 또는 Up)를 누른 상태로 `Alt+Tab` 또는 다른 창을 클릭하면 `keyup`을 받지 못해 무한 주행.
- **방어 로직**:
  ```javascript
  function resetInputs() {
    inputState.up = false;
    inputState.down = false;
    inputState.left = false;
    inputState.right = false;
    inputState.drift = false;
  }
  window.addEventListener('blur', resetInputs);
  document.addEventListener('visibilitychange', () => {
    if (document.hidden) resetInputs();
  });
  ```

---

## 3. 디바이스 입력 감지 및 통합 입력 상태 (Input Abstraction Layer)

```
[Desktop Keyboard] -----> [ InputState ] <----- [ Virtual On-Screen Pad ]
- W / ArrowUp      -> up: boolean             - D-Pad Left / Right
- S / ArrowDown    -> down: boolean           - Pedal Accel (Gas)
- A / ArrowLeft    -> left: boolean           - Pedal Brake / Reverse
- D / ArrowRight   -> right: boolean          - Handbrake / Drift Button
- Space / Shift    -> drift: boolean
```

### 디바이스 감지 분기
1. CSS 미디어 쿼리: `@media (hover: none) and (pointer: coarse)`
2. 최초 터치 감지: 런타임 중 첫 `touchstart` 발생 시 가상 컨트롤러 오버레이 자동 표시 (`isTouchDevice = true`)
3. 데스크톱 마우스 접속 시 가상 버튼을 DOM 상에서 완전히 숨김(`display: none`) 처리하여 광활한 시야 확보

---

## 4. 물리 및 주행 메커니즘 (Vehicle Dynamics)

Steel Racer 및 클래식 탑다운 레이서의 박진감 있는 주행을 위한 아케이드 2D 물리 모델을 설계합니다.

| 물리 요소 | 동작 설명 | 적용 수식 / 알고리즘 |
| :--- | :--- | :--- |
| **가속 (Acceleration)** | 전진 입력 시 속도 증가 | $v \leftarrow v + a_{acc} \cdot dt$ |
| **최대 속도 (Max Speed)** | 최고 전진 속도 제한 | $v \le V_{max}$ (부스터/터보 시 일시적 확장 가능) |
| **후진 / 브레이크** | 제동 및 저속 후진 | 감속: $v \leftarrow v - a_{brake} \cdot dt$, 후진 한계: $v \ge -V_{rev\_max}$ |
| **조향 (Steering)** | 차량 현재 속도에 비례한 회전각 변경 | $\theta \leftarrow \theta + \omega \cdot \frac{v}{V_{max}} \cdot dt$ (정지 상태에서는 회전 불가) |
| **자연 마찰 (Rolling Friction)**| 입력 없을 시 서서히 감속 | $v \leftarrow v \cdot (1 - \mu_{roll} \cdot dt)$ |
| **측면 슬립 (Lateral Grip & Drift)**| 코너링 시 횡방향 원심력 및 타이어 미끄러짐 | 진행 방향 벡터와 차체 방향 벡터의 보간 (드리프트 시 그립 감소 + 연기 파티클) |
| **오프로드 감속 (Grass Penalty)**| 트랙 아스팔트 이탈 시 마찰력 대폭 증가 | $V_{max}^{offroad} = V_{max} \times 0.4$, 마찰계수 3배 증가 |
| **충돌 반발 (Wall Bounce)**| 벽/가드레일 충돌 시 튕김 및 속도 감쇠 | 충돌 법선 벡터로 반사, 속도 $40\%$ 즉시 손실 |

---

## 5. 트랙 렌더링 및 게임 콘텐츠 설계 (Zero-Dependency)

외부 이미지 에셋 404 에러나 로딩 지연을 없애기 위해 **100% Canvas 2D API 절차적 렌더링**을 적용합니다.

### 5.1. 트랙 구성
- **트랙 레이아웃**: 레이싱 전용 서킷 (헤어핀 커브, 고속 직선 구간, S자 연속 코너 포함)
- **구역 정의**:
  - `TRACK (아스팔트)`: 짙은 다크 차콜 컬러, 정상 속도 및 정상 그립
  - `CURB (연석)`: 빨강/하양 체커보드 패턴, 약간의 진동 및 감속
  - `GRASS / GRAVEL (오프로드)`: 잔디/모래 텍스처 절차적 생성, 급격한 감속
  - `START / FINISH LINE`: 체커 플래그 패턴 라인
  - `CHECKPOINTS`: 역주행 방지 및 랩 타임 완주 판정을 위한 투명/반투명 센서 라인 (최소 6~8개)

### 5.2. 게임 루프 및 인게임 요소
1. **타임어택 & 랩타임 시스템**:
   - 현재 랩 타임 (Current Lap)
   - 최고 랩 타임 (Best Lap - LocalStorage 영구 저장)
   - 랩 카운터 (예: 3 Laps 완주 시 결과 화면)
2. **코인 & 수집품**:
   - 트랙 코스 상에 회전하는 네온 코인 배치
   - 코인 수집 시 속도 일시 부스트(Nitro) 효과 및 스코어 증가
3. **시각 연출 (VFX)**:
   - 고속 코너링/드리프트 시 아스팔트에 남는 타이어 스키드마크 (Alpha Fade-out 큐)
   - 배기구 스파크 및 타이어 연기 캔버스 파티클 시스템
   - 속도감에 따른 카메라 쉐이크(Screen Shake) 효과
4. **오디오 (Web Audio API Synthesizer)**:
   - 외부 mp3 파일 없이 Web Audio API의 Oscillator(오실레이터)로 실시간 엔진 배기음(주파수 변조), 스키드 타이어 마찰음, 코인 획득 챠임음 생성 (무결점 로딩)

---

## 6. UI/UX 및 가상 컨트롤러 설계

### 6.1. 반응형 HUD
- **상단**: 랩(Lap 1/3), 현재 시간(00:14.28), 베스트 랩(00:42.10), 코인 카운트
- **하단(또는 우하단)**: 아날로그 감성 네온 디지털 속도계(Speedometer: km/h) 및 부스트 게이지
- **일시정지/재도전 버튼**: 언제든 재시작 가능한 상단 퀵 버튼

### 6.2. 모바일 온스크린 컨트롤러 레이아웃
```
+-------------------------------------------------------------------+
|  [HUD: Lap 1/3 | Time: 00:15.3 | Best: 00:38.2 | 🪙 05 ]   [🔄]   |
|                                                                   |
|                         [ 캔버스 게임 화면 ]                       |
|                                                                   |
|   (조향부 - 좌측 엄지)                      (페달부 - 우측 엄지)   |
|     +---+   +---+                            +-------+            |
|     | ◀ |   | ▶ |                            |   ▲   | (가속/GAS) |
|     +---+   +---+                            +-------+            |
|                                              +-------+            |
|                                              |   ▼   | (브레이크) |
|                                              +-------+            |
+-------------------------------------------------------------------+
```
- 엄지손가락이 닿기 쉬운 인체공학적 여백(Safe-area) 준수.
- 터치 시 햅틱 진동 피드백(`navigator.vibrate`) 연동 (지원 디바이스).
- 터치 활성 상태 시 은은한 네온 글로우 발광 애니메이션.

---

## 7. 주요 파라미터 튜닝 가이드 (Tuning Parameters)

추후 난이도 및 조작감 커스터마이징을 위한 주요 설정 변수 테이블입니다.

| 변수명 | 권장 기본값 | 설명 및 영향 |
| :--- | :--- | :--- |
| `CANVAS_WIDTH` / `HEIGHT` | `960` / `540` | 게임 가상 렌더링 해상도 (16:9) |
| `MAX_SPEED` | `480` px/s | 아스팔트 위에서 차량의 최고 직선 속도 |
| `ACCELERATION` | `320` px/s² | 가속 페달 입력 시 전진 가속도 |
| `BRAKE_FORCE` | `450` px/s² | 브레이크 입력 시 제동력 |
| `REVERSE_MAX_SPEED` | `150` px/s | 후진 최고 속도 |
| `TURN_SPEED` | `3.6` rad/s | 초당 최대 조향 각속도 |
| `ROLLING_FRICTION` | `0.45` | 관성 주행 시 자연 감속 계수 |
| `LATERAL_FRICTION` | `0.85` | 횡방향 타이어 접지력 (낮을수록 잘 미끄러짐) |
| `DRIFT_LATERAL_FRICTION` | `0.35` | 핸드브레이크/드리프트 활성화 시 접지력 |
| `OFFROAD_PENALTY` | `0.40` | 잔디밭 진입 시 최고속도/가속도 감소 배율 ($40\%$) |
| `MAX_DELTA_TIME` | `0.05` s | 델타 타임 상한값 (프레임 스파이크 방어) |

---

## 8. 잠재적 장애 요인 및 기술적 대처 전략 (Troubleshooting & Risk Mitigation)

브라우저 게임 개발 시 발생할 수 있는 주요 런타임 장애 요인 7가지와 이에 대한 사전 방어 구현 전략입니다.

```
+-------------------------------------------------------------------------------------------------+
|                                  런타임 장애 및 예외 방어 매트릭스                               |
+------------------------------------+------------------------------------------------------------+
| 장애 시나리오                       | 원인 및 기술적 대처 방안                                    |
+------------------------------------+------------------------------------------------------------+
| 1. Web Audio 자동 재생 차단        | 브라우저 정책(Autoplay Policy)으로 첫 사용자 인터랙션 전    |
|    (AudioContext Suspended)        | AudioContext 잠김. 첫 키 입력/터치 이벤트 발생 시 1회성      |
|                                    | audioCtx.resume() 해제 핸들러 등록.                         |
+------------------------------------+------------------------------------------------------------+
| 2. 모바일 주소창 가변 높이 및      | 스크롤/방향전환 시 화면 깜빡임. visualViewport API와        |
|    리사이즈 플리커(Flicker)         | 100dvh를 연동하고, 리사이즈 이벤트에 100ms 디바운싱 적용.   |
+------------------------------------+------------------------------------------------------------+
| 3. 오프로드 판정 성능 저하         | 매 프레임 ctx.getImageData() 호출 시 CPU-GPU 병목 발생.    |
|    (Canvas Readback Bottleneck)    | 초저해상도(120x67.5) 오프스크린 캔버스 캐싱 또는 수학적 세그먼트 |
|                                    | 거리 계산(Distance to Centerline) 알고리즘으로 대체.       |
+------------------------------------+------------------------------------------------------------+
| 4. 고속 주행 시 체크포인트 터널링  | 델타타임 동안 이동 거리가 커서 체크포인트 선분을 건너뜀.   |
|    (Tunnelling / Physics Leak)     | 차량의 이전 위치(x0,y0)와 현재 위치(x1,y1) 선분과          |
|                                    | 체크포인트 선분의 교차(Line Intersection) 판정 적용.       |
+------------------------------------+------------------------------------------------------------+
| 5. 모바일 엣지 제스처 간섭         | 화면 가장자리 스와이프 시 브라우저 뒤로가기/홈바 작동.      |
|    (System Gesture Conflict)       | 가상 패드에 env(safe-area-inset-*) 및 외곽 16px 마진 확보. |
+------------------------------------+------------------------------------------------------------+
| 6. 시크릿 모드 localStorage 예외  | Safari 시크릿 모드 등에서 QuotaExceeded 또는 SecurityError. |
|    (Storage Access Error)          | try/catch 래핑 및 메모리 기반 Mock Storage 폴백 적용.       |
+------------------------------------+------------------------------------------------------------+
| 7. 탭 전환 시 사운드 무한 루프     | 탭 백그라운드 전환 시 오디오 오실레이터가 멈추지 않는 현상. |
|    (Audio Stuck on Background)     | visibilitychange 이벤트 감지 시 모든 오디오 게인(Gain) 0 처리|
+------------------------------------+------------------------------------------------------------+
```

### 상세 대처 코드 예시

#### [대처 1] Web Audio API Autoplay Unlock
```javascript
let audioCtx = null;
function initAudio() {
  if (!audioCtx) {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  }
  if (audioCtx.state === 'suspended') {
    audioCtx.resume();
  }
}
window.addEventListener('keydown', initAudio, { once: true });
window.addEventListener('pointerdown', initAudio, { once: true });
```

#### [대처 4] 체크포인트 선분 교차(Line-Segment Intersection) 판정
차량의 점(Point) 좌표만으로 충돌을 검사하면 속도가 빠를 때 체크포인트를 건너뛰는 현상이 발생합니다. 이전 위치와 현재 위치를 잇는 궤적 선분과 체크포인트 선분의 교차 여부를 판정합니다.
```javascript
function checkIntersection(p1, p2, p3, p4) {
  const ccw = (A, B, C) => (C.y - A.y) * (B.x - A.x) > (B.y - A.y) * (C.x - A.x);
  return (ccw(p1, p3, p4) !== ccw(p2, p3, p4)) && (ccw(p1, p2, p3) !== ccw(p1, p2, p4));
}
```

---

## 9. 단계별 기능 구현 로드맵 (Phased Implementation Roadmap)

개발은 **[1단계: 기초 핵심 기능] ➡️ [2단계: 세부 주행/규칙 기능] ➡️ [3단계: 전체 시스템 통합 및 정상 작동 검증] ➡️ [4단계: 비주얼/UI 연출 및 사운드 완성]**의 4단계 점진적 파이프라인으로 진행됩니다.

```
+---------------------------------------------------------------------------------+
|                        단계별 점진적 개발 파이프라인                             |
+---------------------------------------------------------------------------------+
| [Step 1] Core Foundation                                                        |
|   ├── 가상 캔버스 뷰포트 & 레터박스 자동 스케일러                                |
|   ├── 델타타임 물리 루프 & 기초 키보드 조작                                     |
|   └── 박스형 임시 차량의 2D 이동/회전                                           |
+---------------------------------------+-----------------------------------------+
                                        v
+---------------------------------------------------------------------------------+
| [Step 2] Detailed Mechanics                                                     |
|   ├── 관성, 드리프트 접지력 및 횡방향 슬립 모델                                  |
|   ├── 트랙 서킷 레이아웃 및 온트랙/오프로드(잔디) 판정                           |
|   ├── 체크포인트 순차 통과 & 랩타임 카운터 시스템                                |
|   ├── 코인 스폰 & 수집형 니트로 부스터 기능                                     |
|   └── 모바일 터치 패드 (포인터 고착 방지 + 독립 멀티터치)                        |
+---------------------------------------+-----------------------------------------+
                                        v
+---------------------------------------------------------------------------------+
| [Step 3] Integration & QA Verification                                          |
|   ├── 모든 시스템 결합 후 크로스 브라우저/기기별 무결성 검증                     |
|   ├── 탭 전환(Blur) 시 입력 리셋 & 프레임 스파이크 방어 검증                     |
|   ├── 고속 주행 시 체크포인트 건너뜀 및 벽 뚫림 방지 검증                        |
|   └── 모바일 손가락 이탈 시 키 고착 현상 원천 배제 검증                          |
+---------------------------------------+-----------------------------------------+
                                        v
+---------------------------------------------------------------------------------+
| [Step 4] Aesthetics, VFX & Audio Polish                                         |
|   ├── Steel Racer 스타일 네오 레트로 아케이드 디자인                            |
|   ├── 캔버스 절차적 차량 렌더링 (차체, 헤드라이트, 휠)                          |
|   ├── 비주얼 이펙트 (타이어 스키드마크 궤적, 연기/배기 파티클)                   |
|   ├── 사이버펑크 네온 HUD (스피도미터, 랩타이머, 베스트랩 기록)                  |
|   └── Web Audio API 절차적 사운드 (엔진 RPM, 스키드음, 코인음)                  |
+---------------------------------------------------------------------------------+
```

### Step 1: 기본 핵심 기능 구현 (Core Foundation)
- **목표**: 어떤 디바이스에서도 왜곡 없이 화면에 고정 해상도로 출력되고, 델타타임 기반으로 기초 차량 이동이 가능한 최소 구동 뼈대 제작.
- **주요 작업**:
  1. `index.html` 기반 단일 HTML 골격, 전역 CSS(`100dvh`, 스크롤/바운스 차단, 레터박스 래퍼) 구성.
  2. 가상 해상도 `960x540` 캔버스 생성 및 브라우저 창 크기에 맞춘 자동 스케일링 함수(`fitCanvasToScreen`) 작성.
  3. `requestAnimationFrame`과 `performance.now()`를 통한 델타타임 루프 및 캡핑(`dt <= 0.05s`) 연동.
  4. 데스크톱 키보드 입력 핸들러(`keydown`, `keyup`)를 `inputState`에 바인딩.
  5. 임시 사각형 차량 모델의 직진 가속, 후진, 좌우 조향 회전 물리 구현.

### Step 2: 세부 주행 및 게임플레이 기능 구현 (Detailed Mechanics)
- **목표**: 레이싱 게임으로서의 정밀 물리, 서킷 환경, 규칙 시스템 완성.
- **주요 작업**:
  1. **고급 아케이드 물리 모델**:
     - 속도에 비례한 조향각 제어(정지 시 조향 불가, 최고속 시 안정화).
     - 횡방향 타이어 접지력(`LATERAL_FRICTION`) 및 핸드브레이크/드리프트 슬립 물리.
  2. **트랙 및 환경 상호작용**:
     - 아스팔트 메인 트랙, 코너 체커 연석, 외부 잔디 구역 수학적/절차적 정의.
     - 잔디 진입 시 감속 및 마찰 증가(`OFFROAD_PENALTY`) 처리.
     - 가드레일/화면 경계 충돌 바운스 반발력.
  3. **레이싱 규칙 엔진**:
     - 순차 체크포인트(CP0 ~ CP5) 통과 검증 및 역주행 방지 로직.
     - 스타트/피니시 라인 통과 시 랩 카운트 증가 및 랩타임 측정.
     - 트랙 위에 배치되는 수집용 회전 코인 및 획득 시 2초간 부스트(`BOOST_SPEED`) 부여.
  4. **모바일 가상 컨트롤러 구축**:
     - 좌측: 조향 D-Pad (◀ / ▶)
     - 우측: 가속 페달(▲) / 브레이크 페달(▼) / 드리프트 버튼
     - `pointerdown`, `pointerup`, `pointercancel`, `pointerleave` 4중 바인딩 및 독립 `pointerId` 추적으로 키 고착 방어.

### Step 3: 전체 시스템 통합 및 정상 작동 검증 (Integration & QA Verification)
- **목표**: 세부 기능들이 유기적으로 결합된 상태에서 런타임 오류나 예외 상황이 전혀 없는지 철저히 테스트.
- **검증 항목 체크리스트**:
  - [ ] **키 고착 검증**: 가상 패드 버튼을 누른 채 화면 밖이나 다른 버튼으로 손가락을 미끄러뜨렸을 때 입력이 즉시 해제되는가?
  - [ ] **멀티터치 검증**: 좌측 조향 버튼과 우측 가속 버튼을 동시에 누른 채 주행이 매끄러운가?
  - [ ] **창 전환 검증**: 가속 키를 누른 채 `Alt+Tab` 또는 브라우저 창을 벗어났을 때 복귀 시 무한 주행하지 않고 정지하는가?
  - [ ] **프레임 캡핑 검증**: 브라우저 탭을 수 초간 백그라운드로 뒀다가 돌아와도 차량이 맵 밖으로 튕겨나가지 않는가?
  - [ ] **터널링 방지 검증**: 부스터 최고 속도로 체크포인트를 통과해도 랩타임 판정이 누락되지 않는가?
  - [ ] **해상도 검증**: 스마트폰 가로 모드, 세로 모드, 태블릿, 데스크톱 창 리사이즈 시 화면비가 찌그러지지 않고 레터박스가 유지되는가?

### Step 4: UI/UX 고도화, 비주얼 이펙트 및 사운드 완성 (Aesthetics, VFX & Sound Design)
- **목표**: 1980~90년대 네오 아케이드 레이서(Steel Racer) 감성의 화려하고 몰입감 넘치는 시각/청각 피니싱.
- **주요 작업**:
  1. **절차적 비주얼 렌더링**:
     - 캔버스 2D 패스 기반의 정교한 스포츠카 디자인 (차체 그라디언트, 윈드실드, 휠 림, 헤드라이트 빔 효과).
     - 아스팔트 질감, 코너 연석의 붉은색/흰색 체커 패턴, 트랙 스폰서 배너.
  2. **특수 효과 (VFX)**:
     - 급선회 및 드리프트 시 아스팔트에 남는 타이어 스키드마크 궤적(알파값 페이드아웃).
     - 가속 시 배기구 불꽃 파티클 및 타이어 흰색 연기 파티클 분사.
     - 고속 충돌 시 캔버스 스크린 쉐이크(Screen Shake) 효과.
  3. **네온 아케이드 HUD**:
     - 아날로그 게이지 + 디지털 km/h 속도계 (LCD 네온 블루 폰트 스타일).
     - 실시간 랩 타임 타이머, 베스트 랩 표시, 수집한 코인 카운트.
     - 레이스 완주 시 축하 연출 모달 및 최고 기록 로컬스토리지 갱신.
  4. **Web Audio API 절차적 사운드 디자인**:
     - 차량 속도 및 RPM에 따라 주파수가 변조되는 실시간 엔진 배기음(Sawtooth Oscillator + BiquadFilter).
     - 드리프트 및 급제동 시 타이어 마찰 스키드 사운드(White Noise Buffer).
     - 코인 획득 시 청량한 아케이드 챠임 사운드.
     - 첫 사용자 상호작용 시 자동 오디오 언락 안전장치.

---

## 10. 최적화 및 유지보수 보완 전략 (Advanced Optimization & Maintenance)

기존 계획서의 파편화 방어 수준을 넘어, 현업 게임 프로그래머 수준의 부드러운 60프레임 최적화와 견고한 코드 유지보수를 위한 6가지 핵심 보완책을 정의합니다.

```
+--------------------------------------------------------------------------------------------------+
|                              고급 최적화 및 아키텍처 보완 매트릭스                                |
+-----------------------+----------------------------------+---------------------------------------+
| 영역                   | 문제점 (Bottleneck)               | 최적화 해결책 (Solution)               |
+-----------------------+----------------------------------+---------------------------------------+
| 10.1 렌더링 최적화     | 매 프레임 복잡한 서킷 재연산/재드로잉 | 오프스크린 캔버스 1회 사전 렌더링 후   |
|                       | (CPU/GPU 모바일 발열 및 드랍)     | drawImage() 고속 블릿 복사 캐싱       |
+-----------------------+----------------------------------+---------------------------------------+
| 10.2 메모리 GC 방어   | 파티클/스키드마크 동적 new 생성   | 고정 크기 오브젝트 풀링(Object Pool)   |
|                       | 빈번한 가비지 컬렉션 스터터링     | 런타임 메모리 할당 제로(Zero-Alloc)   |
+-----------------------+----------------------------------+---------------------------------------+
| 10.3 물리 공정성      | 주사율(60Hz vs 144Hz) 가변 dt로   | Fixed Timestep(1/60s) 누적기          |
|                       | 인한 랩타임/코너링 물리 오차      | 완전 결정론적(Deterministic) 연산     |
+-----------------------+----------------------------------+---------------------------------------+
| 10.4 오디오 품질      | 게인/주파수 즉시 변경 시 튀는    | exponentialRampToValueAtTime()        |
|                       | 불쾌한 클릭 노이즈(Popping)      | 0.05초 지수 감쇠 곡선 적용            |
+-----------------------+----------------------------------+---------------------------------------+
| 10.5 코드 상태 구조화 | 조건문 난립으로 인한 스파게티 코드| 유한 상태 머신(FSM) 아키텍처 도입     |
|                       | (메뉴, 카운트다운, 주행, 결과)   | (MENU | COUNTDOWN | RACING | OVER)    |
+-----------------------+----------------------------------+---------------------------------------+
| 10.6 로컬스토리지 보안 | F12 개발자 도구를 통한 랩타임     | Salt 기반 해시 체크섬 및 Base64       |
|                       | 평문 변조(어뷰징)                | 무결성 검증 디코딩                     |
+-----------------------+----------------------------------+---------------------------------------+
```

### 10.1. 렌더링 최적화: 매 프레임 트랙 다시 그리기 방지 (Offscreen Canvas Caching)
- **문제점**: 
  - 100% Canvas 2D API 절차적 렌더링 시, 매 프레임(초당 60~120회) 복잡한 트랙 베지에 곡선, 체커보드 연석, 아스팔트 그레인 등을 새로 그리면 모바일 AP의 CPU-GPU 드로우콜 병목이 발생하여 발열 및 프레임 드랍이 불가피합니다.
- **보완책**:
  - 화면에 표시되지 않는 메모리 오프스크린 캔버스(`trackCanvas = document.createElement('canvas')`)를 생성하고, 게임 초기화 시 트랙의 모든 정적 배경(잔디 텍스처, 아스팔트 서킷, 연석, 결승선 라인)을 **단 1회만 사전 렌더링**합니다.
  - 메인 렌더 루프에서는 `mainCtx.drawImage(trackCanvas, 0, 0)` 단 1회의 고속 비트맵 복사(Blit)만 수행하므로 CPU 연산 부하가 90% 이상 절감됩니다.

```javascript
// 트랙 오프스크린 캐시 생성
let trackCanvas = null;

function prerenderTrack(width, height) {
  trackCanvas = document.createElement('canvas');
  trackCanvas.width = width;
  trackCanvas.height = height;
  const offCtx = trackCanvas.getContext('2d');

  drawGrassBackground(offCtx, width, height);
  drawCircuitAsphalt(offCtx);
  drawCheckeredCurbs(offCtx);
  drawFinishLine(offCtx);
}

function render(alpha) {
  // 1. 단 1줄의 고속 블릿으로 정적 서킷 렌더링
  ctx.drawImage(trackCanvas, 0, 0);

  // 2. 동적 객체(스키드마크, 코인, 차량, 파티클)만 그 위에 드로잉
  renderSkidmarks(ctx);
  renderCoins(ctx);
  renderVehicle(ctx, alpha);
  renderParticles(ctx);
}
```

---

### 10.2. 메모리 누수 방지: 가비지 컬렉션(GC) 스터터링 억제 (Object Pooling)
- **문제점**:
  - 코너링 스키드마크 점이나 배기구 불꽃, 타이어 연기 파티클을 `new Particle()` 형태로 매 프레임 생성하고 수명이 끝나 배열에서 `splice()`/`pop()`하면, JavaScript 엔진의 V8 가비지 컬렉터가 마이너 GC를 수시로 실행하여 1~2초마다 미세한 끊김(Micro-stuttering)이 발생합니다.
- **보완책**:
  - **오브젝트 풀링(Object Pooling)** 기법을 필수 적용합니다.
  - 게임 시작 시 고정 크기의 파티클 풀(예: 150개)과 스키드마크 풀(예: 300개)을 단 1회 선할당합니다.
  - 객체가 필요할 때는 풀에서 비활성(`active === false`) 요소를 찾아 파라미터만 덮어씌워 활성화(`active = true`)하고, 수명이 다하면 다시 `active = false`로 마킹하여 **런타임 힙 메모리 재할당을 0으로 유지**합니다.

```javascript
class ParticlePool {
  constructor(size = 150) {
    this.pool = Array.from({ length: size }, () => ({
      x: 0, y: 0, vx: 0, vy: 0, life: 0, maxLife: 1,
      color: '#fff', size: 2, active: false
    }));
  }

  spawn(x, y, vx, vy, color, size, maxLife) {
    const p = this.pool.find(item => !item.active);
    if (!p) return; // 풀이 꽉 찼으면 추가 할당 없이 드랍
    p.x = x; p.y = y; p.vx = vx; p.vy = vy;
    p.color = color; p.size = size; p.life = 0; p.maxLife = maxLife;
    p.active = true;
  }

  update(dt) {
    for (let i = 0; i < this.pool.length; i++) {
      const p = this.pool[i];
      if (!p.active) continue;
      p.life += dt;
      if (p.life >= p.maxLife) {
        p.active = false;
        continue;
      }
      p.x += p.vx * dt;
      p.y += p.vy * dt;
    }
  }

  render(ctx) {
    for (let i = 0; i < this.pool.length; i++) {
      const p = this.pool[i];
      if (!p.active) continue;
      const alpha = 1 - (p.life / p.maxLife);
      ctx.fillStyle = p.color;
      ctx.globalAlpha = alpha;
      ctx.fillRect(p.x, p.y, p.size, p.size);
    }
    ctx.globalAlpha = 1.0;
  }
}
```

---

### 10.3. 물리 엔진의 공정성: 가변 델타 타임의 한계 극복 (Fixed Timestep)
- **문제점**:
  - 60Hz 스마트폰과 120Hz/144Hz 게이밍 모니터 환경에서 단순 가변 `dt`로 주행 물리를 계산하면, 조향 각속도 및 마찰 감속의 부동소수점 오차(Rounding Error)가 누적되어 동일한 입력을 가해도 코너 주행 궤적과 최종 랩타임이 달라집니다.
- **보완책**:
  - `accumulator` 기반의 **고정 타임스텝 물리 루프 (Fixed 60Hz Step, $1/60 \approx 0.01667s$)**를 구축합니다.
  - 물리 시뮬레이션은 화면 주사율과 관계없이 항상 정확히 동일한 $0.01667$초 간격으로 수행되므로 완벽한 결정론적(Deterministic) 공정성을 보장합니다.
  - 남은 오차 시간($renderAlpha$)을 활용해 이전 프레임 위치와 현재 위치 사이를 선형 보간(`lerp`)하여, 144Hz 초고주사율 화면에서도 떨림 없는 부드러운 렌더링을 구현합니다.

---

### 10.4. 오디오 처리: 클릭 노이즈(Popping / Clicking) 방지
- **문제점**:
  - Web Audio API에서 가속 페달을 뗐을 때 엔진 볼륨을 줄이거나, 브레이크 시 스키드 사운드를 멈출 때 `gainNode.gain.value = 0`처럼 불연속적으로 값을 즉시 대입하면, 오디오 신호의 파형이 0V로 급격히 끊어지며 스피커에서 불쾌한 '퍽(Click/Pop)' 소리가 발생합니다.
- **보완책**:
  - `AudioParam.setValueAtTime()` 및 `exponentialRampToValueAtTime()` (또는 `setTargetAtTime`)을 적용하여 0.05초 동안 부드러운 지수 감쇠 곡선을 그리며 전환되도록 처리합니다.

```javascript
function setAudioGainSmooth(gainNode, targetVal, duration = 0.05) {
  if (!audioCtx) return;
  const now = audioCtx.currentTime;
  // 0으로 떨어뜨릴 때는 지수함수 특성상 0.0001로 감쇠 후 0으로 리셋
  const safeTarget = Math.max(targetVal, 0.0001);
  gainNode.gain.cancelScheduledValues(now);
  gainNode.gain.setValueAtTime(gainNode.gain.value, now);
  gainNode.gain.exponentialRampToValueAtTime(safeTarget, now + duration);
}
```

---

### 10.5. 코드 구조화: 유한 상태 머신(FSM) 설계 도입
- **문제점**:
  - 단일 파일 구조에서 타이틀 화면, 카운트다운(3-2-1-GO), 레이싱 주행, 일시정지, 게임 오버/완주 화면 등을 전역 boolean 변수(`isTitle`, `isPaused`, `isFinished`)로 처리하면 분기문이 얽혀 스파게티 코드가 됩니다.
- **보완책**:
  - 명시적 상태 상수(`GameState = { MENU, COUNTDOWN, RACING, PAUSED, FINISHED }`)를 선언하고, 각 상태에 따라 `handleInput()`, `update()`, `renderUI()`의 동작을 분기하는 단일 관리자(`FSM Manager`)를 최상단에 배치합니다.

```
 [MENU] ──(터치/스페이스)──> [COUNTDOWN] ──(3초 경과)──> [RACING]
                                                           │  ▲
                                                (일시정지) │  │ (재개)
                                                           v  │
                                                        [PAUSED]
                                                           │
                                             (3랩 완주)    v
                                                       [FINISHED]
```

```javascript
const GameState = {
  MENU: 'MENU',
  COUNTDOWN: 'COUNTDOWN',
  RACING: 'RACING',
  PAUSED: 'PAUSED',
  FINISHED: 'FINISHED'
};

let currentState = GameState.MENU;

function transitionTo(newState) {
  currentState = newState;
  switch (newState) {
    case GameState.MENU:
      showMenuOverlay();
      break;
    case GameState.COUNTDOWN:
      startCountdownTimer();
      break;
    case GameState.RACING:
      enableVehiclePhysics();
      break;
    case GameState.FINISHED:
      showVictoryModal();
      break;
  }
}
```

---

### 10.6. 보안 및 무결성: 로컬스토리지 랩타임 조작(어뷰징) 방어
- **문제점**:
  - `localStorage.setItem('bestLap', 38.45)` 형태로 기록을 평문 저장하면 브라우저 개발자 도구 콘솔에서 `localStorage.setItem('bestLap', 1.0)` 같은 허위 기록 주입이 매우 간단합니다.
- **보완책**:
  - 서버리스 정적 환경이므로 클라이언트 단에서 난수 솔트(Salt)와 간단한 비트 XOR/해시 체크섬을 생성하여 함께 저장합니다.
  - 로드 시 체크섬이 일치하지 않거나 비현실적인 불가능 랩타임(예: 10초 미만)인 경우 자동으로 기록을 무효화하고 초기화합니다.

```javascript
const STORAGE_KEY = 'steel_racer_record';
const SALT = 'STEEL_RACER_V1_2026';

function saveBestLap(lapTime) {
  // 간단한 체크섬 해시 계산
  const str = lapTime.toFixed(3);
  let hash = 0;
  const combined = str + SALT;
  for (let i = 0; i < combined.length; i++) {
    hash = ((hash << 5) - hash) + combined.charCodeAt(i);
    hash |= 0;
  }
  const payload = btoa(JSON.stringify({ time: lapTime, hash: hash.toString(16) }));
  try {
    localStorage.setItem(STORAGE_KEY, payload);
  } catch (e) {
    console.warn('Storage saving failed', e);
  }
}

function loadBestLap() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return null;
    const data = JSON.parse(atob(raw));
    
    // 무결성 검증
    const combined = data.time.toFixed(3) + SALT;
    let expectedHash = 0;
    for (let i = 0; i < combined.length; i++) {
      expectedHash = ((expectedHash << 5) - expectedHash) + combined.charCodeAt(i);
      expectedHash |= 0;
    }

    // 변조되었거나 물리적으로 불가능한 랩타임(10초 이하) 검출 시 기각
    if (data.hash !== expectedHash.toString(16) || data.time < 10.0) {
      console.warn('Tampered record detected! Resetting best lap.');
      return null;
    }
    return data.time;
  } catch (e) {
    return null;
  }
}
```

---

> 본 계획서의 모든 장애 방어 메커니즘, 점진적 4단계 구현 로드맵, 그리고 6대 고급 최적화/유지보수 전략(섹션 10)을 준수하여 [drive.html](file:///c:/Users/Voltron/Desktop/VScode/new folder/drive.html)의 완벽한 단일 파일 코드를 즉시 작성할 수 있습니다.

