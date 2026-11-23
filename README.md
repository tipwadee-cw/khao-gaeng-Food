# ครัวข้าวแกง — Firebase Spark (ใช้ฟรีตามโควต้า)

โปรเจกต์เว็บหน้าลูกค้า + Admin แบบ static HTML/CSS/JS สามารถเปิดหน้า Demo ได้ทันที และ deploy บน **Firebase Hosting** โดยไม่ใช้ Vercel หรือ Blaze

## สิ่งที่มีในเวอร์ชันนี้
- ข้าวราดแกง 1/2/3 อย่าง, ราคา Admin เปลี่ยนได้, เมนูเปิดขายรายวัน, บันทึกคำสั่งซื้อ, รับอาหาร 06:00–15:00 น.
- อัปโหลดสลิป **แบบบีบอัดเป็น JPEG และเก็บใน Firestore document** เฉพาะออเดอร์ของผู้ใช้ (ไม่มี Firebase Storage)
- Admin ดูสลิปและกดเปลี่ยนสถานะออเดอร์เอง (ไม่มีระบบพิสูจน์ว่าเงินเข้าจริงอัตโนมัติ)
- โหวตเมนูที่ไม่ได้กำหนดขายในวันถัดไปและเสนอเมนูใหม่ ใช้เวลา 16:00–00:00 น. ประเทศไทย, คะแนนแยกตามวัน
- Admin ดูผลโหวต, อนุมัติข้อเสนอเข้าคลัง, กำหนดราคาข้าว 1/2/3 อย่างและราคาเมนูพิเศษ

## ข้อจำกัดสำคัญก่อนเปิดรับเงินจริง
1. **ไม่มี Slip Verification API**: การอ่านข้อความบนสลิปเพียงอย่างเดียว *ยืนยันเงินเข้าจริงไม่ได้* ต้องเปิดบัญชีและตรวจยอดโอนผ่านธนาคารในหน้า Admin ก่อนอนุมัติ
2. **จำนวนอาหารยังเป็นตัวเลขแสดง ไม่ได้จอง/ตัดสต็อกแบบ transaction**: ลูกค้าหลายคนอาจสั่งเกินจำนวน ต้องตรวจออเดอร์ก่อนอนุมัติรับเงินและทำอาหาร
3. **ยอดเงินในออเดอร์มาจากฝั่งลูกค้า**: Admin ต้องตรวจยอดตามราคาในร้านอีกครั้งก่อนยืนยัน ระบบนี้ยังไม่มี backend ที่เชื่อถือได้สำหรับการคำนวณ/บังคับชำระเงิน
4. **สลิปใน Firestore เป็นทางออกชั่วคราวสำหรับปริมาณน้อย**: จำกัดภาพราว 120KB หลังบีบอัด (สตริง <160KB), อาจเสียความคมชัด ตรวจอ่านเลขลำบาก ไม่เหมาะสำหรับการเก็บรูปจำนวนมาก ต้องกำหนดนโยบายลบ/เก็บข้อมูลการเงินให้ถูกต้อง และตรวจโควต้าบ่อย ๆ
5. Firebase Anonymous Auth ใช้เพื่อแยกสิทธิ์ลูกค้า แต่ไม่ป้องกันบอท/โหวตซ้ำด้วยบัญชีใหม่หรือเครื่องอื่น ต้องเพิ่ม anti-abuse และตรวจเวลาโหวตฝั่ง server ก่อนใช้งานสาธารณะจริง
6. โหวตและการเสนอเมนูตรวจเวลาในเบราว์เซอร์เท่านั้นใน MVP นี้ (Rules ยังไม่บังคับเวลา/การเปิดขายรายวัน) จึงไม่ทนต่อการแก้ไขเวลา/ส่งคำขอเอง
7. นี่คือ **ต้นแบบพร้อมทดสอบ** ไม่ใช่ระบบรับชำระเงินที่ผ่าน audit/security review; ห้ามเปิดรับเงินจริงก่อนแก้ข้อ 1–6 ตามความเสี่ยงที่ยอมรับได้

## เปิด Demo โดยยังไม่ตั้ง Firebase
- เปิด `public/index.html` ผ่าน local web server (ไม่ใช้ file:// เพราะเป็น ES module)
- คำสั่ง: `python -m http.server 8000 --directory public` แล้วเปิด http://localhost:8000
- ข้อมูล Demo อยู่ใน localStorage ของ browser เดียวกัน; กด Admin แล้วเข้าสู่โหมด Demo ได้เลย

## เชื่อม Firebase จริง
1. ไป https://console.firebase.google.com และสร้าง Project ใหม่ เลือก **Spark** (ไม่ผูก billing)
2. Build > Authentication > Sign-in method เปิด **Anonymous** และ **Email/Password**
3. Build > Firestore Database สร้างฐานข้อมูล แล้ว Deploy rules ที่ไฟล์ `firestore.rules` ให้เรียบร้อย **ก่อน** เปิดเว็บจริง
4. Project Settings > Your apps > Web App คัดลอก config ไปใส่ใน `public/firebase-config.js` แทน `null` (**ไม่ใส่ service account / private key**)
5. ติดตั้ง Firebase CLI: `npm install -g firebase-tools`, แล้ว `firebase login`
6. เปิด terminal ในโฟลเดอร์โปรเจกต์: `firebase use --add` เพื่อเลือก Project
7. `firebase deploy --only firestore,hosting`
8. ไป Firebase Console > Authentication > Users > Add user สร้างบัญชี Email/Password สำหรับ Admin
9. คัดลอก **UID** ของบัญชี Admin; ไป Firestore > เริ่ม collection `admins` > document ID = UID > field `enabled` ชนิด boolean เป็น `true` (เฉพาะเจ้าของโปรเจกต์ใน Console เท่านั้นที่สร้างได้)
10. ล็อกอินหลังบ้านด้วยอีเมล/รหัสผ่าน Admin และเพิ่มเมนูจริงใน **คลังเมนู** จากนั้นไป **เมนูวันนี้** เลือกรายการและตั้งจำนวน กำหนดราคา/ชื่อร้าน
11. ในแท็บ `ราคา/ร้าน` ใส่พร้อมเพย์เมื่อพร้อมรับเงินจริง; ก่อนหน้านั้นเว็บจะแสดงว่า **ยังไม่มีบัญชีรับเงิน**

**หมายเหตุ:** เมื่อเริ่ม Firebase จริง เมนูตัวอย่างใน Demo จะไม่ถูกย้ายอัตโนมัติ ให้เพิ่มเมนูด้วย Admin เอง เพื่อไม่ให้ข้อมูลปลอมขึ้นหน้าขายจริง

## แหล่งข้อมูลและค่าใช้จ่าย
Firebase Spark/Hosting/Firestore/Auth มีโควต้าฟรี แต่ไม่ได้ฟรีไม่จำกัด และบริการภายนอกตรวจสลิปอาจคิดค่าบริการ การเก็บสลิปใน Firestore รวมกับ read/write/storage quota  ดู https://firebase.google.com/pricing

## โครงสร้าง
- `public/index.html`, `style.css`, `app.js`, `firebase-config.js`
- `firestore.rules` คุมสิทธิ์ Admin และลูกค้า, `firestore.indexes.json` ยกเลิก single-field index สำหรับข้อมูลรูป base64
- `firebase.json` config Hosting + Firestore
