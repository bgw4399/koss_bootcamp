# 3주차 과제 영상

https://user-images.githubusercontent.com/104902657/184536492-9d15a2a9-8633-46ee-b755-a483d628053f.mp4

# 3주차 과제 app.js
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

# 4주차 과제 영상


https://user-images.githubusercontent.com/104902657/184536718-860e9e71-9846-4d79-8203-e8484c52fad6.mp4




