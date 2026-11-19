# 💬 บทที่ 4: การแจ้งเตือนและโต้ตอบผ่าน Telegram Bot

ในบทนี้ เราจะมาเพิ่มช่องทางติดต่อสื่อสารและหน้าจอควบคุมแบบเรียลไทม์ให้กับระบบ **AIoT Smart Environment Monitoring** โดยใช้ **Telegram Bot** ทำหน้าที่รับแจ้งเตือนสถานะอุณหภูมิและความชื้นที่รับมาจาก **MQTT Broker** (ต่อยอดจากข้อมูลของบอร์ด **Cucumber (ESP32-S2-Saola-1)** และเซนเซอร์ **HTS221** ในบทที่ 2 และการประมวลผลบน **Google Colab** ในบทที่ 3) รวมถึงรองรับการโต้ตอบเพื่อสอบถามค่าปัจจุบันได้ด้วย

---

## 🤖 ขั้นตอนการสร้าง Telegram Bot และรับ Chat ID

1. เปิดแอปพลิเคชัน Telegram แล้วค้นหาบอทชื่อ **`@BotFather`**
2. ส่งคำสั่ง `/newbot` เพื่อสร้างบอทใหม่
3. ตั้งชื่อบอท (Bot Name) และ Username ของบอท (ต้องลงท้ายด้วย `_bot` เช่น `CN466_AIoT_Bot`)
4. คัดลอก **HTTP API Token** ที่ `@BotFather` ส่งให้ (เก็บรหัสนี้ไว้เป็นความลับ ห้ามนำไปเผยแพร่ในที่สาธารณะ)
5. ค้นหา **Chat ID** ของตัวเองผ่านบอท เช่น `@userinfobot` เพื่อนำมาใช้ระบุปลายทางที่จะส่งข้อความ
6. **สำคัญ:** กดปุ่ม **Start** (หรือพิมพ์ `/start`) ในหน้าแชทกับบอทที่เราสร้างขึ้นก่อน เพื่อให้บอทมีสิทธิ์ส่งข้อความหาเราได้ (หากยังไม่เริ่มแชท อาจพบข้อผิดพลาด `404` หรือส่งข้อความไม่เข้า)

---

## ⚙️ เงื่อนไขการแจ้งเตือนและการโต้ตอบที่ใช้งานจริง

โค้ดชุดนี้เป็นเวอร์ชันที่ใช้งานจริงบน Google Colab โดยออกแบบให้ทำงานร่วมกันระหว่างการรับค่าจาก MQTT แบบเบื้องหลังและการรอรับคำสั่งจาก Telegram ดังนี้:

* **การแบ่งสถานะ (`NORMAL`, `HIGH`, `LOW`):** เปรียบเทียบค่าอุณหภูมิ (`latest_temp`) กับเกณฑ์ `TEMP_HIGH_LIMIT` และ `TEMP_LOW_LIMIT` เพื่อแจ้งเตือนเมื่ออุณหภูมิสูงหรือต่ำกว่ากำหนด และแจ้งเตือนเมื่ออุณหภูมิกลับมาอยู่ในช่วงปกติ
* **การคุมความถี่การแจ้งเตือน (`ALERT_COOLDOWN`):** กำหนดให้แจ้งเตือนซ้ำในสถานะผิดปกติเดิมได้ต่อเมื่อผ่านไปตามเวลาที่ตั้งไว้ เพื่อไม่ให้ส่งข้อความรัวเกินไป
* **การตรวจความสดของข้อมูล (`DATA_TIMEOUT`):** เมื่อมีคำสั่งเช็กค่า ระบบจะเทียบ `current_time - last_received_time` กับ `DATA_TIMEOUT` หากนานเกินกำหนดจะแจ้งว่า *"ไม่มีข้อมูลอัปเดต! บอร์ดอาจจะออฟไลน์หรือไม่ได้ส่งข้อมูลมา"* แทนการนำค่าเก่าค้างท่อมาตอบ
* **การโต้ตอบเช็กค่าปัจจุบัน:** รองรับคำสั่งในแชท Telegram เช่น `/status`, `temp`, `อุณหภูมิ` หรือ `เช็คค่า` เพื่อดูค่าอุณหภูมิและความชื้นล่าสุด

---

## 💻 โค้ด Python ที่ใช้งานจริงบน Google Colab

นี่คือโค้ด Python ชุดจริงที่ใช้รันบน **Google Colab** เพื่อรับข้อมูล JSON (`temp`, `hum`) จาก MQTT Broker (`broker.hivemq.com`) ผ่าน `paho-mqtt` และส่งแจ้งเตือน/ตอบกลับผ่าน Telegram Bot:

```python
import json
import time
import paho.mqtt.client as mqtt
import requests

# ตั้งค่า Telegram Bot
TELEGRAM_TOKEN = "TELEGRAM_TOKEN"
CHAT_ID = "CHAT_ID"

# ตัวแปรเก็บข้อมูลล่าสุดและเวลา
latest_temp = None
latest_hum = None
last_received_time = 0

# ขีดจำกัดอุณหภูมิ
TEMP_HIGH_LIMIT = 36.0
TEMP_LOW_LIMIT = 20.0

# ตัวแปรควบคุมการส่งซ้ำ
current_status = "NORMAL"
last_alert_time = 0
ALERT_COOLDOWN = 10  # แจ้งเตือนซ้ำทุก 10 วินาที (สอดคล้องกับบอร์ด)
DATA_TIMEOUT = 25
last_update_id = 0


def send_telegram_alert(message):
  url = f"[https://api.telegram.org/bot](https://api.telegram.org/bot){TELEGRAM_TOKEN}/sendMessage"
  payload = {"chat_id": CHAT_ID, "text": message, "parse_mode": "Markdown"}
  try:
    # เพิ่มพิมพ์ Log เวลาส่ง เพื่อเช็กในหน้าจอ Colab ว่าส่งตอนไหน
    print("📤 Sending to Telegram...")
    response = requests.post(url, json=payload, timeout=5)
    print(f"✅ Sent! Status: {response.status_code}")
    return response.json()
  except Exception as e:
    print(f"❌ Error sending telegram: {e}")


# ฟังก์ชันเมื่อเชื่อมต่อกับ MQTT Broker สำเร็จ
def on_connect(client, userdata, flags, reason_code, properties=None):
  print(f"Connected to MQTT Broker with code {reason_code}")
  client.subscribe("cn466/aiot/cucumber/sensor_01")


# ฟังก์ชันเมื่อมีข้อความส่งเข้ามาใน Topic
def on_message(client, userdata, msg):
  global latest_temp, latest_hum, last_received_time, current_status, last_alert_time
  try:
    payload_str = msg.payload.decode("utf-8")
    data = json.loads(payload_str)

    latest_temp = data.get("temp")
    latest_hum = data.get("hum")
    current_time = time.time()
    last_received_time = current_time

    print(f"📥 [MQTT Received] Temp: {latest_temp}°C, Humidity: {latest_hum}%")

    if latest_temp is not None:
      # --- 1. กรณีอุณหภูมิสูงเกิน 36°C ---
      if latest_temp > TEMP_HIGH_LIMIT:
        if current_status != "HIGH" or (
            current_time - last_alert_time >= ALERT_COOLDOWN
        ):
          alert_msg = (
              f"🚨 *เตือนภัย! อุณหภูมิสูงเกินกำหนด*\n"
              f"🌡️ อุณหภูมิ: {latest_temp} °C\n"
              f"💧 ความชื้น: {latest_hum} %"
          )
          send_telegram_alert(alert_msg)
          current_status = "HIGH"
          last_alert_time = current_time

      # --- 2. กรณีอุณหภูมิต่ำกว่า 20°C ---
      elif latest_temp < TEMP_LOW_LIMIT:
        if current_status != "LOW" or (
            current_time - last_alert_time >= ALERT_COOLDOWN
        ):
          alert_msg = (
              f"❄️ *เตือนภัย! อุณหภูมิต่ำเกินกำหนด*\n"
              f"🌡️ อุณหภูมิ: {latest_temp} °C\n"
              f"💧 ความชื้น: {latest_hum} %"
          )
          send_telegram_alert(alert_msg)
          current_status = "LOW"
          last_alert_time = current_time

      # --- 3. กรณีอุณหภูมิกลับมาปกติ ---
      elif TEMP_LOW_LIMIT <= latest_temp <= TEMP_HIGH_LIMIT:
        if current_status != "NORMAL":
          normal_msg = (
              f"✅ *สถานะกลับมาปกติแล้ว*\n"
              f"🌡️ อุณหภูมิปัจจุบัน: {latest_temp} °C\n"
              f"💧 ความชื้น: {latest_hum} %"
          )
          send_telegram_alert(normal_msg)
          current_status = "NORMAL"

  except Exception as e:
    print(f"❌ Error in on_message: {e}")


# ตั้งค่าและรัน MQTT Client
try:
  client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
except AttributeError:
  client = mqtt.Client()

client.on_connect = on_connect
client.on_message = on_message

client.connect("broker.hivemq.com", 1883, 60)
client.loop_start()  # รันเบื้องหลังแบบไม่บล็อกการทำงานหลัก
print("Starting MQTT Listener & Telegram Bot Service...")

# --- ลูปหลักสำหรับเช็กคำสั่ง /status จาก Telegram ---
try:
  while True:
    try:
      # ใช้ timeout สั้นๆ เพื่อให้ลูปหมุนเวียนได้ไว ไม่ค้าง
      url = (
          f"[https://api.telegram.org/bot](https://api.telegram.org/bot){TELEGRAM_TOKEN}/getUpdates"
          f"?offset={last_update_id + 1}&timeout=1"
      )
      response = requests.get(url, timeout=2).json()

      if "result" in response:
        for update in response["result"]:
          last_update_id = update["update_id"]

          if "message" in update and "text" in update["message"]:
            text = update["message"]["text"].strip().lower()
            print(f"💬 Telegram Command received: {text}")

            if text in ["/status", "temp", "อุณหภูมิ", "เช็คค่า"]:
              current_time = time.time()
              if (
                  latest_temp is not None
                  and (current_time - last_received_time) <= DATA_TIMEOUT
              ):
                reply_msg = (
                    f"📊 *ค่าอุณหภูมิและความชื้นปัจจุบัน*\n"
                    f"🌡️ อุณหภูมิ: {latest_temp} °C\n"
                    f"💧 ความชื้น: {latest_hum} %"
                )
              else:
                reply_msg = (
                    "⚠️ *ไม่มีข้อมูลอัปเดต!* บอร์ดอาจจะออฟไลน์หรือไม่ได้ส่งข้อมูลมา"
                )
              send_telegram_alert(reply_msg)

    except requests.exceptions.RequestException:
      pass

    time.sleep(
        0.5
    )  # หน่วงเวลาเล็กน้อยในลูป เพื่อไม่ให้กินทรัพยากร CPU ของ Colab เกินไป

except KeyboardInterrupt:
  print("Stopping service...")
  client.loop_stop()
  client.disconnect()
