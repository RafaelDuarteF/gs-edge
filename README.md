# Edge - Tech OceanBlue

# Projeto de identificação de resíduos no oceano, por sensor ultrassônico e de inclinação

## Equipe

Rafael Duarte, RM 558644 <br>
Enzo Diógenes, RM 555062 <br>

## Link da simulação em TinkerCad
https://www.tinkercad.com/things/eaBaSlt0gzp-medidores-de-distancia-entre-carros-e-temperatura-da-pista

## Observações

O projeto conta com dois protótipos físicos: um desenvolvido no Wokwi, com ESP32 para ser possível a comunicação e envio dos dados obtidos pelo protótipo a uma base de dados real nossa; outra desenvolvida no TinkerCad com Arduino para a utilização do sensor de inclinação (não disponível no Wokwi). Dessa forma, serão apresentadas abaixo as informações nesses dois contextos de protótipos criados.

## Dependências
### Dependências TinkerCad:
1. Arduino Uno
2. Sensor de inclinação
3. Sensor de temperatura
4. 1x Resistor de 220Ω
5. Sensor ultrassônico
6. Tela LCD I2C
7. Conectores macho / fêmea e macho / macho

### Dependências Wokwi
1. ESP32
2. Sensor DHT22
3. Sensor ultrassônico
4. Tela LCD I2C
5. Conectores macho / fêmea e macho / macho

## Conceito do projeto / seu funcionamento

A ideia do projeto consistiu em desenvolver uma maneira de identificar e contabilizar resíduos encontrados boiando nos oceanos, além da temperatura das águas. A solução encontrada foi de criar um sistema baseado em ESP32, com sensores ultrassônicos, de inclinação e DHT22 integrados. O sensor ultrassônico seria o responsável por verificar se um resíduo está próximo (distância de até 10cm), e o sensor de inclinação verifica se nos próximos 5 segundos após a verificação de um resíduo próximo ocorrerá uma mudança brusca de inclinação, para evitar a contabilização de ondas como resíduo. Após ser identificado como resíduo, a informação é enviada até um servidor, na qual é informado a longitude, latitude e horários no qual o resíduo foi encontrado. Assim, é possível ter ciência da quantidade de resíduos identificados nos oceanos pelo sistema, além de suas localizações. Já o sensor DHT22 seria o responsável por medir a temperatura da água e verificar se está acima do normal, ou abaixo. Caso esteja, a informação de temperatura é enviada ao servidor, também com sua geolocalização e horário. Dessa forma, é possível analisar a temperatura média das águas nos oceanos medidas pelo nosso robô. Ambas as informações são exibidas de forma real em nosso website, vindas do banco de dados na qual armazenamos todas essas informações vindas do bot.

## Possibilidades de melhorias futuras

Visando a evolução do sistema, sugerimos algumas possibilidades de melhorias futuras: 
* Implementação de motores para circulação pelos oceanos;
* Implementação de painéis solares para energizar o sistema;
* Conexão de internet via satélite para o envio dos dados aos servidores.

### Reprodução visual:

* TinkerCad <br>
![image](https://github.com/RafaelDuarteF/gs-edge/assets/103393497/4deba006-fb4b-462f-9279-83654084b6d1)
![image](https://github.com/RafaelDuarteF/gs-edge/assets/103393497/50f5210f-c583-4928-a30a-b40ca1c70c79)

* Wokwi <br>
![image](https://github.com/RafaelDuarteF/gs-edge/assets/103393497/e4d4b2fe-756b-4da0-aab6-e10900ecd172)

### Código utilizado:

* TinkerCad:

```C++
#include <Adafruit_LiquidCrystal.h>

Adafruit_LiquidCrystal lcd(0);

#define LCD_ADDRESS 0x27
#define LCD_COLUMNS 16
#define LCD_ROWS 2

#define TRIG_PIN 7
#define ECHO_PIN 6
#define TILT_PIN 4
int tiltDegrees;
unsigned const long interval = 2000;
unsigned long zeroTrash = 0;
unsigned long zeroTemp = 0;
int RawValue = 0;
double Voltage = 0;
double tempC = 0;
const int analogIn = A0;

int tiltCounter = 0;

void setup() {
  Serial.begin(115200);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  lcd.begin(16, 2);
  lcd.begin(16, 2, LCD_5x8DOTS);
  lcd.setCursor(0, 0);
  pinMode (TILT_PIN, INPUT);
}

void loop(){
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);

  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);

  float distance = duration * 0.034 / 2;
  Serial.println(distance);

  RawValue = analogRead(analogIn);
  Voltage = (RawValue / 1023.0) * 5000; 
  tempC = (Voltage - 500) * 0.1; 
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Temp. Oceano:");
  lcd.setCursor(0, 1);
  lcd.print(tempC);
  lcd.print(" Celsius");

  if (distance < 10) {
    unsigned long startTime = millis();
    bool inclinacao = false;
    while (millis() - startTime < 5000) {
      int tiltDegrees = digitalRead(TILT_PIN); 
      if(!tiltDegrees) {
      	inclinacao = true;
        break;
      }
    }
    if (!inclinacao) {
      tiltCounter++;
    }
  }
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Residuos:");
  lcd.setCursor(0, 1);
  lcd.print(tiltCounter); 
  delay(3000);
}
```

* Wokwi:

```C++
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <Wire.h>  // Inclua a biblioteca Wire para comunicação I2C com o LCD
#include <LiquidCrystal_I2C.h>  // Inclua a biblioteca para controle do LCD I2C
#include "DHTesp.h"

#define LCD_ADDRESS 0x27
#define LCD_COLUMNS 16
#define LCD_ROWS 2

const int DHT_PIN = 15;

DHTesp dhtSensor;

const char* ssid = "Wokwi-GUEST";
const char* pass = "";
#define TRIG_PIN 4
#define ECHO_PIN 18
unsigned const long interval = 2000;
unsigned long zeroTrash = 0;
unsigned long zeroTemp = 0;

// Inicializa o objeto LiquidCrystal_I2C com o endereço do LCD, número de colunas e linhas
LiquidCrystal_I2C lcd(LCD_ADDRESS, LCD_COLUMNS, LCD_ROWS);

String latitude = ""; // Variável para armazenar a latitude
String longitude = ""; // Variável para armazenar a longitude

void setup() {
  Serial.begin(115200);
  WiFi.begin(ssid, pass);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
  
  while(WiFi.status() != WL_CONNECTED){
    delay(100);
    Serial.print(".");
  }
  
  Serial.println("\nWiFi Connected!");
  Serial.println(WiFi.localIP());
  lcd.init();
  // Ativa a luz de fundo do LCD
  lcd.backlight();
  
  // Define a mensagem inicial no LCD
  lcd.setCursor(0, 0);
  lcd.print("Distancia:");
  
  // Realiza a solicitação à API do OpenCage Geocoding e armazena a latitude e a longitude
  getGeolocation();
}

void loop(){
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);

  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);

  float distance = duration * 0.034 / 2;
  Serial.println(distance);
  
  // Exibe a distância no LCD
  lcd.setCursor(0, 1);
  lcd.print("                ");  // Limpa a linha anterior
  lcd.setCursor(0, 1);
  lcd.print(distance);  // Exibe a distância na segunda linha do LCD
  lcd.print(" cm");  // Exibe a distância na segunda linha do LCD
  
  if(distance <= 5) {
    if(millis() - zeroTrash > interval){

      HTTPClient http;
      http.begin("https://tech-ocean-blue-api.vercel.app/api/v1/trash"); // URL para o POST
      http.addHeader("Content-Type", "application/json"); // Adiciona cabeçalho para indicar o formato JSON

      // Corpo da solicitação em formato JSON
      String jsonBody = "{\"robotId\":\"5a40b01c-a9d0-4eeb-869b-9fa8a43533d6\",\"location\":{\"latitude\":";
      jsonBody += latitude;// Concatena a latitude com 6 casas decimais
      jsonBody += ",\"longitude\":";
      jsonBody += longitude;// Concatena a longitude com 6 casas decimais
      jsonBody += "}}";

      int httpResponseCode = http.POST(jsonBody); // Envia a solicitação POST com o corpo JSON
      Serial.println(httpResponseCode);

      if (httpResponseCode > 0) {
        String payload = http.getString();
        Serial.println(payload);
      } else {
        Serial.print("Erro: ");
        Serial.println(httpResponseCode);
      }

      zeroTrash = millis();
      http.end();

    }
    delay(3000);
  }
  
  TempAndHumidity data = dhtSensor.getTempAndHumidity();
  if(data.temperature > 28 || data.temperature < 10) {
    if(millis() - zeroTemp > interval) {

      HTTPClient http;
      http.begin("https://tech-ocean-blue-api.vercel.app/api/v1/temperature"); // URL para o POST
      http.addHeader("Content-Type", "application/json"); // Adiciona cabeçalho para indicar o formato JSON

      // Corpo da solicitação em formato JSON
      String jsonBody = "{\"robotId\":\"5a40b01c-a9d0-4eeb-869b-9fa8a43533d6\",\"location\":{\"latitude\":";
      jsonBody += latitude; // Concatena a latitude com 6 casas decimais
      jsonBody += ",\"longitude\":";
      jsonBody += longitude; // Concatena a longitude com 6 casas decimais
      jsonBody += "},\"temperatureCelsius\":";
      jsonBody += String(static_cast<int>(data.temperature)); // Concatena a temperatura com 1 casa decimal
      jsonBody += "}";

      int httpResponseCode = http.POST(jsonBody); // Envia a solicitação POST com o corpo JSON
      Serial.println(httpResponseCode);

      if (httpResponseCode > 0) {
        String payload = http.getString();
        Serial.println(payload);
      } else {
        Serial.print("Erro: ");
        Serial.println(httpResponseCode);
      }

      zeroTemp = millis();
      http.end();
    }
  }
}

void getGeolocation() {
  // Realiza a requisição GET à API do OpenCage Geocoding
  HTTPClient http;
  http.begin("https://api.opencagedata.com/geocode/v1/json?key=554aa02249524faf888a82cde3ac0335&q=" + WiFi.localIP().toString());
  int httpResponseCode = http.GET();

  if (httpResponseCode > 0) {
    // Lê a resposta da requisição HTTP
    String payload = http.getString();

    // Usa a biblioteca ArduinoJson para analisar o JSON retornado
    DynamicJsonDocument doc(1024);
    DeserializationError error = deserializeJson(doc, payload);
    
    if (!error) {
      // Extrai a latitude e a longitude da resposta JSON
      latitude = doc["results"][0]["geometry"]["lat"].as<String>();
      longitude = doc["results"][0]["geometry"]["lng"].as<String>();

      Serial.print("Latitude: ");
      Serial.println(latitude);
      Serial.print("Longitude: ");
      Serial.println(longitude);
    } else {
      Serial.println("Falha ao analisar JSON");
    }
  } else {
    Serial.print("Falha na requisição HTTP. Código de resposta: ");
    Serial.println(httpResponseCode);
  }
  
  // Encerra a conexão HTTP
  http.end();
}
```
