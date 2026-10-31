# ระบบคอร์สแวร์ (CAI) วิทยาการคำนวณ: ความจริงเสริม (AR)

ระบบคอมพิวเตอร์ช่วยสอน (Courseware) ผ่านเว็บเบราว์เซอร์ สำหรับรายวิชาวิทยาการคำนวณ ชั้นประถมศึกษาปีที่ 5 โรงเรียนหนูหิ่นวิทยาคม โดยเน้นเนื้อหาเรื่อง "ความจริงเสริม (Augmented Reality: AR)" พร้อมระบบจัดการฐานข้อมูลหลังบ้านด้วย Supabase

## 🌟 ฟีเจอร์หลัก (Features)

**สำหรับผู้เรียน (Students):**
*   **ระบบลงทะเบียนด้วยรหัส 11 หลัก:** ตรวจสอบสิทธิ์การสมัครจากฐานข้อมูลรายชื่อ (ป้องกันบุคคลภายนอก)
*   **แบบทดสอบก่อน-หลังเรียน:** ระบบสุ่มข้อสอบชุด A และ B อัตโนมัติตามเลขท้ายรหัสนักศึกษา (คู่/คี่) 
*   **บทเรียนแบบ Interactive:** เนื้อหา 5 หน่วยเรียน พร้อมคำถามแทรกที่ต้องตอบให้ถูกเพื่อปลดล็อกบทเรียนถัดไป
*   **WebAR Integration:** รองรับการแสดงผลโมเดล 3 มิติ และเทคโนโลยี AR ผ่านแท็ก `<model-viewer>` โดยไม่ต้องโหลดแอปเพิ่ม
*   **ระบบส่งงานและแสดงผลการเรียน:** คำนวณค่า Normalized Gain ตามทฤษฎีของ Hake (1998) และอัปโหลดไฟล์งานเข้าสู่ระบบคลาวด์ได้อย่างปลอดภัย

**สำหรับผู้สอน (Teachers):**
*   **Dashboard สรุปผล:** ดูความก้าวหน้า คะแนนแบบทดสอบ และบทเรียนที่ทำสำเร็จของนักเรียนทุกคน
*   **จัดการสื่อการเรียน:** อัปโหลดไฟล์เสริม (PDF, DOCX, PPTX) หรือเพิ่มลิงก์วิดีโอให้นักเรียนศึกษาเพิ่มเติม
*   **ส่งออกข้อมูล (Export):** ดาวน์โหลดคะแนนและข้อมูลทั้งหมดเป็นไฟล์ `.csv` ที่รองรับภาษาไทยสำหรับนำไปประมวลผลต่อใน Excel

## 🛠️ เทคโนโลยีที่ใช้ (Tech Stack)

*   **Frontend:** HTML5, Vanilla JavaScript, Tailwind CSS (ผ่าน CDN)
*   **AR Component:** Google `<model-viewer>`
*   **Backend / Database:** Supabase (PostgreSQL, Auth, Storage)

## 🚀 การติดตั้งและตั้งค่า (Installation & Setup)

1. **สร้างโปรเจกต์ Supabase:**
   * สมัครและสร้างโปรเจกต์ใหม่ที่ [Supabase](https://supabase.com)
   * ไปที่เมนู **Authentication** -> **Providers** -> **Email** และปิดการตั้งค่า "Confirm email"

2. **ติดตั้งฐานข้อมูล (Database Setup):**
   * คัดลอกโค้ดทั้งหมดในไฟล์ `setup.sql` 
   * นำไปวางใน **SQL Editor** ของ Supabase แล้วกด "Run"
   * ข้อมูลตาราง, นโยบายความปลอดภัย (RLS), Triggers และ Storage Buckets จะถูกสร้างโดยอัตโนมัติ

3. **นำเข้ารายชื่อนักเรียน:**
   * ไปที่เมนู **Table Editor** เลือกตาราง `students_list`
   * กด Insert หรือ Import ผ่านไฟล์ CSV เพื่อใส่รายชื่อนักเรียนที่มีสิทธิ์เข้าเรียน

4. **ตั้งค่า Frontend (HTML):**
   * ไปที่ Supabase เมนู **Project Settings** -> **API**
   * คัดลอก `Project URL` และ `anon public key`
   * เปิดไฟล์ `courseware.html` และนำค่าที่ได้ไปใส่ในตัวแปร `SUPABASE_URL` และ `SUPABASE_ANON_KEY` (ประมาณบรรทัดที่ 42-43)

5. **ตั้งค่าสิทธิ์ผู้สอน:**
   * เปิดหน้าเว็บและสมัครสมาชิกด้วยอีเมลของผู้สอน (เช่น teacher@school.ac.th)
   * ไปที่เมนู Authentication ใน Supabase แล้วคัดลอก UID ของบัญชีที่เพิ่งสมัคร
   * ไปที่ SQL Editor และรันคำสั่ง: `INSERT INTO teachers (user_id, email) VALUES ('ใส่_UID_ที่นี่', 'ใส่อีเมลที่นี่');`

## 🔒 ความปลอดภัย (Security)

*   **Row Level Security (RLS):** ข้อมูลผู้เรียนแต่ละคนถูกแยกจากกัน ผู้เรียนไม่สามารถอ่านหรือแก้ไขข้อมูลของผู้อื่นได้
*   **Database Triggers:** ป้องกันการโจมตีหรือแก้ไขคะแนนโดยผู้เรียนอย่างเด็ดขาดในระดับฐานข้อมูล
*   **Signed URLs:** ไฟล์สื่อและรูปภาพถูกเก็บใน Private Bucket ซึ่งต้องอาศัย URL ชั่วคราวในการเปิดดูเท่านั้น

## 📚 แหล่งอ้างอิงทางวิชาการในเนื้อหา
*   Azuma, R. T. (1997). A Survey of Augmented Reality.
*   Milgram, P., & Kishino, F. (1994). A Taxonomy of Mixed Reality Visual Displays.
*   Rovinelli, R. J., & Hambleton, R. K. (1977). On the Use of Content Specialists in the Assessment of Criterion-Referenced Test Item Validity.
*   Hake, R. R. (1998). Interactive-engagement versus traditional methods.