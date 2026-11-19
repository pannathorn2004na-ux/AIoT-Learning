# 🧠 บทที่ 3: การประมวลผลข้อมูลด้วย Python บน Google Colab

ในบทนี้ เราจะนำข้อมูลอุณหภูมิและความชื้นที่ส่งมาจากบอร์ด Cucumber ผ่านโปรโตคอล MQTT มารับและประมวลผลบน Google Colab โดยใช้ภาษา Python เพื่อแสดงข้อมูลและตรวจสอบสถานะอุณหภูมิตามเงื่อนไขที่กำหนด

Google Colab ทำหน้าที่เป็นส่วนประมวลผลของระบบ โดยรับข้อมูลจาก MQTT Broker แล้วนำค่าที่ได้รับมาตรวจสอบ ก่อนแสดงผลให้ผู้ใช้ทราบว่าสถานะอุณหภูมิในขณะนั้นเป็นอย่างไร

---

## ☁️ ทำไมต้องใช้ Google Colab?

Google Colab เป็นเครื่องมือสำหรับเขียนและทดลองใช้งานภาษา Python ผ่านเว็บเบราว์เซอร์ โดยมีข้อดีดังนี้

- **ใช้งานผ่านเว็บได้:** ไม่จำเป็นต้องติดตั้ง Python หรือโปรแกรมเพิ่มเติมบนเครื่อง
- **รองรับไลบรารี Python:** สามารถติดตั้งและใช้งานไลบรารีต่าง ๆ สำหรับการประมวลผลข้อมูลได้
- **เชื่อมต่อกับบริการภายนอกได้:** สามารถรับส่งข้อมูลผ่าน MQTT และเชื่อมต่อกับบริการอื่น ๆ เช่น Telegram Bot ได้

---

## 💻 โค้ด Python สำหรับ Google Colab

เริ่มต้นโดยเข้าไปที่ [Google Colab](https://colab.research.google.com/) แล้วสร้าง Notebook ใหม่ จากนั้นคัดลอกโค้ดด้านล่างไปวางใน Code Cell เพื่อทดลองรับข้อมูลจากบอร์ด Cucumber ผ่าน MQTT

### 1. ติดตั้งไลบรารีและนำเข้าโมดูล

ติดตั้งไลบรารี `paho-mqtt` สำหรับเชื่อมต่อและรับส่งข้อมูลผ่าน MQTT และนำเข้าโมดูล `json` สำหรับจัดการข้อมูลที่ได้รับ

```python
!pip install paho-mqtt

import json
import paho.mqtt.client as mqtt
```

### 2. กำหนดเงื่อนไขตรวจสอบสถานะอุณหภูมิ

กำหนดขอบเขตอุณหภูมิสำหรับตรวจสอบและจำแนกสถานะออกเป็น 3 กรณี ได้แก่ HIGH, LOW และ NORMAL

```python
TEMP_HIGH_LIMIT = 36.0
TEMP_LOW_LIMIT = 20.0

def analyze_environment(temp):
    if temp > TEMP_HIGH_LIMIT:
        return "HIGH"
    elif temp < TEMP_LOW_LIMIT:
        return "LOW"
    else:
        return "NORMAL"
```

เงื่อนไขการตรวจสอบมีดังนี้

- **HIGH:** อุณหภูมิสูงกว่า 36°C
- **LOW:** อุณหภูมิต่ำกว่า 20°C
- **NORMAL:** อุณหภูมิตั้งแต่ 20–36°C

**หมายเหตุ:** ระบบใช้ค่าอุณหภูมิเป็นเงื่อนไขในการจำแนกสถานะ ส่วนค่าความชื้นจะถูกรับและแสดงผลเพื่อให้ติดตามข้อมูลสภาพแวดล้อม แต่ไม่ได้ใช้เป็นเงื่อนไขในการจำแนกสถานะ

### 3. สร้างฟังก์ชันรับและประมวลผลข้อมูลจาก MQTT

เมื่อมีข้อมูลส่งเข้ามายัง MQTT Broker ฟังก์ชัน `on_message()` จะทำงานโดยอัตโนมัติ จากนั้นแปลงข้อมูล JSON และนำค่าอุณหภูมิไปตรวจสอบตามเงื่อนไขที่กำหนด

```python
def on_message(client, userdata, msg):
    try:
        payload_str = msg.payload.decode("utf-8")
        data = json.loads(payload_str)

        temp = data.get("temp")
        hum = data.get("hum")

        if temp is not None:
            temp = float(temp)
            status = analyze_environment(temp)

            print(f"📥 [MQTT Received]")
            print(f"อุณหภูมิ: {temp}°C")
            print(f"ความชื้น: {hum}%")
            print(f"📊 สถานะ: {status}")
            print("-" * 40)

    except Exception as e:
        print(f"❌ Error parsing data: {e}")
```

### 4. ตั้งค่าการเชื่อมต่อ MQTT

กำหนดที่อยู่ของ MQTT Broker และ Topic ให้ตรงกับค่าที่ตั้งไว้ในโปรแกรมของบอร์ด Cucumber จากนั้นสร้าง MQTT Client และกำหนดฟังก์ชันสำหรับเชื่อมต่อและรับข้อมูล

```python
broker_address = "broker.hivemq.com"
mqtt_topic = "cn466/aiot/cucumber/sensor_01"

def on_connect(client, userdata, flags, rc):
    print(f"Connected to MQTT Broker with code {rc}")
    client.subscribe(mqtt_topic)

client = mqtt.Client()
client.on_connect = on_connect
client.on_message = on_message

print(f"Connecting to MQTT Broker: {broker_address}...")

client.connect(broker_address, 1883, 60)

# เริ่มรับข้อมูลแบบ Real-time
client.loop_start()

print(f"🚀 กำลังรอรับข้อมูลจาก Topic: {mqtt_topic}")
print("เปิดบอร์ด Cucumber เพื่อเริ่มส่งข้อมูลได้เลย!")
```

เมื่อรันโค้ดครบทุกส่วนแล้ว Google Colab จะเริ่มรอรับข้อมูลจากบอร์ด Cucumber เมื่อมีข้อมูลส่งเข้ามา ระบบจะแสดงค่าอุณหภูมิ ความชื้น และผลการตรวจสอบสถานะตามเงื่อนไขที่กำหนด

---

## 🔄 ขั้นตอนการทำงานของระบบ

1. บอร์ด Cucumber อ่านค่าอุณหภูมิและความชื้นจากเซนเซอร์ HTS221
2. บอร์ดจัดรูปแบบข้อมูลเป็น JSON และส่งไปยัง MQTT Broker
3. Google Colab เชื่อมต่อกับ MQTT Broker และสมัครรับข้อมูลจาก Topic ที่กำหนด
4. Python รับข้อมูลและแปลงข้อมูล JSON เพื่อนำค่ามาใช้งาน
5. ระบบนำค่าอุณหภูมิไปตรวจสอบตามเงื่อนไขที่กำหนด
6. แสดงค่าอุณหภูมิ ความชื้น และผลการตรวจสอบบน Google Colab

## 📝 สิ่งที่ควรรู้

- Topic ที่กำหนดใน Google Colab ต้องตรงกับ Topic ที่ใช้ในโปรแกรมของบอร์ด Cucumber
- Google Colab ต้องเปิดใช้งานอยู่และเชื่อมต่ออินเทอร์เน็ต เพื่อให้สามารถรับข้อมูลจาก MQTT ได้
- หากต้องการรับข้อมูลอย่างต่อเนื่อง ต้องปล่อยให้การเชื่อมต่อ MQTT ทำงานอยู่
- ตัวอย่างนี้ใช้การประมวลผลตามเงื่อนไขที่กำหนด (Rule-based Processing) ยังไม่ใช่การใช้โมเดล AI หรือ Machine Learning ในการเรียนรู้และทำนายผลจริง
- ค่าอุณหภูมิที่ใช้แบ่งสถานะเป็นเงื่อนไขของระบบตัวอย่าง ไม่ใช่เกณฑ์มาตรฐานทางสุขภาพหรือความปลอดภัย

---

## 🎯 สรุปท้ายบท

ในบทนี้เราได้เรียนรู้การใช้ Python บน Google Colab เพื่อเชื่อมต่อกับ MQTT Broker และรับข้อมูลอุณหภูมิและความชื้นจากบอร์ด Cucumber รวมถึงการนำข้อมูลที่ได้รับมาตรวจสอบและประมวลผลตามเงื่อนไขที่กำหนด

ผลการประมวลผลที่ได้จะถูกนำไปใช้เป็นส่วนหนึ่งของระบบแจ้งเตือนและโต้ตอบกับผู้ใช้งานผ่าน Telegram Bot ซึ่งจะอธิบายในบทถัดไป
