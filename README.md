# 🐍 웹캠으로 조종하는 뱀게임 — 강의노트

> **Teachable Machine으로 만든 이미지 분류 모델을 웹 게임의 컨트롤러로 쓰기**
>
> 작업일: 2026-10-07 · 결과물: `index.html` 한 개 파일

---

## 📌 0. 한눈에 보기

| 항목 | 내용 |
|---|---|
| 목표 | 손동작/자세(웹캠 이미지)로 뱀게임의 방향을 바꾼다 |
| 모델 | Teachable Machine 이미지 모델 (`kHqrwGEzw`) |
| 클래스 | `left`, `right`, `up`, `down`, `neut` |
| 기술 | HTML + CSS + JavaScript, TensorFlow.js, Teachable Machine 라이브러리, Canvas |
| 실행 | 로컬 서버(`python -m http.server`) 또는 GitHub Pages |
| 저장소 | https://github.com/higogo1/webcam-snake-game |

```
 [웹캠] ──▶ [Teachable Machine 모델] ──▶ 클래스 + 확률 ──▶ [방향 결정] ──▶ [뱀게임]
            (TensorFlow.js, 브라우저 안에서 실행)     임계값 검사      180° 반전 금지
```

---

## 🎯 1. 학습 목표

이번 시간이 끝나면 다음을 할 수 있다.

1. Teachable Machine 모델을 **URL만으로** 웹 페이지에 불러올 수 있다.
2. 웹캠 영상을 매 프레임 **예측(predict)** 하고 결과를 화면에 보여줄 수 있다.
3. 예측 결과를 **확률 임계값**으로 걸러서 게임 입력으로 바꿀 수 있다.
4. Canvas로 간단한 **뱀게임 루프**를 만들 수 있다.
5. 만든 코드를 **git → GitHub**에 올릴 수 있다.

---

## 🧩 2. 전체 구조

`index.html` 한 파일 안에 세 덩어리가 있다.

| 영역 | 하는 일 |
|---|---|
| **HTML/CSS** | 게임 캔버스, 웹캠 화면, 확률 막대, 슬라이더, 버튼 배치 |
| **Teachable Machine 파트** | 모델 로드 → 웹캠 시작 → 예측 루프 → 방향 요청 |
| **Snake Game 파트** | 뱀/먹이 상태, 한 칸 이동(`step`), 그리기(`draw`), 시작/일시정지/게임오버 |

두 파트는 **`requestDirection(방향)`** 함수 하나로만 연결된다.
→ 웹캠이든 키보드든 "방향을 요청"만 하면 되므로 입력 방식을 쉽게 바꿀 수 있다.

---

## 🤖 3. Teachable Machine 모델 불러오기

### 3-1. 라이브러리 추가

```html
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@1.3.1/dist/tf.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@teachablemachine/image@0.8/dist/teachablemachine-image.min.js"></script>
```

- 1번째: **TensorFlow.js** — 브라우저에서 딥러닝 모델을 돌리는 엔진
- 2번째: **tmImage** — Teachable Machine 모델을 쉽게 쓰게 해 주는 도우미

### 3-2. 모델 + 웹캠 준비

```js
const MODEL_URL = "https://teachablemachine.withgoogle.com/models/kHqrwGEzw/";

model  = await tmImage.load(MODEL_URL + "model.json", MODEL_URL + "metadata.json");
labels = model.getClassLabels();          // ["left", "right", "up", "down", "neut"]

webcam = new tmImage.Webcam(224, 224, true); // 가로, 세로, flip(거울모드)
await webcam.setup();                      // 카메라 권한 요청
await webcam.play();
```

| 파일 | 내용 |
|---|---|
| `model.json` | 신경망 구조 + 가중치 파일 위치 |
| `metadata.json` | 클래스 이름 등 부가 정보 |

> 💡 **224×224** 인 이유: Teachable Machine 이미지 모델(MobileNet 기반)이 이 크기의 입력을 받도록 학습되어 있다.

### 3-3. 예측 루프

```js
async function predictLoop() {
  webcam.update();                              // 새 프레임 캡처
  const preds = await model.predict(webcam.canvas);
  // preds = [{className:"left", probability:0.93}, ...]
  ...가장 확률 높은 클래스(best)를 찾는다...
  requestAnimationFrame(predictLoop);           // 다음 화면 갱신 때 또 실행
}
```

---

## 🎚️ 4. 예측 결과 → 게임 입력으로 바꾸기

### 4-1. 임계값(threshold)으로 거르기

```js
if (best.probability >= th) {          // 기본 0.80
  if (DIRS[label]) requestDirection(label);   // left/right/up/down
  // neut 이면 아무것도 안 함 → 현재 방향 유지
}
```

| 상황 | 결과 |
|---|---|
| `left` 93% (≥ 0.80) | ⬅️ 왼쪽으로 방향 전환 요청 |
| `up` 61% (< 0.80) | 무시 (확신이 부족함) |
| `neut` 95% | 현재 방향 유지 |

- 임계값 **↑** → 오작동 줄어듦, 대신 반응이 둔해짐
- 임계값 **↓** → 반응 빠름, 대신 엉뚱하게 꺾일 수 있음
- 화면의 슬라이더로 0.50 ~ 0.99 사이에서 실시간 조절 가능

### 4-2. 왜 `neut` 클래스가 필요할까?

클래스가 방향 4개뿐이면, 아무 동작도 안 할 때도 모델은 **4개 중 하나를 억지로 고른다.**
"가만히 있음"을 따로 학습시켜 두면 → 의도하지 않은 방향 전환을 막을 수 있다.

### 4-3. 180° 반전 금지

```js
function requestDirection(name) {
  const d = DIRS[name];
  if (d.x === -dir.x && d.y === -dir.y) return;   // 정반대면 무시
  nextDir = d;
}
```

오른쪽으로 가는 중에 `left`가 들어오면 바로 자기 몸에 부딪힌다 → 무시한다.

### 4-4. 방향을 바로 바꾸지 않고 `nextDir`에 저장하는 이유

예측은 **1초에 수십 번**, 뱀 이동은 **1초에 몇 번**이다.
예측할 때마다 방향을 바로 바꾸면 한 칸 이동하는 사이에 방향이 여러 번 바뀌어 꼬인다.
→ "다음에 갈 방향"만 기억해 뒀다가 **이동하는 순간(`step`)에 한 번만 적용**한다.

```
예측 루프 (빠름, rAF)   : left left left up up up up ...   → nextDir 갱신
게임 루프 (느림, 150ms) :        step        step        → dir = nextDir
```

---

## 🕹️ 5. 뱀게임 로직

### 5-1. 데이터 표현

```js
const CELLS = 20;                       // 20×20 격자
snake = [{x:10,y:10}, {x:9,y:10}, {x:8,y:10}];  // [0]이 머리
dir   = { x: 1, y: 0 };                 // 오른쪽
```

| 방향 | x | y |
|---|---|---|
| up | 0 | -1 |
| down | 0 | 1 |
| left | -1 | 0 |
| right | 1 | 0 |

> 💡 Canvas 좌표는 **아래로 갈수록 y가 커진다**. 그래서 위쪽이 `y: -1`.

### 5-2. 한 칸 이동 (`step`)

1. `dir = nextDir` (예약된 방향 적용)
2. 새 머리 위치 = 현재 머리 + 방향
3. 벽 또는 자기 몸에 부딪혔으면 → **게임오버**
4. 머리를 배열 앞에 추가(`unshift`)
5. 먹이를 먹었으면 → 점수 +1, 새 먹이 / 아니면 → 꼬리 제거(`pop`)

> 💡 "머리 추가 + 꼬리 제거" = 움직임, "머리 추가만" = 길어짐

### 5-3. 부가 기능

| 기능 | 구현 방법 |
|---|---|
| 속도 5단계 | `setInterval` 간격 260ms ~ 80ms |
| 최고 점수 저장 | `localStorage` (브라우저를 닫아도 유지) |
| 일시정지/재시작 | 버튼 또는 `Space` |
| 키보드 조작 | 방향키 → 똑같이 `requestDirection()` 호출 |

---

## ▶️ 6. 실행 방법

웹캠(`getUserMedia`)은 **보안 컨텍스트**(https 또는 localhost)에서만 안정적으로 작동한다.
파일을 더블클릭(`file://`)해서 열면 카메라가 안 켜질 수 있다.

```bash
cd C:\Users\sejin\Desktop\snake
python -m http.server 8000
# 브라우저에서 http://localhost:8000
```

1. **📷 웹캠 & 모델 시작** → 카메라 권한 허용
2. **▶ 게임 시작** (또는 Space)
3. 동작으로 조종! 확률 막대가 초록색이면 그 동작이 인식된 것

---

## 🚀 7. GitHub에 올리기

```powershell
git init -b main
git add index.html
git commit -m "Add webcam-controlled snake game"
gh repo create webcam-snake-game --public --source . --remote origin --push
```

| 명령 | 의미 |
|---|---|
| `git init -b main` | 현재 폴더를 git 저장소로 만들고 기본 브랜치를 `main`으로 |
| `git add` | 커밋할 파일을 올려 둠 (스테이징) |
| `git commit` | 변경 내용을 기록(스냅샷) |
| `gh repo create ... --push` | GitHub에 저장소를 만들고 바로 업로드 |

> ⚠️ 이번에 겪은 문제: `gh` 명령이 **PATH에 없어서** 인식되지 않았다.
> → `C:\Program Files\GitHub CLI\gh.exe` 전체 경로로 실행해서 해결.
> (영구 해결: 시스템 환경 변수 PATH에 `C:\Program Files\GitHub CLI\` 추가 후 터미널 재시작)

---

## 🛠️ 8. 문제 해결 (Troubleshooting)

| 증상 | 원인 / 해결 |
|---|---|
| 카메라가 안 켜짐 | `file://`로 열었음 → 로컬 서버나 https로 실행 |
| 왼쪽/오른쪽이 반대로 인식됨 | 학습 때와 거울모드가 다름 → `new tmImage.Webcam(224, 224, true)`의 `true` ↔ `false` |
| 뱀이 제멋대로 꺾임 | 임계값을 올리거나, `neut` 샘플을 더 많이 학습 |
| 동작해도 반응이 없음 | 임계값을 내리거나, 조명·배경을 학습 때와 비슷하게 |
| 모델 로드 오류 | 인터넷 연결 확인, 모델 URL 끝에 `/` 가 있는지 확인 |

---

## 📚 9. 용어 정리

### 머신러닝

| 용어 | 뜻 |
|---|---|
| **Teachable Machine** | 구글이 만든 웹 도구. 코딩 없이 이미지/소리/자세를 학습시켜 모델을 만들 수 있다 |
| **모델 (Model)** | 데이터로 학습된 "판단 규칙 덩어리". 입력(이미지)을 받아 출력(클래스)을 낸다 |
| **클래스 (Class)** | 모델이 구분하는 정답 종류. 여기선 `left/right/up/down/neut` |
| **학습 (Training)** | 예시 데이터를 보여 주며 모델을 만드는 과정 |
| **추론 / 예측 (Inference / Predict)** | 학습된 모델에 새 입력을 넣어 결과를 얻는 과정 |
| **확률 (Probability)** | 각 클래스일 가능성(0~1). 모든 클래스의 합 = 1 |
| **임계값 (Threshold)** | "이 값 이상일 때만 믿겠다"는 기준선 |
| **TensorFlow.js** | 브라우저/Node.js에서 머신러닝을 돌리는 자바스크립트 라이브러리 |
| **MobileNet** | 가볍고 빠른 이미지 분류 신경망. Teachable Machine 이미지 모델의 바탕 |
| **전이 학습 (Transfer Learning)** | 이미 학습된 모델을 가져와 마지막 부분만 새로 학습시키는 방법. 적은 데이터로도 학습 가능 |

### 웹 개발

| 용어 | 뜻 |
|---|---|
| **Canvas** | 자바스크립트로 그림을 그리는 HTML 요소 (`<canvas>`) |
| **CDN** | 라이브러리 파일을 인터넷에서 바로 불러오게 해 주는 서버 (jsDelivr 등) |
| **`async` / `await`** | 시간이 걸리는 작업(모델 로딩, 카메라 켜기)이 끝날 때까지 기다리는 문법 |
| **`requestAnimationFrame` (rAF)** | 화면이 다시 그려질 때마다(보통 초당 60번) 함수를 실행 |
| **`setInterval`** | 정해진 시간마다 함수를 반복 실행 (게임 속도) |
| **`getUserMedia`** | 웹캠·마이크에 접근하는 브라우저 API |
| **보안 컨텍스트 (Secure Context)** | https 또는 localhost. 카메라 같은 민감한 기능은 여기서만 허용 |
| **`localStorage`** | 브라우저에 작은 데이터를 영구 저장하는 공간 (최고 점수 저장) |
| **로컬 서버** | 내 컴퓨터에서 웹 페이지를 서비스하는 서버 (`python -m http.server`) |
| **거울 모드 (Flip)** | 웹캠 화면을 좌우 반전해서 거울처럼 보이게 하는 것 |

### 게임 로직

| 용어 | 뜻 |
|---|---|
| **게임 루프 (Game Loop)** | 일정 간격으로 "상태 갱신 → 화면 그리기"를 반복하는 구조 |
| **틱 (Tick)** | 게임 루프 한 번. 여기선 뱀이 한 칸 움직이는 단위 |
| **격자 (Grid)** | 화면을 같은 크기의 칸으로 나눈 것 (20×20) |
| **충돌 판정 (Collision)** | 머리가 벽이나 몸과 같은 칸인지 검사 |
| **입력 버퍼 (`nextDir`)** | 다음 틱에 적용할 입력을 잠시 저장해 두는 변수 |

### Git / GitHub

| 용어 | 뜻 |
|---|---|
| **Git** | 파일 변경 이력을 관리하는 버전 관리 도구 |
| **저장소 (Repository)** | Git이 관리하는 프로젝트 폴더 |
| **커밋 (Commit)** | 변경 내용을 하나의 기록으로 저장하는 것 |
| **브랜치 (Branch)** | 독립적인 작업 줄기. 기본은 `main` |
| **원격 (Remote) / origin** | GitHub처럼 다른 곳에 있는 저장소. `origin`은 기본 원격 이름 |
| **푸시 (Push)** | 내 커밋을 원격 저장소로 업로드 |
| **GitHub CLI (`gh`)** | 터미널에서 GitHub 작업(저장소 생성 등)을 하는 공식 도구 |
| **PATH** | 명령어 이름만 쳐도 실행되도록 프로그램 위치를 등록해 두는 환경 변수 |
| **GitHub Pages** | 저장소의 HTML을 무료로 웹사이트(https)로 공개해 주는 기능 |

---

## ✅ 10. 오늘의 핵심 요약

1. Teachable Machine 모델은 **URL + `tmImage.load()`** 한 줄이면 웹에서 쓸 수 있다.
2. 예측은 `requestAnimationFrame`으로 **계속**, 게임은 `setInterval`로 **일정 간격**으로 돌린다.
3. 둘 사이는 **`nextDir`(입력 버퍼)** 로 연결해서 타이밍 차이를 흡수한다.
4. **임계값 + `neut` 클래스**가 오작동을 막는 핵심 장치다.
5. 웹캠은 **https / localhost**에서만 확실하게 동작한다.

## 🔭 11. 더 해볼 것

- [ ] GitHub Pages로 배포해서 https 주소로 바로 플레이
- [ ] 같은 클래스가 N프레임 연속일 때만 방향 전환 (흔들림 방지)
- [ ] 벽 통과 모드, 장애물, 레벨업 시 속도 증가
- [ ] 소리 모델(Teachable Machine Audio)로 "위/아래" 음성 조종
