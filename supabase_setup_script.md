-- ==========================================
-- 1. สร้างตารางและตั้งค่าพื้นฐาน
-- ==========================================
CREATE TABLE teachers (
    user_id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    email TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- รายชื่อนักเรียนที่มีสิทธิ์สมัคร (ผู้สอนนำเข้าผ่าน CSV)
CREATE TABLE students_list (
    id TEXT PRIMARY KEY CHECK (id ~ '^[0-9]{11}$'),
    title TEXT,
    first_name TEXT,
    last_name TEXT,
    section TEXT
);

CREATE TABLE students (
    id TEXT PRIMARY KEY REFERENCES students_list(id),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE UNIQUE,
    photo_path TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE attempts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
    kind TEXT CHECK (kind IN ('pre', 'post')),
    score INTEGER NOT NULL,
    answers JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, kind)
);

CREATE TABLE progress (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
    unit INTEGER NOT NULL,
    done BOOLEAN DEFAULT FALSE,
    UNIQUE(user_id, unit)
);

CREATE TABLE submissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
    unit INTEGER NOT NULL,
    kind TEXT CHECK (kind IN ('link', 'file')),
    url TEXT,
    storage_path TEXT,
    note TEXT,
    score INTEGER NULL,
    teacher_comment TEXT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    teacher_id UUID REFERENCES teachers(user_id) ON DELETE SET NULL,
    title TEXT NOT NULL,
    kind TEXT CHECK (kind IN ('file', 'link', 'video')),
    url TEXT,
    storage_path TEXT,
    file_name TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ==========================================
-- 2. ฟังก์ชันและ Triggers
-- ==========================================

-- ฟังก์ชันตรวจสอบว่าเป็นผู้สอนหรือไม่ (Security Definer เพื่อข้าม RLS ชั่วคราว)
CREATE OR REPLACE FUNCTION is_teacher()
RETURNS BOOLEAN
SET search_path = ''
AS $$
BEGIN
  RETURN EXISTS (SELECT 1 FROM public.teachers WHERE user_id = auth.uid());
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Trigger สมัครสมาชิกเฉพาะผู้มีรายชื่อ
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
DECLARE
    v_student_id TEXT;
BEGIN
    IF NEW.email ~ '^[0-9]{11}@' THEN
        v_student_id := split_part(NEW.email, '@', 1);
        IF NOT EXISTS (SELECT 1 FROM public.students_list WHERE id = v_student_id) THEN
            RAISE EXCEPTION 'รหัสนักศึกษาไม่มีในระบบ';
        END IF;
        IF EXISTS (SELECT 1 FROM public.students WHERE id = v_student_id) THEN
            RAISE EXCEPTION 'รหัสนักศึกษานี้ลงทะเบียนแล้ว';
        END IF;
        INSERT INTO public.students (id, user_id) VALUES (v_student_id, NEW.id);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
AFTER INSERT ON auth.users
FOR EACH ROW EXECUTE FUNCTION handle_new_user();

-- Trigger ป้องกันผู้เรียนแก้คะแนนและคอมเมนต์ใน submissions
-- อธิบาย: หากไม่ใช่ผู้สอน อัปเดตข้อมูลให้ค่า score และ comment คงเดิมเสมอ (Ignore การพยายามแก้)
CREATE OR REPLACE FUNCTION protect_submission_scores()
RETURNS TRIGGER AS $$
BEGIN
    IF NOT is_teacher() THEN
        NEW.score = OLD.score;
        NEW.teacher_comment = OLD.teacher_comment;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER enforce_submission_protection
BEFORE UPDATE ON submissions
FOR EACH ROW EXECUTE FUNCTION protect_submission_scores();

-- ==========================================
-- 3. Row Level Security (RLS)
-- ==========================================
ALTER TABLE teachers ENABLE ROW LEVEL SECURITY;
ALTER TABLE students_list ENABLE ROW LEVEL SECURITY;
ALTER TABLE students ENABLE ROW LEVEL SECURITY;
ALTER TABLE attempts ENABLE ROW LEVEL SECURITY;
ALTER TABLE progress ENABLE ROW LEVEL SECURITY;
ALTER TABLE submissions ENABLE ROW LEVEL SECURITY;
ALTER TABLE materials ENABLE ROW LEVEL SECURITY;

REVOKE ALL ON teachers, students_list, students, attempts, progress, submissions, materials FROM anon;

-- Teachers
CREATE POLICY "Teacher read all" ON teachers FOR SELECT TO authenticated USING (true);
-- Students List
CREATE POLICY "Teacher read list" ON students_list FOR SELECT TO authenticated USING (is_teacher());
-- Students
CREATE POLICY "Student read own, Teacher read all" ON students FOR SELECT TO authenticated USING (user_id = auth.uid() OR is_teacher());
CREATE POLICY "Student update own photo" ON students FOR UPDATE TO authenticated USING (user_id = auth.uid()) WITH CHECK (user_id = auth.uid());
-- Attempts
CREATE POLICY "Student read own attempts, Teacher read all" ON attempts FOR SELECT TO authenticated USING (user_id = auth.uid() OR is_teacher());
CREATE POLICY "Student insert own attempts" ON attempts FOR INSERT TO authenticated WITH CHECK (user_id = auth.uid());
-- Progress
CREATE POLICY "Student read own progress, Teacher read all" ON progress FOR SELECT TO authenticated USING (user_id = auth.uid() OR is_teacher());
CREATE POLICY "Student insert/update own progress" ON progress FOR ALL TO authenticated USING (user_id = auth.uid()) WITH CHECK (user_id = auth.uid());
-- Submissions
CREATE POLICY "Student read own submissions, Teacher read all" ON submissions FOR SELECT TO authenticated USING (user_id = auth.uid() OR is_teacher());
CREATE POLICY "Student insert own submissions" ON submissions FOR INSERT TO authenticated WITH CHECK (user_id = auth.uid());
CREATE POLICY "Student update own submissions, Teacher update all" ON submissions FOR UPDATE TO authenticated USING (user_id = auth.uid() OR is_teacher());
-- Materials
CREATE POLICY "All read materials" ON materials FOR SELECT TO authenticated USING (true);
CREATE POLICY "Teacher manage materials" ON materials FOR ALL TO authenticated USING (is_teacher()) WITH CHECK (is_teacher());

-- ==========================================
-- 4. View สำหรับผู้สอน (Security Invoker)
-- ==========================================
CREATE OR REPLACE VIEW results WITH (security_invoker = true) AS
SELECT 
    sl.id AS student_id,
    sl.title, sl.first_name, sl.last_name, sl.section,
    s.photo_path,
    (SELECT score FROM attempts WHERE user_id = s.user_id AND kind = 'pre' LIMIT 1) AS pre_score,
    (SELECT score FROM attempts WHERE user_id = s.user_id AND kind = 'post' LIMIT 1) AS post_score,
    (SELECT COUNT(*) FROM progress WHERE user_id = s.user_id AND done = true) AS units_done
FROM students_list sl
LEFT JOIN students s ON sl.id = s.id;

-- ==========================================
-- 5. Storage Buckets & Policies
-- ==========================================
INSERT INTO storage.buckets (id, name, public) VALUES ('photos', 'photos', false);
INSERT INTO storage.buckets (id, name, public) VALUES ('submissions', 'submissions', false);
INSERT INTO storage.buckets (id, name, public) VALUES ('materials', 'materials', false);

-- Photos Policy
CREATE POLICY "Users can manage own photos folder" ON storage.objects FOR ALL TO authenticated USING (bucket_id = 'photos' AND (storage.foldername(name))[1] = auth.uid()::text) WITH CHECK (bucket_id = 'photos' AND (storage.foldername(name))[1] = auth.uid()::text);
CREATE POLICY "Teachers can view all photos" ON storage.objects FOR SELECT TO authenticated USING (bucket_id = 'photos' AND is_teacher());

-- Submissions Policy
CREATE POLICY "Users can manage own submissions" ON storage.objects FOR ALL TO authenticated USING (bucket_id = 'submissions' AND (storage.foldername(name))[1] = auth.uid()::text) WITH CHECK (bucket_id = 'submissions' AND (storage.foldername(name))[1] = auth.uid()::text);
CREATE POLICY "Teachers can view all submissions" ON storage.objects FOR SELECT TO authenticated USING (bucket_id = 'submissions' AND is_teacher());

-- Materials Policy
CREATE POLICY "Anyone can view materials" ON storage.objects FOR SELECT TO authenticated USING (bucket_id = 'materials');
CREATE POLICY "Teachers can manage materials" ON storage.objects FOR ALL TO authenticated USING (bucket_id = 'materials' AND is_teacher()) WITH CHECK (bucket_id = 'materials' AND is_teacher());

-- ==========================================
-- 6. ข้อมูลจำลอง (Mock Data)
-- ==========================================
-- รายชื่อผู้เรียน (ใช้ Table Editor นำเข้า CSV เพิ่มเติมได้)
INSERT INTO students_list (id, title, first_name, last_name, section) VALUES
('11239868001', 'ด.ช.', 'สมชาย', 'ใจดี', '5/1'),
('11239868002', 'ด.ญ.', 'สมหญิง', 'รักเรียน', '5/1'),
('11239868003', 'ด.ช.', 'กล้าหาญ', 'ชาญชัย', '5/1'),
('11239868004', 'ด.ญ.', 'ใบบัว', 'บานชื่น', '5/2'),
('11239868005', 'ด.ช.', 'ตะวัน', 'ฉายแสง', '5/2'),
('11239868006', 'ด.ญ.', 'ดวงดาว', 'พราวฟ้า', '5/2');

-- คำสั่งตั้งผู้สอน (รันคำสั่งนี้หลังจากที่ผู้สอนสมัครสมาชิกผ่านหน้าเว็บแล้ว)
-- นำคอมเมนต์ออกและเปลี่ยน 'teacher_uid_here' เป็น user_id ของผู้สอน
-- INSERT INTO teachers (user_id, email) VALUES ('teacher_uid_here', 'teacher@nuhin.ac.th');
