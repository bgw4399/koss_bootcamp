# 3주차 과제 영상

https://user-images.githubusercontent.com/104902657/184536492-9d15a2a9-8633-46ee-b755-a483d628053f.mp4

# 3주차 과제 app.js
```
const express = require("express");
// express를 요청
const app = express();
// express의 리턴값을 app에 넣음
const path = require("path");
// path를 요청
const bodyParser = require("body-parser");
//body-parser을 요청
const mqtt = require("mqtt");
//mqtt(mosquitto)를 mqtt에 넣음
const http = require("http");
//http를 const http에 넣음 
const mongoose = require("mongoose");
//mongoose를 mongoose 객체로 불러옴
const Sensors = require("./models/sensors");
//model 파일에 sensors.js를 Sensors에 넣음
const devicesRouter = require("./routes/devices");
//routes파일에 devices.js를 devicesRouter에 넣음
require("dotenv/config");
//dotenv를 통해 현 디렉토리에 위치한 .env 파일로부터 환경변수를 읽어 낸다.

app.use(express.static(__dirname + "/public"));
//express를 public 디렉토리에서 실행
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: false }));
//bodyparser을 이용하기 위해 만든 코드
app.use("/devices", devicesRouter);
//devices를 불러옴

const client = mqtt.connect("mqtt://192.168.46.88"); 
// 라즈베리파이 url에 mqtt를 이용해 연결
client.on("connect", () => {
  console.log("mqtt connect");
  //console에 mqtt connect가 뜨는지 확인.
  client.subscribe("sensors");
  // mqtt를 통해 sensors라는 데이터베이스로 받음
});

client.on("message", async (topic, message) => {
  var obj = JSON.parse(message);
  var date = new Date(); // 현재 시간을 불러옴
  var year = date.getFullYear(); // 현재 연도를 불러옴
  var month = date.getMonth(); // 현재 월을 불러옴
  var today = date.getDate(); // 현재 일을 불러옴
  var hours = date.getHours(); // 현재 시를 불러옴
  var minutes = date.getMinutes(); // 현재 분을 불러옴
  var seconds = date.getSeconds(); // 현재 초를 불러옴
  obj.created_at = new Date(
    Date.UTC(year, month, today, hours, minutes, seconds)
  );  // year, month, today, hours, minutes, seconds 정보를 UTC로 취급한 obj를 만듬
  // console.log(obj);

  const sensors = new Sensors({
    tmp: obj.tmp,
    // mongodb에 보낼 온도 저장
    hum: obj.humi,
    // mongodb에 보낼 습도 저장
    pm1: obj.pm1,
    // mongodb에 보낼 pm1 저장
    pm2: obj.pm25,
    // mongodb에 보낼 pm2 저장
    pm10: obj.pm10,
    // mongodb에 보낼 pm10 저장
    created_at: obj.created_at,
    // mongodb에 보낼 시간 저장
  });

  try {
    // 센서 정보 저장 시도
    const saveSensors = await sensors.save();
    // 센서 정보 저장ㅌ
    console.log("insert OK");
    // 연결 시 insert OK를 띄움
  } catch (err) {  
    console.log({ message: err });
    // 에러 발생시 err메세지를 띄움
  }
});
app.set("port", "3000");
//3000 포트 설정
var server = http.createServer(app);
// 서버 생성
var io = require("socket.io")(server);
// 소켓 서버 생성
io.on("connection", (socket) => {
  //웹에서 소켓을 이용한 sensors 센서데이터 모니터링
  socket.on("socket_evt_mqtt", function (data) {
    Sensors.find({})
    // 센서 정보 불러옴
      .sort({ _id: -1 })
      // 최신 정보
      .limit(1)
      // 1개 조회
      .then((data) => {
        //console.log(JSON.stringify(data[0]));
        socket.emit("socket_evt_mqtt", JSON.stringify(data[0]));
        // 서버가 현재 접속해 있는 모든 클라이언트에 이벤트 전송
      });
  });
  //웹에서 소켓을 이용한 LED ON/OFF 제어하기
  socket.on("socket_evt_led", (data) => {
    // 소켓 led 이벤트 실행 시
    var obj = JSON.parse(data);
    // data의 json 문자열의 구문을 분석 한 값을 객체로 생성 
    client.publish("led", obj.led + "");
    // 토픽 'led'로 정보를 보냄
  });
});
//웹서버 구동 및 DATABASE 구동
server.listen(3000, (err) => {
  if (err) {
    // err 발생시
    return console.log(err);
    // err를 로그로 띄움
  } else {
    console.log("server ready");
    // 성공 시 server ready를 띄움
    mongoose.connect(
      process.env.MONGODB_URL,
      // env의 url로 진행
      { useNewUrlParser: true, useUnifiedTopology: true },
      // DB 옵션
      () => console.log("connected to DB!")
      //Connection To DB
    );
  }
});

```
# 3주차 과제 devices.js
```
var express = require("express");
// express의 리턴값을 app에 넣음
var router = express.Router();
// 모듈식 마운팅 가능한 핸들러를 작성
const mqtt = require("mqtt");
// mqtt 요청
const Sensors = require("../models/sensors");
// sensors 정보 불러옴
// MQTT Server 접속
const client = mqtt.connect("mqtt://172.20.10.12"); // 라즈베리파이 url
//웹에서 rest-full 요청받는 부분(POST)
router.post("/led", function (req, res, next) {
  res.set("Content-Type", "text/json");
  // 응답 정보 설정
  if (req.body.flag == "on") {
    // MQTT->led : 1
    client.publish("led", "1");
    res.send(JSON.stringify({ led: "on" }));
    // 정보 1을 보내 led on 전송
    } else {
    client.publish("led", "2");
    res.send(JSON.stringify({ led: "off" }));
      // 정보 2을 보내 led off 전송
  }
});
module.exports = router;
// 모듈 내보냄

```
#3주차 MQTT.HTML
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Insert title here</title>
    <script type="text/javascript" src="/socket.io/socket.io.js"></script>
    <script src="http://code.jquery.com/jquery-3.3.1.min.js"></script>
    <script type="text/javascript">
      var socket = null;
      // 소켓 초기화
      $;
      var timer = null;
      // timer 초기화
      $(document).ready(function () {
        socket = io.connect(); // 3000port
        // Node.js보낸 데이터를 수신하는 부분
        socket.on("socket_evt_mqtt", function (data) {
          data = JSON.parse(data);
          // data의 json 문자열의 구문을 분석 한 값을 객체로 생성  
          console.log(data);
          $(".mqttlist").html(
            "<li>" +
              " tmp: " +
              data.tmp + // 온도 띄우기
              "°C" +
              " hum: " +
              data.hum + // 습도 띄우기
              "%" +
              " pm1: " +
              data.pm1 + // pm1 띄우기
              " pm2.5: " +
              data.pm2 + // pm2 띄우기
              " pm10: " +
              data.pm10 + // pm10 띄우기
              "</li>"
          );
        });
        if (timer == null) {
          timer = window.setInterval("timer1()", 1000);
          // timer가 없을 시 1초마다 timer1 함수를 실행
        }
      });
      function timer1() {
        socket.emit("socket_evt_mqtt", JSON.stringify({}));
        // 서버가 현재 접속해 있는 모든 클라이언트에 이벤트 전송
        console.log("---------");
        // ---------를 띄움
      }
      function ledOnOff(value) {
        // {"led":1}, {"led":2}
        socket.emit("socket_evt_led", JSON.stringify({ led: Number(value) }));
        // 서버가 현재 접속해 있는 모든 클라이언트에 이벤트 전송
      }
      function ajaxledOnOff(value) {
        if (value == "1") var value = "on";
        // 1일 시 led를 on으로 설정
        else if (value == "2") var value = "off";
        // 2일 시 led를 off로 설정
        $.ajax({
          url: "http://localhost:3000/devices/led", //local url
          type: "post", // post 설정
          data: { flag: value }, //flag에 value 설정
          success: ledStatus,
          // 성공 시 ledStatus 실행
          error: function () {
            alert("error");
            // error 발생 시 error을 알림
          },
        });
      }
      function ledStatus(obj) {
        $("#led").html("<font color='red'>" + obj.led + "</font> 되었습니다.");
        // 빨간 색깔로 led의 상태를 보여줌
      }
    </script>
  </head>
  <body>
    <h2>socket 이용한 센서 모니터링 서비스</h2>
    <div id="msg">
      <div id="mqtt_logs">
        <ul class="mqttlist"></ul>
      </div>
      <h2>socket 통신 방식(LED제어)</h2>
      <button onclick="ledOnOff(1)">LED_ON</button>
      <button onclick="ledOnOff(2)">LED_OFF</button>
      <h2>RESTfull Service 통신 방식(LED제어)</h2>
      <button onclick="ajaxledOnOff(1)">LED_ON</button>
      <button onclick="ajaxledOnOff(2)">LED_OFF</button>
      <div id="led">LED STATUS</div>
    </div>
  </body>
</html>

# 4주차 과제 영상

https://user-images.githubusercontent.com/104902657/184536718-860e9e71-9846-4d79-8203-e8484c52fad6.mp4

# 4주차 과제 코드
```
import sys
import time
import numpy as np
from PyQt5.QtWidgets import QApplication, QMainWindow, QWidget, QVBoxLayout, QLabel, QHBoxLayout, QVBoxLayout
from matplotlib.backends.backend_qt5agg import FigureCanvasQTAgg as FigureCanvas
from matplotlib.backends.backend_qt5agg import NavigationToolbar2QT as NavigationToolbar
from matplotlib.figure import Figure
from pymongo import MongoClient
from PyQt5.QtGui import QIcon, QPixmap, QFont
from PyQt5.QtCore import Qt
from PyQt5.QtGui import QFontDatabase
from PyQt5.QtGui import QFont


client = MongoClient("mongodb+srv://bgw4399:qowlsdn4399@cluster0.2f7mb.mongodb.net/?retryWrites=true&w=majority")
# 내 mongodb의 client를 불러옴

db = client['test'] # test라는 이름의 데이터베이스에 접속


class MyApp(QMainWindow):

  def __init__(self):
      super().__init__()
      #다른 클래스의 속성 및 메소드를 자동으로 불러와 해당 클래스에서도 사용이 가능
    #   count = []
    #   pm2 = []
    #   time = []
    #   for d, cnt in zip(db['sensors'].find(), range(10, 0, -1)):
    #     count.append(cnt)
    #     pm2.append(int(d['pm2']))
    #     tm = str(d['created_at'])
    #     time.append(tm[11:])
        
      self.main_widget = QWidget()
      self.setCentralWidget(self.main_widget) # 메인 위젯

    #   canvas = FigureCanvas(Figure(figsize=(4, 3))) # 그래프를 그리기 위한 도화지 준비 // figsize -> 도화지 크기
      self.vbox = QVBoxLayout(self.main_widget)  # verticalbox 레이아웃
      # 수직 박스 생성
      self.label1 = QLabel()
      self.image = QLabel()
      self.label2 = QLabel()
      self.label3 = QLabel()
      self.label4 = QLabel()

      self.Chbox = QHBoxLayout()
      # 수평 박스 생성
      
    #   self.addToolBar(NavigationToolbar(canvas, self
      

    #   self.ax = canvas.figure.subplots()  # 좌표축 준비
    #   self.ax.plot(time, pm2, '-') # 점 찍기(x좌표, y좌표, 뭘로 그릴건지)

      self.dynamic_canvas = FigureCanvas(Figure(figsize=(4, 3)))  # 움직이는 그래프를 위한 도화지
    #   vbox.addWidget(dynamic_canvas)  # 도화지 넣기

      self.dynamic_ax = self.dynamic_canvas.figure.subplots()  # 좌표축 준비
      self.timer = self.dynamic_canvas.new_timer(
          1000, [(self.update_canvas, (), {})])  # 그래프가 변하는 주기를 위한 타이머 준비 (1초, 실행되는 함수)
      self.timer.start()  # 타이머 시작

      self.vbox.addWidget(self.label1)
      # 라벨 1을 수직 박스에 넣음
      self.vbox.addWidget(self.label3)
      # 라벨 3을 수직 박스에 넣음
      self.vbox.addWidget(self.label4)
      # 라벨 4을 수직 박스에 넣음
      self.vbox.addLayout(self.Chbox)
      # CHbox를 수직 박스에 넣음
      self.vbox.addWidget(self.dynamic_canvas)
      # 움직이는 그래프를 수직 박스에 넣음
        
      # self.box.addLayout(self.box1)
      # self.box.addLayout(vbox)
      # self.setLayout(self.box)
      self.setWindowTitle('실내 미세먼지 농도 측정기')
      # 창 이름을 바꿈띄어줌
      self.setGeometry(300, 100, 600, 600)
      # x축으로부터 300거리만큼 y축으로 100거리만큼 떨어진 곳에 창을 x축으로 600 거리만큼, y축으로 600거리 크기의 창을 생성
      self.show()
      # 창을 띄어줌

  def update_canvas(self):  # 그래프 변화 함수
      self.dynamic_ax.clear()
      # 리스트 초기화
      count = []
      pm1 = []
      pm2 = []
      pm10 = []
      time = []
      tmp = []
      for i in db['sensors'].find():
        tmp.append(i)
      # sensors에서 정보를 tmp로 다 추가함
      tmp = tmp[len(tmp)-10:]
      # 뒤에서부터 10개의 정보를 가져옴
      for i in range(len(tmp)):
        count.append(i)
        pm1.append(int(tmp[i]["pm1"]))
        pm2.append(int(tmp[i]["pm2"]))
        pm10.append(int(tmp[i]["pm10"]))
        tm = str(tmp[i]["created_at"])
        #created at 부분을 str형태로 tm 리스트에 추가
        time.append(tm[11:])
        # tm 11번째 길이부터 시간에 추가
      # 시간, pm1 , pm2, pm10를 리스트로 넣어줌
      # for d, cnt in zip(db['sensors'].find(), range(10, 0, -1)):
      #   count.append(cnt)
      #   pm2.append(int(d['pm2']))
      #   tm = str(d['created_at'])
      #   time.append(tm[11:])

    #   t = np.linspace(0, 2 * np.pi, 101)  # 0 ~ 2파이까지 101개의 숫자로 채우기
      self.dynamic_ax.plot(time, pm1, color="black") 
      # 점 찍기(x좌표, y좌표, 검정색)
      self.dynamic_ax.plot(time, pm2, color='green') 
      # 점 찍기(x좌표, y좌표, 초록색)
      self.dynamic_ax.plot(time, pm10, color="red")  
      # 점 찍기(x좌표, y좌표, 빨간색)
      self.dynamic_ax.figure.canvas.draw()  
      # 그리기

      if pm2[-1] <= 15:
        self.image.setPixmap(QPixmap("./verygood.png").scaled(50,50))
        status = '매우좋음'
        #pm2 의 값이 15 보다 작으면 매우좋음 이모티콘과 글을 띄움
      elif pm2[-1] <= 35:
        self.image.setPixmap(QPixmap("./good.png").scaled(50,50))
        status = '좋음'
        #pm2 의 값이 35 보다 작으면 좋음 이모티콘과 글을 띄움
      elif pm2[-1] <= 75:
        self.image.setPixmap(QPixmap("./bad.png").scaled(50,50))
        status = '나쁨'
        #pm2 의 값이 75 보다 작으면 나쁨 이모티콘과 글을 띄움
      else:
        self.image.setPixmap(QPixmap("./verybad.png").scaled(50,50))
        status = '매우나쁨'
        #pm2 의 값이 75 보다 크면 매우나쁨 이모티콘과 글을 띄움
      self.label1.setText(f'현재 PM1 농도는 {pm1[-1]}입니다.')
      # pm1의 현 상태를 보여줌 
      self.label3.setText(f'현재 PM2 농도는 {pm2[-1]}입니다.')
      # pm2의 현 상태를 보여줌
      self.label4.setText(f'현재 PM10 농도는 {pm10[-1]}입니다.')
      # pm10의 현 상태를 보여줌
      self.label2.setText(status)
      # status 내용을 라벨 2에 넣음
      self.Chbox.addWidget(self.image)
      # chbox에 이미지 추가 
      self.Chbox.addWidget(self.label2)
      # chbox에 이미지 추가

      self.label1.setAlignment(Qt.AlignCenter)
      #라벨 1을 중앙에 배치
      self.label3.setAlignment(Qt.AlignCenter)
      #라벨 3을 중앙에 배치
      self.label4.setAlignment(Qt.AlignCenter)
      #라벨 4를 중앙에 배치
      self.label2.setAlignment(Qt.AlignLeft)
      #라벨 2를 왼쪽에 배치
      self.image.setAlignment(Qt.AlignRight)
      #이미지를 오른쪽에 배치
      
      
if __name__ == '__main__':
  #py파일은 하나의 모듈형태로 만들어지기 때문에 누가 임포트하냐에 따라 __name__ 값이 달라진다.
  #자기가 직접 실행해야 실행된다. __name__(main) == __main__
  app = QApplication(sys.argv)
  # QApplication의 객체를 app으로 생성
  fontDB = QFontDatabase()
  # QFontDatabase의 객체를 fontDB로 생성
  fontDB.addApplicationFont('./NotoSansKR-Light.otf')
  # 같은 파일 내에 있는 폰트를 추가시킴(원래 말하신 폰트를 찾는데 실패했습니다..)
  app.setFont(QFont('Noto Sans CJK KR'))
  # 폰트를 적용시킴
  ex = MyApp()
  # 생성자의 self는 ex를 전달받게 된다
  sys.exit(app.exec_())
  # app객체를 실행시키고, system 의 x버튼을 누르면 실행되고 있는 App을 종료
```




