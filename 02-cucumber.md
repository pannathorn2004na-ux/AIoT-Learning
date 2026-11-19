# 🔌 บทที่ 2: บอร์ด Cucumber และเซนเซอร์วัดอุณหภูมิและความชื้น

ในบทนี้เราจะมาทำความรู้จักกับบอร์ด **Cucumber (ESP32-S2-Saola-1)** ซึ่งเป็นอุปกรณ์หลักในส่วนของระบบ IoT พร้อมเรียนรู้ส่วนประกอบบนบอร์ดและการทำงานของเซนเซอร์ **HTS221** สำหรับตรวจวัดอุณหภูมิและความชื้น จากนั้นจะทดลองเขียนโปรแกรมด้วย **Arduino / PlatformIO** เพื่ออ่านค่าจากเซนเซอร์และส่งข้อมูลไปยัง **MQTT Broker** เพื่อนำไปประมวลผลในขั้นตอนถัดไป

---

## 🛠️ ทำความรู้จักกับบอร์ด Cucumber (ESP32-S2-Saola-1)

ก่อนเริ่มเขียนโปรแกรม เรามาทำความรู้จักกับส่วนประกอบต่าง ๆ บนบอร์ด Cucumber กันก่อน เพื่อให้เข้าใจหน้าที่ของแต่ละส่วนและสามารถใช้งานบอร์ดได้อย่างถูกต้อง

<p align="center">
  <img width="1532" height="850" alt="image" src="https://github.com/user-attachments/assets/1a8a5c58-6696-48ed-a8f3-1e454fa71d77" />
  <br>
  <em>ภาพแสดงตำแหน่งอุปกรณ์บนบอร์ด Cucumber</em>
</p>

ส่วนประกอบที่ควรรู้จักมีดังนี้:

* **เซนเซอร์ HTS221:** ใช้สำหรับตรวจวัดอุณหภูมิ (Temperature) และความชื้นสัมพัทธ์ (Relative Humidity) จากสภาพแวดล้อม
* **พอร์ต USB:** บนบอร์ดมีพอร์ต USB สำหรับใช้งาน 2 ช่อง ได้แก่:
  * **🔌 Debug USB:** ใช้เชื่อมต่อบอร์ดกับคอมพิวเตอร์ เพื่อจ่ายไฟ อัปโหลดโปรแกรม (Flash Code) และตรวจสอบข้อมูลผ่าน Serial Monitor ระหว่างพัฒนาโปรแกรม
  * **OTG USB:** ใช้สำหรับเชื่อมต่อกับอุปกรณ์ USB ภายนอกตามรูปแบบการใช้งานที่บอร์ดรองรับ
* **ปุ่ม Boot และ Reset:** ใช้ควบคุมการทำงานของบอร์ด โดยปุ่ม **Reset** ใช้เริ่มการทำงานของบอร์ดใหม่ ส่วนปุ่ม **Boot** ใช้สำหรับเข้าสู่โหมดอัปโหลดโปรแกรมในกรณีที่จำเป็น

---

## 📡 หลักการทำงานของระบบ IoT

หลังจากรู้จักส่วนประกอบของบอร์ดแล้ว ต่อไปเราจะมาดูขั้นตอนการทำงานของระบบ ตั้งแต่การอ่านค่าจากเซนเซอร์ไปจนถึงการส่งข้อมูลผ่าน MQTT

1. **เชื่อมต่อ Wi-Fi:** บอร์ด Cucumber เชื่อมต่อกับเครือข่าย Wi-Fi (คลื่น 2.4GHz) เพื่อให้สามารถสื่อสารและส่งข้อมูลผ่านอินเทอร์เน็ตได้ โดยใช้พอร์ต **Debug USB** สำหรับเชื่อมต่อกับคอมพิวเตอร์หรือแหล่งจ่ายไฟ
2. **อ่านค่าจากเซนเซอร์:** อ่านค่าอุณหภูมิ (°C) และความชื้นสัมพัทธ์ (%) จากเซนเซอร์ **HTS221** ผ่านบัส I2C (ขา `SDA = 41`, `SCL = 40`, Address `0x5F`)
3. **จัดรูปแบบข้อมูล:** นำค่าที่อ่านได้มาจัดรูปแบบเป็น **JSON** เพื่อให้ง่ายต่อการส่งและนำไปประมวลผล เช่น `{"temp": 28.50, "hum": 65.00}`
4. **ส่งข้อมูลผ่าน MQTT:** ส่งข้อมูลไปยัง **MQTT Broker** (`broker.hivemq.com`) ด้วยวิธี **Publish** โดยกำหนด Topic สำหรับส่งข้อมูลไว้ล่วงหน้า

เมื่อส่งข้อมูลเรียบร้อยแล้ว บอร์ดจะทำซ้ำตามขั้นตอนเดิมทุก ๆ 10 วินาที ทำให้สามารถรับข้อมูลอุณหภูมิและความชื้นได้อย่างต่อเนื่อง

---

## 💻 ตัวอย่างโค้ดสำหรับบอร์ด Cucumber (ESP32-S2-Saola-1)

ในส่วนนี้เป็นตัวอย่างโค้ดสำหรับเชื่อมต่อ Wi-Fi (คลื่น 2.4GHz) อ่านค่าอุณหภูมิและความชื้นจากเซนเซอร์ **HTS221** ผ่านบัส I2C (ขา `SDA = 41`, `SCL = 40`, Address `0x5F`) และส่งข้อมูลรูปแบบ JSON ไปยัง **MQTT Broker**

สามารถเขียนและอัปโหลดโปรแกรมผ่าน Arduino IDE หรือ PlatformIO ได้ ก่อนเริ่มใช้งานควรติดตั้งไลบรารีที่จำเป็น ได้แก่ `PubSubClient` และ `Adafruit HTS221` รวมถึงตั้งค่าบอร์ดให้ตรงกับรุ่นที่ใช้งาน

> **ก่อนอัปโหลดโค้ด:** เปลี่ยน `YOUR_WIFI_SSID` เป็นชื่อ Wi-Fi (2.4GHz เท่านั้น) และ `YOUR_WIFI_PASSWORD` เป็นรหัสผ่านของเครือข่าย รวมถึงตรวจสอบหรือปรับ `mqtt_topic` ให้ตรงกับที่ใช้งาน

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <PubSubClient.h>
#include <Adafruit_HTS221.h>

// 1. ตั้งค่า Wi-Fi (คลื่น 2.4GHz เท่านั้น)
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// 2. ตั้งค่า MQTT Broker
const char* mqtt_server = "broker.hivemq.com";
const char* mqtt_topic = "cn466/aiot/cucumber/sensor_01"; // เปลี่ยนชื่อ Topic ตามต้องการ

WiFiClient espClient;
PubSubClient client(espClient);
Adafruit_HTS221 hts;

void setup_wifi() {
  delay(10);
  Serial.print("\nConnecting to ");
  Serial.println(ssid);
  
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  Serial.println("\nWiFi connected!");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    String clientId = "CucumberClient-" + String(random(0xffff), HEX);
    if (client.connect(clientId.c_str())) {
      Serial.println("connected!");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  while (!Serial) delay(10);

  Serial.println("Starting HTS221 test...");

  // กำหนดขา I2C สำหรับบอร์ด Cucumber (SDA = 41, SCL = 40)
  Wire.begin(41, 40);

  // เริ่มต้นเซนเซอร์ HTS221 ที่ I2C Address 0x5F
  if (!hts.begin_I2C(0x5F, &Wire)) {
    Serial.println("Failed to find HTS221 chip! Check wiring.");
    while (1) { delay(10); }
  }
  Serial.println("HTS221 Found!");

  setup_wifi();
  client.setServer(mqtt_server, 1883);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  // อ่านค่าจากเซนเซอร์ HTS221
  sensors_event_t temp_event, humidity_event;
  hts.getEvent(&humidity_event, &temp_event);

  float temperature = temp_event.temperature;
  float humidity = humidity_event.relative_humidity;

  // สร้างข้อความรูปแบบ JSON
  String payload = "{\"temp\":" + String(temperature, 2) + ",\"hum\":" + String(humidity, 2) + "}";

  Serial.print("Publishing -> ");
  Serial.println(payload);
  
  // ส่งข้อมูลขึ้น MQTT Topic
  client.publish(mqtt_topic, payload.c_str());

  delay(10000); // ส่งข้อมูลทุกๆ 10 วินาที
}
