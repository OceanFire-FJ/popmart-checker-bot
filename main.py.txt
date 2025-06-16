!apt-get update > /dev/null
!apt install chromium-chromedriver -y > /dev/null
!cp /usr/lib/chromium-browser/chromedriver /usr/bin

!pip install selenium > /dev/null

import os
import shutil
import time
import uuid
import requests
import traceback
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait

def send_discord(message, webhook_url):
    data = {"content": message}
    try:
        response = requests.post(webhook_url, json=data)
        if response.status_code == 204:
            print("✅ แจ้งเตือน Discord สำเร็จ")
        else:
            print("❌ แจ้งเตือน Discord ล้มเหลว:", response.text)
    except Exception as e:
        print("❌ Error ส่ง Discord:", e)

def create_driver(temp_profile_dir):
    options = Options()
    options.add_argument(f"--user-data-dir={temp_profile_dir}")
    options.add_argument("--headless=new")
    options.add_argument("--no-sandbox")
    options.add_argument("--disable-dev-shm-usage")
    options.add_argument("--disable-gpu")
    options.add_argument("--window-size=1280,800")
    return webdriver.Chrome(options=options)

def check_popmart_real_page(driver, url, webhook_url, counters):
    try:
        driver.get(url)

        WebDriverWait(driver, 15).until(
            lambda d: any(txt in d.page_source for txt in [
                "เลือกให้ฉัน",
                "ซื้อเลย",
                "แจ้งฉันเมื่อมีสินค้า",
                "สินค้านี้รองรับการซื้อผ่านแอปพลิเคชั่นเท่านั้น"
            ])
        )

        page = driver.page_source

        if "เลือกให้ฉัน" in page or "ซื้อเลย" in page:
            msg = f"✅ PopMart: มีสินค้าเข้าแล้ว!\n{url}"
            print(msg)
            send_discord(msg, webhook_url)
            counters["available"] += 1

        elif "แจ้งฉันเมื่อมีสินค้า" in page or "สินค้านี้รองรับการซื้อผ่านแอปพลิเคชั่นเท่านั้น" in page:
            msg = f"❌ PopMart: ยังไม่มีสินค้า\n{url}"
            print(msg)
            counters["unavailable"] += 1

        else:
            msg = f"😕 ไม่พบข้อความปุ่มในหน้าเว็บ: {url}"
            print(msg)
            send_discord(msg, webhook_url)
            counters["not_found"] += 1

    except Exception as e:
        error_msg = f"⛔ ERROR: {repr(e)}"
        print(error_msg)
        send_discord(error_msg, webhook_url)
        traceback.print_exc()

# ================================
# ✅ ลิงก์ PopMart ทั้งหมด 9 ลิงก์
urls = {
    "1": "https://www.popmart.com/th/products/2205",
    "2": "https://www.popmart.com/th/products/2174",
    "3": "https://www.popmart.com/th/products/2176",
    "4": "https://www.popmart.com/th/products/1432",
    "5": "https://www.popmart.com/th/products/1844",
    "6": "https://www.popmart.com/th/products/924",
    "7": "https://www.popmart.com/th/products/1208",
    "8": "https://www.popmart.com/th/products/923",
    "9": "https://www.popmart.com/th/pop-now/set/205"
}

webhook_url = "https://discord.com/api/webhooks/1383983423072239717/QMUTauSddOw73iZrLylwJXqdm1pYVixF7GV-16CDZa6ODTXYDgCSfkQjyrgNa1Swli9U"

# 🕒 รัน 4 ชั่วโมง
duration_sec = 4 * 60 * 60
hour_checkpoint = 1 * 60 * 60  # ส่งสรุปทุกชั่วโมง

# 🎯 เตรียมเคาน์เตอร์ลิงก์
counters = {
    key: {"available": 0, "unavailable": 0, "not_found": 0}
    for key in urls
}

# ================================
# 🔁 เริ่มจับเวลา
start_time = time.time()
last_summary_time = start_time
round_count = 0

while (time.time() - start_time) < duration_sec:
    round_count += 1
    print(f"\n⏱️ รอบที่ {round_count}")

    for key, url in urls.items():
        print(f"🔗 เช็คลิงก์ {key}: {url}")
        temp_profile_dir = f"/tmp/chrome-profile-{uuid.uuid4().hex}"
        os.makedirs(temp_profile_dir, exist_ok=True)

        try:
            driver = create_driver(temp_profile_dir)
            check_popmart_real_page(driver, url, webhook_url, counters[key])
        finally:
            driver.quit()
            del driver
            shutil.rmtree(temp_profile_dir, ignore_errors=True)

    # 🕒 ส่งสรุปทุก 1 ชั่วโมง
    if (time.time() - last_summary_time) >= hour_checkpoint:
        elapsed_hr = int((time.time() - start_time) // 3600)
        summary = f"🧾 สรุปผลหลัง {elapsed_hr} ชั่วโมง (รวม {round_count} รอบ x {len(urls)} ลิงก์)\n"
        for key, result in counters.items():
            summary += (
                f"\n🔗 ลิงก์ {key} ({urls[key]}):\n"
                f"✅ มีสินค้า: {result['available']} ครั้ง\n"
                f"❌ ยังไม่มีสินค้า: {result['unavailable']} ครั้ง\n"
                f"😕 ไม่พบปุ่ม: {result['not_found']} ครั้ง\n"
            )
        send_discord(summary, webhook_url)
        last_summary_time = time.time()

# 🧾 สรุปสุดท้ายตอนครบ 4 ชั่วโมง
summary = f"📊 สรุปผลสุดท้ายหลังครบ 4 ชั่วโมง (รวม {round_count} รอบ x {len(urls)} ลิงก์)\n"
for key, result in counters.items():
    summary += (
        f"\n🔗 ลิงก์ {key} ({urls[key]}):\n"
        f"✅ มีสินค้า: {result['available']} ครั้ง\n"
        f"❌ ยังไม่มีสินค้า: {result['unavailable']} ครั้ง\n"
        f"😕 ไม่พบปุ่ม: {result['not_found']} ครั้ง\n"
    )

send_discord(summary, webhook_url)
print("\n📤 ส่งสรุปสุดท้ายเข้า Discord เรียบร้อย ✅")
