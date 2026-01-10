# 💡 시각장애인을 위한 쇼핑카트 보조기기

## ✍️ 프로젝트 개요
 쇼핑은 일상에서 빠질 수 없는 활동이지만, 시각장애인에게는 이런 일상생활 속에도 어려움과 불편함이 있음을 인식함. **기존 쇼핑카트에 탈부착 가능한 보조 모듈**을 제공함으로써, 시각장애인이 타인의 도움 없이 **독립적으로 쇼핑을 수행**할 수 있도록 지원하는 제품 개발을 목적으로 함.

---
## 🛒 전체 구조 설계도
![제작할 제품의 전체 구성도](all.png)

- 카트 옆면 : Raspberry pi 이용 - 카메라, 푸시버튼, usb 사운드 카드, 이어폰, 보조배터리 사용
- 카트 앞면 : Arduino uno 이용 - 거리센서, 무선통신 모듈 + 어답터, 스위치, aa건전지 4개 사용
- 손목 밴드 : Arduino nano 이용 - 무선통신 모듈 + 어답터, 진동 모듈, 스위치 2개(진동과 전원용), 3.7리튬이온 배터리, DC-DC 승압기 사용

---
## 🎯 주요 기능 (1~4)
### **Raspberry pi + Arduino**를 이용한 기능
 - Raspberry pi를 통해 **상품 인식, 최저가 비교, 도움요청** 기능을 추가
 - Arduino를 이용해 **장애물 감지** 기능을 추가

**Raspberry pi bread board view**
![라즈베리 브레드보드 뷰](bread.png)
- 버튼 1 - 상품 인식 + 유통기한 인식
- 버튼 2 - 최저가 비교
- 버튼 3 - 도움 요청
- usb 포트에 사운드 카드 연결 + 이어폰 연결 -> 음성인식 가능

**Arduino bread board view**
![아두이노 브레드보드 뷰](aa.png)

---
1. ### 🏷️ 상품 및 유통기한 식별 - Raspberry pi ( 코드 변한거 넣기.)
    카메라에 상품을 인식하면 **AI 학습**을 통해 제품명을 알려주고 실시간으로 **유통기한을 판독**하고 상품과 관련한 **DB**를 통해 사용자에게 제품명, 가격, 유통기한 제공. 필요에 따라 제품에 관한 상세정보도 제공. 동일 카테고리내에서 제품 구별.
   
   - 라벨링 된 이미지를 이용하여 AI 학습. (Roboflow, Google colab 이용)
   - **YOLOv8 모델**을 통해 상품을 인식.  
   - **OCR 기술**을 활용해 유통기한을 판독하고, **gTTS 음성 안내**로 사용자에게 정보를 제공합.
   - 상품 인식 후 **데이터베이스**에서 가격, 칼로리, 영양 성분 등 정보를 조회하여 **음성으로 출력**.

👉 상품 인식 기능 세부 내용(코드)
  
	- 상품 인식 파일, 유통기한 추출 파일, DB 파일 모듈화, main파일에서 스크립트 호출
	- 상품은 라면을 중심으로 (너구리, 짜파게티, 신라면, 진라면, 불닭볶음면) 코드 작성
	
<details>
	
   <summary> 상품 YOLOv8 인식 관련 코드 </summary>
	
```python
from picamera2 import Picamera2
import cv2
import os
from gtts import gTTS
import uuid
import subprocess
from ultralytics import YOLO

def speak(text):
    filename = f"/tmp/tts_{uuid.uuid4()}.mp3"
    tts = gTTS(text=text, lang='ko')
    tts.save(filename)
    subprocess.call(f'mpg123 "{filename}"', shell=True)
    os.remove(filename)

speak("상품을 카메라 앞에 나둬주세요.")

model = YOLO("학습시킨 파일 경로")

output_path = "/home/파일 저장 경로/"
os.makedirs(output_path, exist_ok=True)

cam = Picamera2()
cam.configure(
    cam.create_video_configuration(
        main={"format": "XRGB8888", "size": (320, 240)}
    )
)
cam.start()

label_to_kor = {
    "shinramen": "신라면",
    "jinramyun": "진라면",
    "nuguri": "너구리",
    "jjapagetti": "짜파게티",
    "buldakbokkeummyun": "불닭볶음면"
}

print("인식모드 시작")

while True:
    frame = cam.capture_array()
    frame = cv2.cvtColor(frame, cv2.COLOR_RGBA2RGB)

    results = model(frame)
    boxes = results[0].boxes

    if len(boxes) > 0:
        cls = int(boxes.cls[0])
        label = model.names[cls]

        kor = label_to_kor.get(label)
        if kor:
            with open(os.path.join(output_path, "ramen.txt"), "w") as f:
                f.write(label)

            print(f"제품 인식됨: {kor}")
            break

    cv2.waitKey(1)

cam.stop()
cv2.destroyAllWindows()
```
</details>

<details>
   <summary> 유통기한 추출 파일 코드 </summary>
   
```python
from picamera2 import Picamera2
import pytesseract
import cv2
import re
import os
import time
from gtts import gTTS
import uuid
import subprocess


output_path = "/home/파일 저장 경로/"
os.makedirs(output_path, exist_ok=True)

def speak(text):
    filename = f"/tmp/tts_{uuid.uuid4()}.mp3"
    tts = gTTS(text=text, lang='ko')
    tts.save(filename)
    subprocess.call(f'mpg123 "{filename}"', shell=True)
    os.remove(filename)

def extract_date(text):
    m = re.search(r"\d{4}\.\d{2}\.\d{2}", text)
    if m:
        return m.group()
    return None

speak("유통기한 인식을 위해 상품을 돌려주세요.")

cam = Picamera2()
cam.configure(cam.create_video_configuration(main={"format":"XRGB8888","size":(640,480)}))
cam.start()
time.sleep(1)

print("자동 인식 모드 시작")

frame_count = 0
max_attempts = 100

while True:
    frame = cam.capture_array()
    frame = cv2.cvtColor(frame, cv2.COLOR_RGBA2RGB)
    cv2.imshow("OCR", frame)

    frame_count += 1
    if frame_count % 8 != 0:
        if cv2.waitKey(1) == ord('q'):
            break
        continue

    print("시도중...")
    text = pytesseract.image_to_string(frame, lang='kor+eng')
    expiry = extract_date(text)

    if expiry:
        with open(output_path + "expiry.txt", "w") as f:
            f.write(expiry)

        print("유통기한:", expiry)
        break
    else:
        print("유통기한 인식 실패 : 재시도 중”)

    if frame_count > max_attempts:
        speak("유통기한을 찾지 못했습니다. 상품 위치를 조정해주세요.")
        break
	
    if cv2.waitKey(1) == ord('q'):
        break

cam.stop()
cv2.destroyAllWindows()

```
</details>

<details>
   <summary> 음성 및 DB 상세정보 코드 </summary>

```python
# DB만들기

sqlite3 /home/생성할 DB경로

CREATE TABLE ramen_info (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    calories INTEGER,
    sodium INTEGER,
    fat REAL,
    carbohydrate REAL,
    protein REAL
);

INSERT INTO ramen_info (name, calories, sodium, fat, carbohydrate, protein) VALUES
('진라면', 500, 1900, 16, 78, 10),
('신라면', 505, 1890, 17, 80, 9),
('너구리', 510, 1940, 18, 76, 8),
('짜파게티', 600, 1200, 21, 90, 10),
('불닭볶음면', 530, 1600, 20, 85, 11);

.exit  # 나가기

UPDATE ramen_info SET sodium = 1950 WHERE name = '신라면';  # 내용 변경시 명령어
ALTER TABLE ramen_info ADD COLUMN price INTEGER;  # 열 추가 (가격)

SELECT * FROM ramen_info;  # 내용확인
```

```python
import sqlite3
import os
from gtts import gTTS
import time
import speech_recognition as sr
import uuid
import subprocess

def speak(text):
    filename = f"/tmp/tts_{uuid.uuid4()}.mp3"
    tts = gTTS(text=text, lang='ko')
    tts.save(filename)
    subprocess.call(f'mpg123 "{filename}"', shell=True)
    os.remove(filename)
    time.sleep(0.3)


def format_expiry(expiry):
    y, m, d = expiry.split('.')
    return f"{y} 년 {m} 월 {d} 일“

label_to_kor = {
    "shinramen": "신라면”,
    "jinramyun": "진라면“,
    "nuguri": "너구리”,
    "jjapagetti": "짜파게티“,
    "buldakbokkeummyun": "불닭볶음면”
}

def recognize_yes_no():
    r = sr.Recognizer()
    with sr.Microphone(device_index=1) as source:
        speak("상세 정보를 읽을까요? 네 또는 아니요 라고 말씀해주세요")
        try:
            audio = r.listen(source, timeout=5)
            result = r.recognize_google(audio, language="ko-KR")

            YES_KEYWORDS = {"네", "예", "응", "해주세요"}
            if any(keyword in result for keyword in YES_KEYWORDS):
                return "yes"
            elif "아니요" in result:
                return "no“

        except:
            return "unknown"
    return "unknown"

with open("/home/텍스트 파일 경로") as f:
    ramen_label = f.read().strip()

kor_name = label_to_kor.get(ramen_label, ramen_label)

with open("/home/see2407me/result/expiry.txt") as f:
    expiry = f.read().strip()

expiry = format_expiry(expiry)
conn = sqlite3.connect('/home/DB 경로')
cursor = conn.cursor()
cursor.execute("SELECT * FROM ramen_info WHERE name=?", (kor_name,))
row = cursor.fetchone()
conn.close()

speak(f"이 제품은 {kor_name}입니다.")

if row:
    _, name, cal, sodium, fat, carb, protein, price = row
    speak(f”가격은 {price}원 이고, 유통기한은 {expiry}입니다.")
else:
    price_text = ""
time.sleep(1)

print("상세 정보 여부 대기중")

answer = recognize_yes_no()

if answer == "yes":
    if row:
        info = (
        f"{name}의 상세정보입니다. "
        f"열량은 {cal} 킬로칼로리"
        f"탄수화물 {carb}그램"
        f"지방 {fat} 그램,"
        f"나트륨 {sodium}밀리그램"
        f"단백질 {protein}그램입니다."
    )
        speak(info)
    else:
        speak("상세정보를 찾을 수 없습니다.")
elif answer == "no":
    speak("프로그램을 종료합니다.”)
else:
    speak("음성 인식을 실패했습니다. 프로그램을 종료합니다.“)

exit(0)
```
</details>

<details>
   <summary> 상품인식 통합 코드 </summary>

```python
from gpiozero import Button
import subprocess
import os
import time
from gtts import gTTS
import uuid


button = Button(17, pull_up=True, bounce_time=0.3)

def speak(text):
    filename = f"/tmp/tts_{uuid.uuid4()}.mp3"
    tts = gTTS(text=text, lang='ko')
    tts.save(filename)
    subprocess.call(f'mpg123 "{filename}"', shell=True)
    os.remove(filename)


def run_and_wait(cmd):
    return subprocess.call(cmd, shell=True)

first_run = True

while True:

    if first_run:
        print("버튼 누르면 인식 시작.")
        button.wait_for_press()
        first_run = False

    else:
        speak("상품 인식을 계속합니다.")

    speak("상품 인식을 시작합니다. 잠시만 기다려주세요.")


    run_and_wait("python3 /home/상품 인식 코드 경로")
    run_and_wait("python3 /home/유통기한 추출 코드 경로")
    run_and_wait("python3 /home/음성 및 DB 코드 경로")

    speak("상품 인식을 계속 하겠습니까? 계속하려면 5초안에 버튼을 눌러주세요.")

    start = time.time()
    pressed = False

    while time.time() - start < 5:
        if button.is_pressed:
            pressed = True
            break
        time.sleep(0.1)

    if not pressed:
        speak("프로그램을 종료합니다.")
        break
```

</details>

---
2. ### ➕ 최저가 비교 - Raspberry pi
    상품을 인식하게 되면 그 **상품의 최저가를 네이버 검색**을 통해 검색 후 알려줌.

   - **라즈베리 카메라**를 통해 상품을 인식함.
   - 인식한 상품의 **낱개 가격**을 실시간 음성으로 알려줌.

<details>
<summary> 최저가 검색(코드)</summary>

  여기에 접히는 내용을 작성합니다.
  여러 줄도 가능하고 마크다운도 쓸 수 있어요.

```python
  print("예시 코드")
```
</details>

---
3. ### 🚨 도움 요청 안내 - Raspberry pi
    사용자가 도움이 필요할때 버튼을 누르면 **사이트를 통해 알림**을 보냄.

   - 버튼을 통해 **실시간**으로 도움 요청을 확인.
   - **카메라**를 통해 사용자의 위치를 확인.
   - **소요 시간**과 **도움 요청 사유**를 기록할 수 있음.

<details>
<summary>도움 요청 서버,사이트 (코드)</summary>

  여기에 접히는 내용을 작성합니다.
  여러 줄도 가능하고 마크다운도 쓸 수 있어요.

```python
  print("예시 코드")
```
</details>

<details>
<summary>도움 요청 디바이스 (코드)</summary>

  여기에 접히는 내용을 작성합니다.
  여러 줄도 가능하고 마크다운도 쓸 수 있어요.

```python
  print("예시 코드")
```
</details>

#### 📝 Raspberry pi의 메인 코드 (상품 인식 + 최저가 비교 + 도움요청)
- 버튼을 통해 원하는 기능을 불러오도록 하는 코드 (호출)

<details>
<summary>전체 main 코드</summary>

```python
from gpiozero import Button
import subprocess
import time
import socket
from gtts import gTTS
import os
import uuid
import subprocess

BTN_SCAN = Button(17, pull_up=True, bounce_time=0.3)
BTN_PRICE = Button(27, pull_up=True, bounce_time=0.3)
BTN_HELP = Button(22, pull_up=True, bounce_time=0.3)

SERVER_PATH = "/home/서버 경로"
SERVER_IP = "서버 아이피"
SERVER_PORT = 5000
SERVER_URL = f"http://{SERVER_IP}:{SERVER_PORT}"


def speak(text):
    filename = f"/tmp/tts_{uuid.uuid4()}.mp3"
    tts = gTTS(text=text, lang='ko')
    tts.save(filename)
    subprocess.call(f'mpg123 "{filename}"', shell=True)
    os.remove(filename)

def is_port_open(ip, port):
    s = socket.socket()
    try:
        s.connect((ip, port))
        s.close()
        return True
    except:
        return False

if not is_port_open(SERVER_IP, SERVER_PORT):
    print("서버가 실행 중이 아니므로 시작합니다...")
    subprocess.Popen(["python3", SERVER_PATH])
    time.sleep(2)
else:
    print("서버가 이미 실행 중입니다.")

print("버튼 대기 중 (17: 상품, 27: 최저가, 22: 도움 요청)")

while True:
    if BTN_SCAN.is_pressed:
        print("상품 인식 시작")
        speak("상품인식을 시작합니다 잠시만 기다려주세요.")
        subprocess.call(["python3", "/home/상품인식 메인 경로"])
        time.sleep(1)

    elif BTN_PRICE.is_pressed:
        print("최저가 비교 시작")
        speak("최저가 비교를 시작합니다. 잠시만 기다려주세요.")
        subprocess.call(["python3", "/home/최저가 비교 경로"])
        time.sleep(1)

    elif BTN_HELP.is_pressed:
        print("도움 요청 실행 (device1.py)")
        speak("도움 요청을 보냈습니다. 응답이 올때까지 기다려주세요.")
        # 도움 요청 디바이스 파일을 백그라운드로 실행
        subprocess.Popen(["python3", "/home/도움요청 디바이스 경로"])
        time.sleep(1)

    time.sleep(0.1)
```
</details>

---
4. ### 🚧 실시간 장애물 감지- Arduino
    장애물이 카트 앞에 위치할 경우 임계거리를 설정하여 임계거리 안으로 들어오면, 손목밴드의 진동을 이용해 사용자에게 알림. **장애물 거리에 따라 진동 속도를 다르게 하여 위험도**를 표현.

   - **아두이노 우노 + 거리 센서**를 활용해 전방 장애물 거리 측정.
   - **무선통신모듈 + 어답터**을 통해 장애물에 따른 신호 송수신.
   - **아두이노 나노 + 진동모듈**을 이용하여 사용자에게 실시간으로 위험도를 알림.
   - **스위치**를 통해 진동 **온오프** 및 **전원 온오프**

<details>
<summary>장애물 감지 기능 송신단(코드)</summary>

  거리 센서에서 장애물 측정 후 신호 송신.
  **최대거리 임계 거리 800cm** 이하면 장애물이 있음을 인식 .

```python
#include <Wire.h>
#include <Adafruit_VL53L0X.h>
#include <SPI.h>
#include <nRF24L01.h>
#include <RF24.h>

Adafruit_VL53L0X lox = Adafruit_VL53L0X();

// nRF24
RF24 radio(9, 10);                 // CE, CSN
const byte address[6] = "00001";

// 최대 거리 (이 이상은 진동 X)
const int MAX_DISTANCE_MM = 800;   // 80 cm

void setup() {
  Serial.begin(9600);
  Serial.println("TX START");

  Wire.begin();
  if (!lox.begin()) {
    Serial.println("VL53L0X 인식 실패");
    while (1);
  }
  Serial.println("VL53L0X OK");

  radio.begin();
  radio.openWritingPipe(address);
  radio.setPALevel(RF24_PA_LOW);
  radio.stopListening();            // 송신 모드
}

void loop() {
  VL53L0X_RangingMeasurementData_t measure;
  lox.rangingTest(&measure, false);

  if (measure.RangeStatus != 4) {   // 정상 측정
    int distance = measure.RangeMilliMeter;

    // 너무 먼 값은 컷
    if (distance > MAX_DISTANCE_MM) {
      distance = MAX_DISTANCE_MM;
    }

    radio.write(&distance, sizeof(distance));

    Serial.print("Distance TX: ");
    Serial.print(distance);
    Serial.println(" mm");
  }

  delay(100);
}

```
</details>

<details>
<summary>장애물 감지 기능 수신단(코드)</summary>

  송신단에서 받은 신호로 진동모듈을 통해 사용자에세 장애물 알림.
  거리에 따라 진동 횟수 변동하여 가까운 장애물을 알려줌.

```python
#include <SPI.h>
#include <nRF24L01.h>
#include <RF24.h>

#define CE_PIN     9
#define CSN_PIN    10
#define MOTOR_PIN  3 // 스위치가 모터에 직접 연결되었으므로 SWITCH_PIN은 사용하지 않습니다.

RF24 radio(CE_PIN, CSN_PIN);
const byte address[6] = "00001";

const int MIN_DISTANCE = 50;     // 5 cm
const int MAX_DISTANCE = 800;    // 80 cm
const unsigned long MIN_INTERVAL = 80;
const unsigned long MAX_INTERVAL = 600;

unsigned long lastToggleTime = 0;
unsigned long vibrationInterval = 0;

bool motorState = false;

int currentDistance = MAX_DISTANCE;
bool hasReceived = false;

void initRadio() {

  if (radio.begin()) {
    radio.openReadingPipe(0, address);
    radio.setPALevel(RF24_PA_LOW);

    radio.startListening();

    Serial.println(">>> Radio Initialized & Listening...");
  } else {
    Serial.println(">>> Radio Hardware Error!");
  }
}

void setup() {

  Serial.begin(9600);
  pinMode(MOTOR_PIN, OUTPUT);
  digitalWrite(MOTOR_PIN, LOW);

  initRadio();
  Serial.println("RX SYSTEM START (Motor Switch Mode)");
}
void loop() {

  // 1. 📡 데이터 수신 시도 (항상 작동)
  if (radio.available()) {
    radio.read(&currentDistance, sizeof(currentDistance));
    hasReceived = true;

    if (currentDistance >= MAX_DISTANCE) {
      vibrationInterval = 0;

    } else {
      vibrationInterval = map(currentDistance, MIN_DISTANCE, MAX_DISTANCE, MIN_INTERVAL, MAX_INTERVAL);
      vibrationInterval = constrain(vibrationInterval, MIN_INTERVAL, MAX_INTERVAL);
    }
    Serial.print("Dist: "); Serial.print(currentDistance);
    Serial.print("mm | Inter: "); Serial.println(vibrationInterval);
  }

  // 2. 🔔 진동 신호 출력 (모터 스위치와 상관없이 아두이노는 신호를 계속 보냄)
  if (hasReceived && vibrationInterval > 0) {
    unsigned long now = millis();
    if (now - lastToggleTime >= vibrationInterval) {
      lastToggleTime = now;
      motorState = !motorState;
      digitalWrite(MOTOR_PIN, motorState);
    }

  } else {
    digitalWrite(MOTOR_PIN, LOW);
    motorState = false;
  }
}

```
</details>

---
## 완성 모형
### 시연 영상 및 사진
---

## 보안할 점

---
## 기대 효과

- **시각장애인의 자율적인 쇼핑 환경 제공**  
- **충돌 방지를 통한 안전성 확보**  
- **사회적 약자 배려 기술로서의 확장성**  
- **다양한 유통 환경에 적용 가능한 범용 모듈**

---
