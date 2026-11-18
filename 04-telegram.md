# 💬 บทที่ 4: การแจ้งเตือนและโต้ตอบผ่าน Telegram Bot

ในบทนี้ เราจะมาเพิ่มช่องทางติดต่อสื่อสารให้กับระบบ AIoT ของเรา โดยใช้ **Telegram Bot** ทำหน้าที่เป็นหน้าจอ UI บนมือถือ เพื่อรับรายงานสภาพอากาศและแจ้งเตือนเมื่อเกิดเหตุการณ์สำคัญ

---

## 🤖 ขั้นตอนการสร้าง Telegram Bot
1. เปิดแอปพลิเคชัน Telegram แล้วค้นหาบอทชื่อ **`@BotFather`**
2. ส่งคำสั่ง `/newbot` เพื่อสร้างบอทใหม่
3. ตั้งชื่อบอท (Bot Name) และ Username ของบอท (ต้องลงท้ายด้วย `_bot` เช่น `MyAIoT_Bot`)
4. เมื่อสร้างเสร็จ `@BotFather` จะส่ง **HTTP API Token** มาให้ (เก็บรหัสนี้ไว้เป็นความลับสำหรับใช้เขียนโค้ด)
5. ค้นหา Chat ID ของตัวเองผ่านบอทเช่น `@userinfobot` เพื่อนำมาใช้รับข้อความ

---

## 💻 โค้ด Python สำหรับส่งข้อความเข้า Telegram
เราสามารถนำฟังก์ชันส่ง Telegram นี้ไปรวมเข้ากับโค้ด MQTT ใน Google Colab (จากบทที่ 3) เพื่อให้เมื่อ AI วิเคราะห์สภาพอากาศแล้ว ส่งข้อความแจ้งเตือนเข้ามือถือทันที:

```python
import requests

# ตั้งค่า Telegram Bot Token และ Chat ID ของคุณ
TELEGRAM_BOT_TOKEN = "YOUR_BOT_TOKEN_HERE"
TELEGRAM_CHAT_ID = "YOUR_CHAT_ID_HERE"

def send_telegram_message(message):
    url = f"[https://api.telegram.org/bot](https://api.telegram.org/bot){TELEGRAM_BOT_TOKEN}/sendMessage"
    payload = {
        "chat_id": TELEGRAM_CHAT_ID,
        "text": message,
        "parse_mode": "Markdown"
    }
    
    try:
        response = requests.post(url, json=payload)
        if response.status_code == 200:
            print("📲 [Telegram] ส่งข้อความแจ้งเตือนสำเร็จ!")
        else:
            print(f"❌ [Telegram Error] ไม่สามารถส่งข้อความได้: {response.text}")
    except Exception as e:
        print(f"❌ [Telegram Exception]: {e}")

# ตัวอย่างการเรียกใช้งานฟังก์ชัน
# send_telegram_message("🚨 แจ้งเตือนจาก AIoT: ตรวจพบอุณหภูมิสูงเกินกำหนด!")
