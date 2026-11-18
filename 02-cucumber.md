# 🔌 บทที่ 2: บอร์ด Cucumber และเซนเซอร์วัดสภาพอากาศ

ในบทนี้เราจะมาทำความรู้จักกับบอร์ด Cucumber (ESP32-S2-Saola-1) ซึ่งเป็นอุปกรณ์หลักที่ใช้ในส่วนของ IoT พร้อมเรียนรู้วิธีการเชื่อมต่ออุปกรณ์และทดลองเขียนโปรแกรมเพื่ออ่านค่าอุณหภูมิและความชื้นจากเซนเซอร์ จากนั้นจะส่งข้อมูลไปยังระบบผ่านโปรโตคอล MQTT เพื่อนำไปใช้งานในขั้นตอนต่อไป

---

## 🛠️ ทำความรู้จักกับบอร์ด Cucumber (ESP32-S2-Saola-1)

ก่อนเริ่มเขียนโปรแกรม เรามาดูส่วนประกอบต่าง ๆ บนบอร์ด Cucumber กันก่อน เพื่อให้รู้ว่าแต่ละส่วนใช้ทำอะไรและต้องเชื่อมต่ออย่างไร
<img width="1532" height="930" alt="image" src="https://github.com/user-attachments/assets/1a8a5c58-6696-48ed-a8f3-1e454fa71d77" />
*(ภาพแสดงตำแหน่งอุปกรณ์และพอร์ตต่าง ๆ บนบอร์ด)*

ส่วนประกอบที่ควรรู้จักมีดังนี้

- **เซนเซอร์ HTS221:** เป็นเซนเซอร์ที่ติดตั้งอยู่บนบอร์ด ใช้สำหรับวัดอุณหภูมิและความชื้นสัมพัทธ์ (Temperature & Humidity)

- **พอร์ต USB:** บนบอร์ดมีพอร์ต USB สำหรับใช้งานอยู่ 2 ช่อง ได้แก่
  - 🔌 **Debug USB:** ใช้เชื่อมต่อบอร์ดกับคอมพิวเตอร์ เพื่อจ่ายไฟ อัปโหลดโปรแกรม (Flash Code) และดูข้อมูลผ่าน Serial Monitor ในระหว่างพัฒนาโปรแกรม
  - **OTG USB:** ใช้สำหรับเชื่อมต่อกับอุปกรณ์ USB ภายนอกตามรูปแบบการใช้งานที่รองรับ

- **ปุ่ม Boot และ Reset:** ใช้สำหรับควบคุมการทำงานของบอร์ด โดยปุ่ม Reset ใช้เริ่มการทำงานของบอร์ดใหม่ ส่วนปุ่ม Boot ใช้สำหรับเข้าสู่โหมดอัปโหลดโปรแกรมในกรณีที่จำเป็น

---

## 📡 หลักการทำงานของระบบ IoT

หลังจากรู้จักส่วนประกอบของบอร์ดแล้ว ต่อไปเราจะมาดูขั้นตอนการทำงานของระบบตั้งแต่การอ่านค่าจากเซนเซอร์ไปจนถึงการส่งข้อมูลผ่าน MQTT

1. บอร์ด Cucumber เชื่อมต่อกับเครือข่าย Wi-Fi เพื่อให้สามารถส่งข้อมูลผ่านอินเทอร์เน็ตได้ โดยใช้พอร์ต Debug USB สำหรับเชื่อมต่อกับคอมพิวเตอร์และจ่ายไฟให้บอร์ด
2. อ่านค่าอุณหภูมิ (°C) และความชื้น (%) จากเซนเซอร์ HTS221 ทุก ๆ 5 วินาที
3. นำค่าที่อ่านได้มาจัดรูปแบบเป็น JSON เช่น `{"temp": 28.5, "hum": 65.0}` เพื่อให้ง่ายต่อการนำไปใช้งาน
4. ส่งข้อมูลไปยัง MQTT Broker (`broker.hivemq.com`) ด้วยวิธี Publish โดยกำหนด Topic สำหรับรับส่งข้อมูลไว้ล่วงหน้า

เมื่อส่งข้อมูลเรียบร้อยแล้ว บอร์ดจะทำซ้ำตามขั้นตอนเดิมทุก ๆ 5 วินาที ทำให้สามารถติดตามค่าอุณหภูมิและความชื้นได้อย่างต่อเนื่อง

---

## 💻 ตัวอย่างโค้ด Arduino (C++) สำหรับบอร์ด Cucumber

ในส่วนนี้จะเป็นตัวอย่างโค้ดสำหรับใช้อ่านค่าอุณหภูมิและความชื้นจากเซนเซอร์ HTS221 แล้วส่งข้อมูลไปยัง MQTT Broker

สามารถเขียนและอัปโหลดโปรแกรมผ่าน Arduino IDE หรือ PlatformIO ได้ ก่อนเริ่มใช้งานควรติดตั้งไลบรารี `PubSubClient` และ `HTS221` ให้เรียบร้อย รวมถึงตั้งค่าบอร์ดให้ตรงกับรุ่นที่ใช้งาน

**ก่อนอัปโหลดโค้ด:** อย่าลืมเปลี่ยน `YOUR_WIFI_SSID` เป็นชื่อ Wi-Fi และ `YOUR_WIFI_PASSWORD` เป็นรหัสผ่านของเครือข่ายที่ต้องการเชื่อมต่อ

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include <Wire.h>
#include <HTS221Sensor.h>

// 1. ตั้งค่า Wi-Fi
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// 2. ตั้งค่า MQTT Broker
const char* mqtt_server = "broker.hivemq.com";
const char* mqtt_topic = "cn466/aiot/cucumber/sensor";

WiFiClient espClient;
PubSubClient client(espClient);
HTS221Sensor HumTemp(&Wire);

void setup_wifi() {
  delay(10);
  Serial.print("Connecting to ");
  Serial.println(ssid);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected!");
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    String clientId = "CucumberClient-" + String(random(0xffff), HEX);
    if (client.connect(clientId.c_str())) {
      Serial.println("connected");
    } else {
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  Wire.begin();
  HumTemp.begin();
  HumTemp.Enable();

  setup_wifi();
  client.setServer(mqtt_server, 1883);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  // อ่านค่าจากเซนเซอร์ HTS221
  float humidity = 0, temperature = 0;
  HumTemp.GetHumidity(&humidity);
  HumTemp.GetTemperature(&temperature);

  // สร้างข้อความ JSON
  String payload = "{\"temp\":" + String(temperature, 2) + ",\"hum\":" + String(humidity, 2) + "}";
  
  // ส่งข้อมูลขึ้น MQTT
  Serial.print("Publishing: ");
  Serial.println(payload);
  client.publish(mqtt_topic, payload.c_str());

  delay(5000); // ส่งข้อมูลทุกๆ 5 วินาที
}
```

---

## 🎯 สรุปท้ายบท

ในบทนี้เราได้รู้จักกับบอร์ด Cucumber และส่วนประกอบต่าง ๆ บนบอร์ด รวมถึงวิธีอ่านค่าอุณหภูมิและความชื้นจากเซนเซอร์ HTS221 และการส่งข้อมูลผ่าน MQTT

ข้อมูลที่ได้จากเซนเซอร์จะถูกนำไปใช้ในขั้นตอนถัดไป ซึ่งเราจะนำข้อมูลเหล่านี้มาประมวลผลและแสดงผลผ่านระบบที่พัฒนาขึ้น
