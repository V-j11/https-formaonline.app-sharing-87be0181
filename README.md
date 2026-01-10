import imaplib
import email
import os
import base64
from email.header import decode_header
from datetime import datetime
import re

# ========== الإعدادات - عدّل هنا فقط ==========

EMAIL = “78mshahrani@gmail.com”
PASSWORD = “ضع_كلمة_المرور_هنا”  # App Password من Gmail
SAVE_FOLDER = “المرفقات_المحملة”

# ========== لا تعدل تحت هذا الخط ==========

def decode_text(text):
“”“فك تشفير النصوص العربية والمشفرة”””
if not text:
return “”

```
try:
    decoded = decode_header(text)[0]
    if isinstance(decoded[0], bytes):
        charset = decoded[1] or 'utf-8'
        return decoded[0].decode(charset, errors='ignore')
    return str(decoded[0])
except:
    return str(text)
```

def decode_attachment(part):
“”“فك تشفير المرفق”””
try:
payload = part.get_payload(decode=True)
if payload:
return payload

```
    # محاولة فك base64 يدوياً
    raw_payload = part.get_payload()
    if isinstance(raw_payload, str):
        return base64.b64decode(raw_payload)
    
    return raw_payload
except Exception as e:
    print(f"    ⚠️ خطأ في فك التشفير: {e}")
    return None
```

def clean_name(name):
“”“تنظيف أسماء الملفات”””
name = re.sub(r’[\/*?:”<>|]’, ‘_’, name)
return name[:150]

def setup_folder():
“”“إنشاء مجلد الحفظ”””
if not os.path.exists(SAVE_FOLDER):
os.makedirs(SAVE_FOLDER)

def download():
“”“تحميل المرفقات”””

```
print("\n" + "="*60)
print("📥 تحميل المرفقات من Gmail")
print("="*60 + "\n")

try:
    # الاتصال
    print("🔄 جاري الاتصال...")
    mail = imaplib.IMAP4_SSL("imap.gmail.com")
    
    # تسجيل الدخول
    print("🔐 جاري تسجيل الدخول...")
    mail.login(EMAIL, PASSWORD)
    print("✅ تم الدخول بنجاح!\n")
    
    # فتح البريد الوارد
    mail.select("inbox")
    
    # البحث عن آخر 30 رسالة
    _, msgs = mail.search(None, "ALL")
    msg_list = msgs[0].split()
    
    # أخذ آخر 30 رسالة
    recent = msg_list[-30:] if len(msg_list) > 30 else msg_list
    recent = list(reversed(recent))
    
    print(f"📧 فحص آخر {len(recent)} رسالة...\n")
    
    setup_folder()
    count = 0
    
    # معالجة كل رسالة
    for i, num in enumerate(recent, 1):
        _, data = mail.fetch(num, "(RFC822)")
        msg = email.message_from_bytes(data[0][1])
        
        # معلومات الرسالة
        subject = decode_text(msg.get("Subject", "بدون عنوان"))
        sender = decode_text(msg.get("From", ""))
        
        print(f"[{i}/{len(recent)}] {subject[:40]}...")
        
        # البحث عن المرفقات
        found = False
        
        for part in msg.walk():
            if part.get_content_maintype() == 'multipart':
                continue
            if part.get('Content-Disposition') is None:
                continue
            
            filename = part.get_filename()
            if filename:
                found = True
                filename = decode_text(filename)
                filename = clean_name(filename)
                
                # فك تشفير المرفق
                file_data = decode_attachment(part)
                
                if file_data:
                    # حفظ الملف
                    path = os.path.join(SAVE_FOLDER, filename)
                    
                    # تجنب التكرار
                    if os.path.exists(path):
                        name, ext = os.path.splitext(filename)
                        c = 1
                        while os.path.exists(path):
                            path = os.path.join(SAVE_FOLDER, f"{name}_{c}{ext}")
                            c += 1
                    
                    with open(path, "wb") as f:
                        f.write(file_data)
                    
                    size = len(file_data) / 1024
                    print(f"  ✅ {filename} ({size:.1f} KB)")
                    count += 1
                else:
                    print(f"  ❌ فشل فك التشفير: {filename}")
        
        if not found:
            print("  ⚠️ لا توجد مرفقات")
    
    mail.logout()
    
    print("\n" + "="*60)
    print(f"✅ انتهى التحميل!")
    print(f"📎 المرفقات المحملة: {count}")
    print(f"📁 المجلد: {os.path.abspath(SAVE_FOLDER)}")
    print("="*60)
    
except imaplib.IMAP4.error:
    print("\n❌ خطأ في تسجيل الدخول!")
    print("\n💡 الحل:")
    print("1. روح لـ: https://myaccount.google.com/apppasswords")
    print("2. اضغط 'Create app password'")
    print("3. انسخ كلمة المرور المكونة من 16 رقم")
    print("4. الصقها في PASSWORD أعلى الكود")
    
except Exception as e:
    print(f"\n❌ خطأ: {e}")
```

# ========== تشغيل البرنامج ==========

if **name** == “**main**”:
download()

```
# إبقاء النافذة مفتوحة
input("\n\nاضغط Enter للخروج...")
```