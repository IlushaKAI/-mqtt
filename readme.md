Status: нужно сделать руководство по выводу значений датчиков и настройке мобильного приложения
Tags: [[it-школа]] [[esp32]]

**Вкратце:**
используем протокол mqtt и брокера emqx

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
   ![|528x381](./images/mqtt%20занятие-1744275734559.png)
   Единственное, что нас может напрячь - это чуть меньшее количество открытых веб портов для соединения
   ![](./images/mqtt%20занятие-1744275840271.png)
   Ниже, в качестве Deployment name лучше указать что-нибудь понятное, иначе сервис сам рандомно сгенерирует
   ![|393x177](./images/mqtt%20занятие-1744276131827.png)
5. Нажимаем кнопку Deploy справа, соглашаемся со всеми условиями(самое интересное, как мне показалось, что сервер отключится, если к нему не будет подключений от клиентов в течении 30 дней). Через пару минут сервер будет полностью готов к работе, в этом время можно пока что ознакомится с интерфейсом сервиса и изучить необходимые нам данные

Какие данные нам важны для дальнейшей настройки?
Для дальнейшего подключения нам нужно:

1. Скачать CA сертификат, чтобы иметь возможность безопасно обмениваться данными
2. Обратить внимание на ссылку для сервера
3. В пункте меню - Acess Control - Authentication необходимо создать 2-ух пользователей нажатием на клавишу Add+: один будет наша esp32, вторым - наш компьютер. Тут особо не мудрите

Настройки MQTTX web

1. Переходим по ссылке https://mqttx.app/web и нажимаем get started
2. Слева в панели нажимаем + и указываем настройки подключения
   ![](./images/mqtt%20занятие-1744280054136.png)
   ALPN(серийный номер) берется из сертификата, который скачивается из раздела overview на EMQX Platform
   ![|404x301](./images/mqtt%20занятие-1744280121956.png)

Если для просмотра данных сертификата требуются права администратора, то можно воспользоваться этим сайтом https://www.sslchecker.com/certdecoder
там нажать кнопку browse, выбрать наш сертификат и скопировать серийный номер
![|509x381](./images/mqtt%20занятие-1744280192445.png)
Далее необходимо добавить тему(subscription). В нашем случае `emqx/esp32`

Настройка keepalive контроллера
Производится в `PubSubClent.h` соответствующим параметром `MQTT_KEEPALIVE 60`
либо в коде в теле функции подключения к брокеру `mqtt_client.setKeepAlive(60);`

# Практика

### Собственный сервер

Код для подъема пробного соединения с собственным сервером

```cpp
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include "PubSubClient.h"

// Данные для подключения к Wi-Fi
const char* ssid = "ssid";
const char* password = "password";

// Данные MQTT-брокера
const char* mqtt_broker = "xxx.emqxsl.com"; // доменое имя либо ip адрес
const char* mqtt_username = "esp32client";  // MQTT username
const char* mqtt_password = "esp32client";     // MQTT password
const char* mqtt_topic = "emqx/esp32";
int mqtt_port = 8883;


const char *ca_cert = R"EOF(
-----BEGIN CERTIFICATE-----
// замените данными вашего сертификата
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
const char* ssid = "ssid";
const char* password = "password";

// Данные MQTT-брокера
const char* mqtt_broker = "xxx.emqxsl.com"; // доменое имя либо ip адрес
const char* mqtt_username = "esp32client";  // MQTT username
const char* mqtt_password = "esp32client";     // MQTT password
const char* mqtt_topic = "emqx/esp32";
int mqtt_port = 8883;

// Пин для управления светодиодом (например, встроенный светодиод на ESP32)
// Обычно на ESP32 встроенный светодиод может быть на GPIO2, но уточните для своей платы.
const int LED_PIN = 2;

const char *ca_cert = R"EOF(
-----BEGIN CERTIFICATE-----
///замените данными вашего сертификата
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

Код уже имеет асинхронный функционал и подключается к моему серверу, на котором я развернул emqx open source

Username для esp32 имеет формат: `ИмяEsp32`

Например, `IlyaEsp32`

Пароль такой же как и username

```cpp
#include <WiFi.h>
#include "PubSubClient.h"
//добавляем библиотеку для работы с dht11
#include <DHT.h>

// Данные для подключения к Wi-Fi
const char* ssid = "ssid";
const char* password = "password";

// Данные MQTT-брокера
const char* mqtt_broker = "0.0.0.0"; // изменить на свой ip адрес сервера
const char* mqtt_username = "username";  // MQTT username
const char* mqtt_password = "password";     // MQTT password
const char* mqtt_topic = "esp32/led";
const char* mqtt_topic_temp = "esp32/temp";
const char* mqtt_topic_hum = "esp32/hum";
int mqtt_port = 1883;

// Обычно на ESP32 встроенный светодиод на GPIO2
const int LED_PIN = 2;

// Пин, к которому подключен датчик DHT11
#define DHTPIN 33      // GPIO 33 для данных DHT11
#define DHTTYPE DHT11  // Тип датчика DHT11

// Создаем объект DHT
DHT dht(DHTPIN, DHTTYPE);
const int sensorPIN = 36;

// Создаем WiFi и MQTT объекты
WiFiClient espClient;
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
  mqtt_client.setKeepAlive(10000);  // Set the keep-alive interval to 10 seconds

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
      mqtt_client.publish(mqtt_topic_temp, "ESP32 connected and ready");
      mqtt_client.publish(mqtt_topic_hum, "ESP32 connected and ready");
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

void publishSensorData() {
  // Чтение с датчиков
  float humidity = dht.readHumidity();
  float temperature = dht.readTemperature();

  // Проверка на ошибку чтения
  if (isnan(humidity) || isnan(temperature)) {
    Serial.println("Failed to read from DHT sensor!");
    return;
  }

  // Выводим в консоль для отладки
  Serial.print("Температура: ");
  Serial.print(temperature);
  Serial.print(" °C | Влажность: ");
  Serial.print(humidity);
  Serial.println(" %");

  // Публикуем данные в MQTT
  mqtt_client.publish(mqtt_topic_temp, String(temperature).c_str(), true);
  mqtt_client.publish(mqtt_topic_hum, String(humidity).c_str(), true);
}

void setup() {
  Serial.begin(115200);

  // Настройка пина для светодиода как выход
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // По умолчанию светодиод выключен

  // Добавили инициализацию датчика DHT11
  dht.begin();

  // Подключаемся к Wi-Fi
  connectToWiFi();

  // Настраиваем MQTT клиент
  mqtt_client.setServer(mqtt_broker, mqtt_port);
  mqtt_client.setCallback(mqttCallback);
}

// Переменные для неблокирующей работы с таймингами
unsigned long lastSensorPublishTime = 0;
const unsigned long SENSOR_PUBLISH_INTERVAL = 3000; // 3 секунды

void loop() {
  // Если MQTT клиент отключен, пытаемся переподключиться
  if (!mqtt_client.connected()) {
    connectToMQTTBroker();
  }

  // Обрабатываем входящие MQTT сообщения
  mqtt_client.loop();

  // Публикуем данные датчиков с заданным интервалом
  unsigned long currentMillis = millis();
  if (currentMillis - lastSensorPublishTime >= SENSOR_PUBLISH_INTERVAL) {
    lastSensorPublishTime = currentMillis;
    publishSensorData();
  }
}
```

## Настройка мобильного приложения

Буду объяснять настройку на примере этого приложения
https://play.google.com/store/apps/details?id=snr.lab.iotmqttpanel.prod

### Создание подключения

На вкладке connections создаем новое подключение и указываем необходимые параметры
![|242x531](./images/занятие%20mqtt-1745482785087.png)

Поля где стоят красные звездочки обязательны к заполнению
Давайте пройдемся по полям формы для большей ясности:

- Connection name - имя соединения, которое будет отображаться у брокера. Можно указывать любое
- Client id - на всякий случай стоит оставить изначальное рандомно сгенерированное
- Broker Web/IP address - вполне понятно, что нужно либо указать имя сайта, либо его ip адрес
- Port - указываем именно 8083, так как у меня на сервере он отвечает за Websocket соединение
- Network Protocol - как следует из порта, необходимо выбрать Websocket.
- Add Dashboard - добавляем панель управления всеми устройствами. Как в arduino iot cloud. Пока просто добавляем, к настройке передём позже
- Additional options - Дополнительные настройки соединения по смыслу аналогичны параметрам, которые мы указываем для esp32
  Username для ваших телефонов имеет формат: `ИмяPhone

Например, `IlyaPhone`

Пароль такой же как и username

### Настройка dashboard

После создания соединения необходимо на него тапнуть и нас перекинет на dashboard. Он пока что разумеется пустой

Добавим тип панели switch и рассмотрим основные поля
![|196x433](./images/занятие%20mqtt-1745483899151.png)

- Topic - тема, к которой эта панель подключается. Она должна соответствовать topic нужного устройства на esp32
- Payload скажу обобщенно - это само сообщение в рамках темы. В данном случае наш переключатель будет отправлять ON либо OFF сообщение
- Также рекомендую поставить QoS в второй режим, так как таким образом оно точно дойдет до получателя
  Все остальные панели настраиваются аналогично, там не так сложно разобраться

# Источники инфы

фундаментальная статья по устройству MQTT, как я понимаю
https://randomnerdtutorials.com/what-is-mqtt-and-how-it-works/

на русском, но без картинок
https://voltiq.ru/how-to-use-mqtt-for-smart-home-arduino-esp32/

Код тестовых сообщений
https://docs.emqx.com/en/cloud/latest/connect_to_deployments/esp8266.html#test-connection

Начало работы с esp32 с публичным сервером, но не совсем понятно было а как подключится к этому публичному серверу - подробно описывается структура кода
https://www.emqx.com/en/blog/esp32-connects-to-the-free-public-mqtt-broker

Настройка клиента через MQTTX web для возможности отправки сообщений
https://mqttx.app/docs/get-started

Крутые статьи по программированию mqttx
https://www.emqx.com/en/blog/category/mqtt-programming

Библиотека
https://github.com/knolleary/pubsubclient/blob/master/README.md
