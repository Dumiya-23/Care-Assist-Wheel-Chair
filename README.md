#include <Arduino.h>
#include <WiFi.h>
#include <AsyncTCP.h>
#include <ESPAsyncWebServer.h>

#include <DHT11.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <DFRobot_MAX30102.h>

#include <iostream>
#include <sstream>

#define UP 1         
#define DOWN 2
#define LEFT 3
#define RIGHT 4
#define SREAD 5
#define STOP 0

#define IN1_PIN 32  
#define IN2_PIN 33  
#define IN3_PIN 25  
#define IN4_PIN 26 

const char* ssid     = "ABCDEF";
const char* password = "abcdefghi";

#define ONE_WIRE_BUS 5

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);
DHT11 dht11(18);
DFRobot_MAX30102 particleSensor;

AsyncWebServer server(80);
AsyncWebSocket wsCarInput("/CarInput");

float BT;
float RT;
int RH;

const char* htmlHomePage PROGMEM = R"HTMLHOMEPAGE(
<!DOCTYPE html>
<html>
  <head>
  <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
    <style>
    .arrows {
      font-size:40px;
      color:red;
    }
    td.button {
      background-color:black;
      border-radius:25%;
      box-shadow: 5px 5px #888888;
    }
    td.button:active {
      transform: translate(5px,5px);
      box-shadow: none; 
    }

    .noselect {
      -webkit-touch-callout: none; /* iOS Safari */
        -webkit-user-select: none; /* Safari */
         -khtml-user-select: none; /* Konqueror HTML */
           -moz-user-select: none; /* Firefox */
            -ms-user-select: none; /* Internet Explorer/Edge */
                user-select: none; /* Non-prefixed version, currently
                                      supported by Chrome and Opera */
    }

    .slidecontainer {
      width: 100%;
    }

    .slider {
      -webkit-appearance: none;
      width: 100%;
      height: 20px;
      border-radius: 5px;
      background: #d3d3d3;
      outline: none;
      opacity: 0.7;
      -webkit-transition: .2s;
      transition: opacity .2s;
    }

    .slider:hover {
      opacity: 1;
    }
  
    .slider::-webkit-slider-thumb {
      -webkit-appearance: none;
      appearance: none;
      width: 40px;
      height: 40px;
      border-radius: 50%;
      background: red;
      cursor: pointer;
    }

    .slider::-moz-range-thumb {
      width: 40px;
      height: 40px;
      border-radius: 50%;
      background: red;
      cursor: pointer;
    }

    </style>
  
  </head>
  <body class="noselect" align="center" style="background-color:white">
     
    <h1 style="color: teal;text-align:center;">Wheel Chair Project</h1>
    
    <table id="mainTable" style="width:350px;margin:auto;table-layout:fixed" CELLSPACING=10>
      <tr>
        <td></td>
        <td class="button" ontouchstart='sendButtonInput("MoveCar","1")' ontouchend='sendButtonInput("MoveCar","0")'><span class="arrows" >&#8679;</span></td>
        <td></td>
      </tr>
      <tr>
        <td class="button" ontouchstart='sendButtonInput("MoveCar","3")' ontouchend='sendButtonInput("MoveCar","0")'><span class="arrows" >&#8678;</span></td>
        <td></td>    
        <td class="button" ontouchstart='sendButtonInput("MoveCar","4")' ontouchend='sendButtonInput("MoveCar","0")'><span class="arrows" >&#8680;</span></td>
      </tr>
      <tr>
        <td></td>
        <td class="button" ontouchstart='sendButtonInput("MoveCar","2")' ontouchend='sendButtonInput("MoveCar","0")'><span class="arrows" >&#8681;</span></td>
        <td></td>
      </tr>       
    </table>
      <h4>Room Temperature: <span id="roomTemperature"></span> °C</h4>
      <h4>Room Humidity: <span id="roomHumidity"></span>%</h4>
      <h4>Body Temperature: <span id="bodyTemperature"></span> °C</h4>
      <h4>Body Heart Rate: <span id="heartRate"></span> BPM</h4>
      <h4>Body SPO2 Level: <span id="spo2Level"></span> %</h4>
  
    <script>
      var webSocketCarInputUrl = "ws:\/\/" + window.location.hostname + "/CarInput";      
      var websocketCarInput;
      
      function initCarInputWebSocket() 
      {
        websocketCarInput = new WebSocket(webSocketCarInputUrl);
        websocketCarInput.onopen    = function(event){};
        websocketCarInput.onclose   = function(event){setTimeout(initCarInputWebSocket, 2000);};
        websocketCarInput.onmessage = function(event){
          var data = JSON.parse(event.data);
        // Update room temperature
        document.getElementById("roomTemperature").innerText = data.temp;
        // Update room humidity
        document.getElementById("roomHumidity").innerText = data.humidity;
        // Update body temperature
        document.getElementById("bodyTemperature").innerText = data.bodyTemp;
        // Update heart rate
        document.getElementById("heartRate").innerText = data.heartRate;
        // Update SpO2 level
        document.getElementById("spo2Level").innerText = data.spo2;
        };        
      }
      
      function sendButtonInput(key, value) 
      {
        var data = key + "," + value;
        websocketCarInput.send(data);
      }

      window.onload = initCarInputWebSocket;
      document.getElementById("mainTable").addEventListener("touchend", function(event){
        event.preventDefault()
      });      
    </script>
  </body>    
</html>
)HTMLHOMEPAGE";

void setup(void) 
{
  Serial.begin(115200);

  WiFi.softAP(ssid, password);
  IPAddress IP = WiFi.softAPIP();
  Serial.print("AP IP address: ");
  Serial.println(IP);

  server.on("/", HTTP_GET, handleRoot);
  server.onNotFound(handleNotFound);
      
  wsCarInput.onEvent(onCarInputWebSocketEvent);
  server.addHandler(&wsCarInput);

  server.begin();
  Serial.println("HTTP server started");

  sensors.begin();
  particleSensor.begin();
  particleSensor.sensorConfiguration(50, SAMPLEAVG_4, MODE_MULTILED, SAMPLERATE_100, PULSEWIDTH_411, ADCRANGE_16384);

  pinMode(IN1_PIN, OUTPUT);
  pinMode(IN2_PIN, OUTPUT);
  pinMode(IN3_PIN, OUTPUT);
  pinMode(IN4_PIN, OUTPUT);

}

void SensorReadings()
{
  int humidity = dht11.readHumidity();
  delay(1000);
  float temperature = dht11.readTemperature();
  delay(100);

  // Body Temperature
  sensors.requestTemperatures(); // Send the command to get temperatures
  float tempC = sensors.getTempCByIndex(0);

  // Print Humidity Sensor
  Serial.print("Room Temperature: ");
  RT = temperature;
  Serial.print(temperature);
  Serial.println(" °C ");

  Serial.print("Room Humidity: ");
  RH = humidity;
  Serial.print(humidity);
  Serial.println("%");

  // Print Body Temperature
  // Check if reading was successful
  if (tempC != DEVICE_DISCONNECTED_C) {
    BT = tempC;
    Serial.print("Body Temperature: ");
    Serial.println(tempC);
  } else {
    Serial.println("Error: Could not read temperature data");
  }

  // Print Pulse Oximeter
  int32_t SPO2;
  int8_t SPO2Valid;
  int32_t heartRate;
  int8_t heartRateValid;
  particleSensor.heartrateAndOxygenSaturation(&SPO2, &SPO2Valid, &heartRate, &heartRateValid);

  // Print results
  Serial.print(F("Heart Rate = "));
  Serial.print(heartRate, DEC);
  Serial.println(F(" BPM"));
  Serial.print(F("SPO2 = "));
  Serial.print(SPO2, DEC);
  Serial.println(F("%"));

  // WebSocket data transmission (modify as needed)
  String sensorData = "{\"temp\":\"" + String(RT) + "\", \"bodyTemp\":\"" + String(BT) + "\", \"humidity\":\"" + String(RH) + "\", \"heartRate\":\"" + String(heartRate) + "\", \"spo2\":\"" + String(SPO2) + "\"}";
  wsCarInput.textAll(sensorData);
  delay(6000);

}

void handleRoot(AsyncWebServerRequest *request) 
{
  request->send_P(200, "text/html", htmlHomePage);
}

void handleNotFound(AsyncWebServerRequest *request) 
{
    request->send(404, "text/plain", "File Not Found");
}

void onCarInputWebSocketEvent(AsyncWebSocket *server, 
                      AsyncWebSocketClient *client, 
                      AwsEventType type,
                      void *arg, 
                      uint8_t *data, 
                      size_t len) 
{                      
  switch (type) 
  {
    case WS_EVT_CONNECT:
      Serial.printf("WebSocket client #%u connected from %s\n", client->id(), client->remoteIP().toString().c_str());
      break;
    case WS_EVT_DISCONNECT:
      Serial.printf("WebSocket client #%u disconnected\n", client->id());
      Serial.printf("Car Stop");
      break;
    case WS_EVT_DATA:
      AwsFrameInfo *info;
      info = (AwsFrameInfo*)arg;
      if (info->final && info->index == 0 && info->len == len && info->opcode == WS_TEXT) 
      {
        std::string myData = "";
        myData.assign((char *)data, len);
        std::istringstream ss(myData);
        std::string key, value;
        std::getline(ss, key, ',');
        std::getline(ss, value, ',');
        Serial.printf("Key [%s] Value[%s]\n", key.c_str(), value.c_str()); 
        int valueInt = atoi(value.c_str());     
        if (key == "MoveCar")
        {
          moveCar(valueInt);        
        }
      }
      break;
    case WS_EVT_PONG:
    case WS_EVT_ERROR:
      break;
    default:
      break;  
  }
}

void moveCar(int inputValue)
{
  Serial.printf("Got value as %d\n", inputValue);  
  switch(inputValue)
  {

    case UP:
      Serial.println("Forward");
      CAR_moveForward();                  
      break;
  
    case DOWN:
      Serial.println("Backward");  
      CAR_moveBackward();
      break;
  
    case LEFT:
      Serial.println("Turn Left");  
      CAR_turnLeft();
      break;
  
    case RIGHT:
      Serial.println("Turn Right");
      CAR_turnRight();
      break;

    case STOP:
      Serial.println("STOP"); 
      CAR_stop();  
      break;

    case SREAD:
      Serial.println("Sensor Read");
      SensorReadings();
      break;
  
    default:
      Serial.println("STOP");
      CAR_stop();
      break;
  }
}

void loop() 
{
  wsCarInput.cleanupClients(); 
  SensorReadings();
}

void CAR_moveForward() {
  digitalWrite(IN1_PIN, HIGH);
  digitalWrite(IN2_PIN, LOW);
  digitalWrite(IN3_PIN, HIGH);
  digitalWrite(IN4_PIN, LOW);
}

void CAR_moveBackward() {
  digitalWrite(IN1_PIN, LOW);
  digitalWrite(IN2_PIN, HIGH);
  digitalWrite(IN3_PIN, LOW);
  digitalWrite(IN4_PIN, HIGH);
}

void CAR_turnLeft() {
  digitalWrite(IN1_PIN, HIGH);
  digitalWrite(IN2_PIN, LOW);
  digitalWrite(IN3_PIN, LOW);
  digitalWrite(IN4_PIN, LOW);
}

void CAR_turnRight() {
  digitalWrite(IN1_PIN, LOW);
  digitalWrite(IN2_PIN, LOW);
  digitalWrite(IN3_PIN, HIGH);
  digitalWrite(IN4_PIN, LOW);
}

void CAR_stop() {
  digitalWrite(IN1_PIN, HIGH);
  digitalWrite(IN2_PIN, HIGH);
  digitalWrite(IN3_PIN, HIGH);
  digitalWrite(IN4_PIN, HIGH);
}
