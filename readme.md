Status: многое нужно разобрать и оформить
Tags: [[it-школа]] 

**Вкратце:**
	используем протокол mqtt
# Теория
	Цели занятия:
	1. Научиться управлять esp32 при помощи сообщений протокола mqtt
	2. Научиться настраивать инфрастуктуру mqtt брокера EMQX

Разворачивание(deployment) сервера eqmx
1. переходим на главную страницу сервиса https://www.emqx.com/en
2. нажимаем login и авторизуемся удобным для нас способом(самый быстрый наверное через гугл аккаунт, при условии, что он у вас уже залогинен в браузере. Можно и просто по почте). Нужно ввести данные(можно вводить не настоящие(страну можно указать Россию), главное чтобы номер телефона выглядел правдоподобно и в первую очередь соответствовал выбранной стране) 
3. Затем нам предлагают сделать New Deploy. Это то что нам надо
4. Выбираем настройки нашего сервера.
Нас интересует serverless, так как он имеет достаточно широкие бесплатные лимиты и быстро поднимается.
![|528x381](mqtt%20занятие-1744275734559.png)
Единственное, что нас может напрячь - это чуть меньшее количество открытых веб портов для соединения
![](mqtt%20занятие-1744275840271.png)
Ниже, в качестве Deployment name лучше указать что-нибудь понятное, иначе сервис сам рандомно сгенерирует
![|393x177](mqtt%20занятие-1744276131827.png)
5. Нажимаем кнопку Deploy справа, соглашаемся со всеми условиями(самое интересное, как мне показалось, что сервер отключится, если к нему не будет подключений от клиентов в течении 30 дней). Через пару минут сервер будет полностью готов к работе, в этом время можно пока что ознакомится с интерфейсом сервиса и изучить необходимые нам данные

Какие данные нам важны для дальнейшей настройки?
Для дальнейшего подключения нам нужно:
1. Скачать CA сертификат, чтобы иметь возможность безопасно обмениваться данными
2. Обратить внимание на ссылку для сервера
3. В пункте меню - Acess Control - Authentication необходимо создать 2-ух пользователей нажатием на клавишу Add+: один будет наша esp32, вторым - наш компьютер. Тут особо не мудрите

Настройки MQTTX web
1. Переходим по ссылке https://mqttx.app/web и нажимаем get started
2. Слева в панели нажимаем + и указываем настройки подключения
![](mqtt%20занятие-1744280054136.png)
ALPN(серийный номер) берется из сертификата, который скачивается из раздела overview на EMQX Platform
![|404x301](mqtt%20занятие-1744280121956.png)

Если для просмотра данных сертификата требуются права администратора, то можно воспользоваться этим сайтом https://www.sslchecker.com/certdecoder
там нажать кнопку browse, выбрать наш сертификат и скопировать серийный номер
![|509x381](mqtt%20занятие-1744280192445.png)
 Далее необходимо добавить тему(subscription). В нашем случае ``emqx/esp32``

Настройка keepalive контроллера
Производится в ``PubSubClent.h`` соответствующим параметром ``MQTT_KEEPALIVE 60``
либо в коде в теле функции подключения к брокеру `mqtt_client.setKeepAlive(60);`


# Практика
### Собственный сервер
Код для подъема пробного соединения с собственным сервером
```cpp
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include "PubSubClient.h"

// Данные для подключения к Wi-Fi
const char* ssid = "POCOPHONE F1";
const char* password = "23456780";

// Данные MQTT-брокера
const char* mqtt_broker = "ie655665.ala.eu-central-1.emqxsl.com";
const char* mqtt_username = "esp32client";  // MQTT username
const char* mqtt_password = "esp32client";     // MQTT password
const char* mqtt_topic = "emqx/esp32";
int mqtt_port = 8883;


const char *ca_cert = R"EOF(
-----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIQCDvgVpBCRrGhdWrJWZHHSjANBgkqhkiG9w0BAQUFADBh
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3
d3cuZGlnaWNlcnQuY29tMSAwHgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBD
QTAeFw0wNjExMTAwMDAwMDBaFw0zMTExMTAwMDAwMDBaMGExCzAJBgNVBAYTAlVT
MRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxGTAXBgNVBAsTEHd3dy5kaWdpY2VydC5j
b20xIDAeBgNVBAMTF0RpZ2lDZXJ0IEdsb2JhbCBSb290IENBMIIBIjANBgkqhkiG
9w0BAQEFAAOCAQ8AMIIBCgKCAQEA4jvhEXLeqKTTo1eqUKKPC3eQyaKl7hLOllsB
CSDMAZOnTjC3U/dDxGkAV53ijSLdhwZAAIEJzs4bg7/fzTtxRuLWZscFs3YnFo97
nh6Vfe63SKMI2tavegw5BmV/Sl0fvBf4q77uKNd0f3p4mVmFaG5cIzJLv07A6Fpt
43C/dxC//AH2hdmoRBBYMql1GNXRor5H4idq9Joz+EkIYIvUX7Q6hL+hqkpMfT7P
T19sdl6gSzeRntwi5m3OFBqOasv+zbMUZBfHWymeMr/y7vrTC0LUq7dBMtoM1O/4
gdW7jVg/tRvoSSiicNoxBN33shbyTApOB6jtSj1etX+jkMOvJwIDAQABo2MwYTAO
BgNVHQ8BAf8EBAMCAYYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUA95QNVbR
TLtm8KPiGxvDl7I90VUwHwYDVR0jBBgwFoAUA95QNVbRTLtm8KPiGxvDl7I90VUw
DQYJKoZIhvcNAQEFBQADggEBAMucN6pIExIK+t1EnE9SsPTfrgT1eXkIoyQY/Esr
hMAtudXH/vTBH1jLuG2cenTnmCmrEbXjcKChzUyImZOMkXDiqw8cvpOp/2PV5Adg
06O/nVsJ8dWO41P0jmP6P6fbtGbfYmbW0W5BjfIttep3Sp+dWOIrWcBAI+0tKIJF
PnlUkiaY4IBIqDfv8NZ5YBberOgOzW6sRBc4L0na4UU+Krk2U886UAb3LujEV0ls
YSEY1QSteDwsOoBrp+uvFRTp2InBuThs4pFsiv9kuXclVzDAGySj4dzp30d8tbQk
CAUw7C29C79Fv1C5qfPrmAESrciIxpg0X40KPMbp1ZWVbd4=
-----END CERTIFICATE-----
)EOF";
// Создаем WiFi и MQTT объекты
WiFiClientSecure espClient;
PubSubClient mqtt_client(espClient);


void connectToWiFi() {
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConnected to the WiFi network");
}

void connectToMQTTBroker() {
  espClient.setCACert(ca_cert);  // Set the CA certificate for secure connection
  mqtt_client.setKeepAlive(60);  // Set the keep-alive interval to 60 seconds
  // Если клиент не подключен, пытаемся подключиться
  while (!mqtt_client.connected()) {
    // Создаем уникальный client_id, используя MAC-адрес
    String client_id = "esp32-client-" + String(WiFi.macAddress());
    Serial.printf("Connecting to MQTT Broker as %s...\n", client_id.c_str());
    if (mqtt_client.connect(client_id.c_str(), mqtt_username, mqtt_password)) {
      Serial.println("Connected to MQTT broker");
      // Подписываемся на нужный топик
      mqtt_client.subscribe(mqtt_topic);
      // При успешном подключении публикуем стартовое сообщение
      mqtt_client.publish(mqtt_topic, "ESP32 connected and ready");
    } else {
      Serial.print("Failed to connect, state=");
      Serial.print(mqtt_client.state());
      Serial.println(" - trying again in 5 seconds");
      delay(5000);
    }
  }
}

// Функция обратного вызова, которая вызывается при получении сообщения с MQTT-брокера
void mqttCallback(char *topic, byte *payload, unsigned int length) {
  Serial.print("Message received on topic: ");
  Serial.println(topic);
  
  // Преобразуем payload в строку
  String message = "";
  for (unsigned int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  Serial.print("Message: ");
  Serial.println(message);
  Serial.println("-----------------------");
}

void loop() {
    if (!mqtt_client.connected()) {
        connectToMQTTBroker();
    }
    mqtt_client.loop();
}
```

### Код управления светодиодом при помощи сообщений on/off
так как у нас ограниченное количество портов для подключения, то использовать мы можем только ssl порты. Это вынуждает нас подключать esp32 при помощи библиотеки `WifiClientSecure` и указывать CA сертификат в коде. 
Также мы дополнительно указываем `keepalive`параметр, чтобы esp32 не отключалась каждые 10 секунд
```cpp
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include "PubSubClient.h"

// Данные для подключения к Wi-Fi
const char* ssid = "POCOPHONE F1";
const char* password = "23456780";

// Данные MQTT-брокера
const char* mqtt_broker = "ie655665.ala.eu-central-1.emqxsl.com";
const char* mqtt_username = "esp32client";  // MQTT username
const char* mqtt_password = "esp32client";     // MQTT password
const char* mqtt_topic = "emqx/esp32";
int mqtt_port = 8883;

// Пин для управления светодиодом (например, встроенный светодиод на ESP32)
// Обычно на ESP32 встроенный светодиод может быть на GPIO2, но уточните для своей платы.
const int LED_PIN = 2;

const char *ca_cert = R"EOF(
-----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIQCDvgVpBCRrGhdWrJWZHHSjANBgkqhkiG9w0BAQUFADBh
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3
d3cuZGlnaWNlcnQuY29tMSAwHgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBD
QTAeFw0wNjExMTAwMDAwMDBaFw0zMTExMTAwMDAwMDBaMGExCzAJBgNVBAYTAlVT
MRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxGTAXBgNVBAsTEHd3dy5kaWdpY2VydC5j
b20xIDAeBgNVBAMTF0RpZ2lDZXJ0IEdsb2JhbCBSb290IENBMIIBIjANBgkqhkiG
9w0BAQEFAAOCAQ8AMIIBCgKCAQEA4jvhEXLeqKTTo1eqUKKPC3eQyaKl7hLOllsB
CSDMAZOnTjC3U/dDxGkAV53ijSLdhwZAAIEJzs4bg7/fzTtxRuLWZscFs3YnFo97
nh6Vfe63SKMI2tavegw5BmV/Sl0fvBf4q77uKNd0f3p4mVmFaG5cIzJLv07A6Fpt
43C/dxC//AH2hdmoRBBYMql1GNXRor5H4idq9Joz+EkIYIvUX7Q6hL+hqkpMfT7P
T19sdl6gSzeRntwi5m3OFBqOasv+zbMUZBfHWymeMr/y7vrTC0LUq7dBMtoM1O/4
gdW7jVg/tRvoSSiicNoxBN33shbyTApOB6jtSj1etX+jkMOvJwIDAQABo2MwYTAO
BgNVHQ8BAf8EBAMCAYYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUA95QNVbR
TLtm8KPiGxvDl7I90VUwHwYDVR0jBBgwFoAUA95QNVbRTLtm8KPiGxvDl7I90VUw
DQYJKoZIhvcNAQEFBQADggEBAMucN6pIExIK+t1EnE9SsPTfrgT1eXkIoyQY/Esr
hMAtudXH/vTBH1jLuG2cenTnmCmrEbXjcKChzUyImZOMkXDiqw8cvpOp/2PV5Adg
06O/nVsJ8dWO41P0jmP6P6fbtGbfYmbW0W5BjfIttep3Sp+dWOIrWcBAI+0tKIJF
PnlUkiaY4IBIqDfv8NZ5YBberOgOzW6sRBc4L0na4UU+Krk2U886UAb3LujEV0ls
YSEY1QSteDwsOoBrp+uvFRTp2InBuThs4pFsiv9kuXclVzDAGySj4dzp30d8tbQk
CAUw7C29C79Fv1C5qfPrmAESrciIxpg0X40KPMbp1ZWVbd4=
-----END CERTIFICATE-----
)EOF";
// Создаем WiFi и MQTT объекты
WiFiClientSecure espClient;
PubSubClient mqtt_client(espClient);


void connectToWiFi() {
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConnected to the WiFi network");
}

void connectToMQTTBroker() {
  espClient.setCACert(ca_cert);  // Set the CA certificate for secure connection
  mqtt_client.setKeepAlive(60);  // Set the keep-alive interval to 60 seconds
  // Если клиент не подключен, пытаемся подключиться
  while (!mqtt_client.connected()) {
    // Создаем уникальный client_id, используя MAC-адрес
    String client_id = "esp32-client-" + String(WiFi.macAddress());
    Serial.printf("Connecting to MQTT Broker as %s...\n", client_id.c_str());
    if (mqtt_client.connect(client_id.c_str(), mqtt_username, mqtt_password)) {
      Serial.println("Connected to MQTT broker");
      // Подписываемся на нужный топик
      mqtt_client.subscribe(mqtt_topic);
      // При успешном подключении публикуем стартовое сообщение
      mqtt_client.publish(mqtt_topic, "ESP32 connected and ready");
    } else {
      Serial.print("Failed to connect, state=");
      Serial.print(mqtt_client.state());
      Serial.println(" - trying again in 5 seconds");
      delay(5000);
    }
  }
}

// Функция обратного вызова, которая вызывается при получении сообщения с MQTT-брокера
void mqttCallback(char *topic, byte *payload, unsigned int length) {
  Serial.print("Message received on topic: ");
  Serial.println(topic);
  
  // Преобразуем payload в строку
  String message = "";
  for (unsigned int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  Serial.print("Message: ");
  Serial.println(message);
  
  // Управляем светодиодом в зависимости от сообщения
  if (message.equalsIgnoreCase("ON")) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("LED turned ON");
  }
  else if (message.equalsIgnoreCase("OFF")) {
    digitalWrite(LED_PIN, LOW);
    Serial.println("LED turned OFF");
  }
  
  Serial.println("-----------------------");
}

void setup() {
  Serial.begin(115200);
  // Настройка пина для светодиода как выход
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // По умолчанию светодиод выключен
  
  // Подключаемся к Wi-Fi
  connectToWiFi();
  
  // Настраиваем MQTT клиент
  mqtt_client.setServer(mqtt_broker, mqtt_port);
  mqtt_client.setCallback(mqttCallback);
  
  // Подключаемся к MQTT брокеру
  connectToMQTTBroker();
}

void loop() {
  // Если MQTT клиент отключен, пытаемся переподключиться
  if (!mqtt_client.connected()) {
    connectToMQTTBroker();
  }
  mqtt_client.loop();
}
```

### Код вывода значений датчика влажности + управление светодиодом

```cpp

```
# Источники инфы
фундаментальная статья по устройству MQTT как я понимаю
https://randomnerdtutorials.com/what-is-mqtt-and-how-it-works/
на русском, но без картинок
https://voltiq.ru/how-to-use-mqtt-for-smart-home-arduino-esp32/
Библиотека
https://github.com/knolleary/pubsubclient/blob/master/README.md
Код тестовых сообщений
https://docs.emqx.com/en/cloud/latest/connect_to_deployments/esp8266.html#test-connection
Начало работы с esp32 с публичным сервером, но не совсем понятно было а как подключится к этому публичному серверу - подробно описывается структура кода
https://www.emqx.com/en/blog/esp32-connects-to-the-free-public-mqtt-broker

Настройка клиента через MQTTX web для возможности отправки сообщений
https://mqttx.app/docs/get-started

Крутые статьи по программированию mqttx
https://www.emqx.com/en/blog/category/mqtt-programming

