# 🚀 คู่มือ Setup ระบบ Truck Service Tracker (สำหรับติดตั้งใหม่)

> 📌 เอกสารนี้สำหรับใครที่อยากเอาระบบนี้ไปใช้ที่บริษัทของตัวเอง
> ⏱ เวลาที่ใช้ติดตั้ง: ประมาณ **2-3 ชั่วโมง** (รวมสมัครบัญชี + ตั้งค่า)
> 💰 ค่าใช้จ่าย: **ฟรี** (Supabase free tier + GitHub Pages free + LINE LIFF free)

---

## 📋 สารบัญ
1. [สิ่งที่ต้องเตรียม](#1-สิ่งที่ต้องเตรียม)
2. [โหลด Source Code](#2-โหลด-source-code)
3. [สร้าง Supabase Project](#3-สร้าง-supabase-project)
4. [Run Database Migrations](#4-run-database-migrations)
5. [Set Environment Secrets](#5-set-environment-secrets)
6. [Deploy Edge Functions](#6-deploy-edge-functions)
7. [Setup LINE LIFF](#7-setup-line-liff)
8. [Setup GitHub Pages](#8-setup-github-pages)
9. [แก้ URL ใน HTML](#9-แก้-url-ใน-html)
10. [สร้าง Admin User คนแรก](#10-สร้าง-admin-user-คนแรก)
11. [ทดสอบระบบ](#11-ทดสอบระบบ)
12. [Optional: Cloudflare R2](#12-optional-cloudflare-r2)
13. [Customize ตามบริษัท](#13-customize-ตามบริษัท)

---

## 1. สิ่งที่ต้องเตรียม

### บัญชีที่ต้องสมัคร (ฟรีหมด)
- [ ] **Supabase** → https://supabase.com (สมัครด้วย GitHub ได้)
- [ ] **LINE Developer** → https://developers.line.biz
- [ ] **GitHub** → https://github.com
- [ ] (Optional) **Cloudflare R2** → https://cloudflare.com

### ติดตั้งในเครื่อง
- [ ] **Git** → https://git-scm.com/downloads
- [ ] **Supabase CLI** → https://supabase.com/docs/guides/cli/getting-started
  - Windows: `scoop install supabase` หรือ `winget install Supabase.CLI`
- [ ] (Optional) **Node.js** ถ้าอยากแก้ frontend ในเครื่อง

### ตรวจติดตั้งสำเร็จ
เปิด PowerShell / Terminal:
```bash
git --version
# git version 2.x.x

supabase --version
# 2.x.x
```

---

## 2. โหลด Source Code

### Option A: Clone จาก GitHub
```bash
git clone https://github.com/jtgchainat3-max/truck-liff.git
cd truck-liff
```

ในไฟล์นี้จะมี:
- `admin.html` — หน้าหลังบ้าน
- `liff.html` — หน้า LIFF (LINE)
- `gps-test.html` — utility ทดสอบ GPS
- `CLAUDE.md` — Quick reference
- `PROJECT_KNOWLEDGE.md` — Documentation ครบ
- `SETUP_GUIDE.md` — คุณกำลังอ่านไฟล์นี้

### Option B: ขอ source code เต็มจากเจ้าของระบบ
- โครงสร้างไฟล์ที่ต้องมี:
  ```
  supabase-system/
  ├── supabase/
  │   ├── config.toml
  │   ├── functions/
  │   │   ├── admin-api/index.ts
  │   │   ├── liff-api/index.ts
  │   │   └── _shared/
  │   │       ├── security.ts
  │   │       ├── datetime.ts
  │   │       ├── ocr.ts
  │   │       └── line-notify.ts
  │   └── migrations/
  │       └── *.sql
  └── static/
      ├── admin.html
      ├── liff.html
      └── gps-test.html
  ```

---

## 3. สร้าง Supabase Project

### Step 3.1: สร้าง project
1. เข้า https://supabase.com/dashboard
2. กด **New Project**
3. กรอก:
   - **Name**: `your-truck-tracker` (ตั้งเอง)
   - **Database Password**: เก็บใส่ Notepad ไว้!
   - **Region**: `Southeast Asia (Singapore)` (ใกล้ไทยที่สุด)
4. กด **Create new project** → รอ 1-2 นาที

### Step 3.2: เก็บข้อมูล project
ที่หน้า dashboard → **Project Settings** → **API**:
- เก็บ **Project URL** (เช่น `https://abcxyz.supabase.co`)
- เก็บ **service_role key** (อยู่ใน Project API keys, click reveal)
- เก็บ **Project Reference ID** (ใน URL: `https://supabase.com/dashboard/project/abcxyz` → `abcxyz`)

⚠️ **service_role key ห้ามให้ใครเห็น** — ใช้สิทธิ์เต็ม

### Step 3.3: Link CLI กับ project
ใน Terminal/PowerShell:
```bash
cd path/to/supabase-system
supabase login
# กดเปิด browser → authorize

supabase link --project-ref YOUR_PROJECT_REF
# (กรอก database password ถ้าถาม)
```

---

## 4. Run Database Migrations

### Step 4.1: รัน migrations ตามลำดับ
ที่ https://supabase.com/dashboard/project/YOUR_REF/sql/new

Copy SQL จากแต่ละไฟล์ใน `supabase/migrations/` **เรียงตามวันที่** → paste → Run:

```
20240101_init.sql                       ← Initial schema
20260518_truck_enhancements.sql
20260520_maintenance_constraints.sql
20260524_truck_types.sql
20260525_owner_and_pickable.sql
20260531_truck_reserved.sql
20260601_tab_permissions.sql
20260602_perf_indexes.sql
20260603_maintenance_intervals.sql
20260604_admin_driver_type.sql
20260605_maintenance_v2.sql
```

### Step 4.2: รัน migrations ใหม่ (ที่ยังไม่อยู่ในไฟล์)
หลังจาก migrations หลัก รัน inline migrations เพิ่ม:

```sql
-- Maintenance v3: types ใหม่ + total_cost_entered
INSERT INTO maintenance_intervals (maintenance_type, label_th, icon) VALUES
  ('wheel_bearing', 'เปลี่ยนลูกปืนล้อ', '🔩'),
  ('wheel_rim',     'เปลี่ยนกะทะล้อ',   '🛞'),
  ('tire_patch',    'ปะยาง',           '🔧'),
  ('labor',         'ค่าแรง',          '💼')
ON CONFLICT (maintenance_type) DO NOTHING;

ALTER TABLE maintenance_logs ADD COLUMN IF NOT EXISTS total_cost_entered NUMERIC;

-- Composite PK สำหรับ groups
ALTER TABLE maintenance_intervals ADD COLUMN IF NOT EXISTS truck_type_group TEXT NOT NULL DEFAULT 'all';
DO $$ BEGIN
  IF EXISTS (SELECT 1 FROM pg_constraint WHERE conname = 'maintenance_intervals_pkey') THEN
    ALTER TABLE maintenance_intervals DROP CONSTRAINT maintenance_intervals_pkey;
  END IF;
END $$;
ALTER TABLE maintenance_intervals ADD CONSTRAINT maintenance_intervals_pkey
  PRIMARY KEY (truck_type_group, maintenance_type);
CREATE INDEX IF NOT EXISTS idx_maint_intervals_type ON maintenance_intervals(maintenance_type);

-- Storage nullable
ALTER TABLE maintenance_logs
  ALTER COLUMN mileage_image_storage DROP NOT NULL,
  ALTER COLUMN bill_image_storage    DROP NOT NULL;
```

### Step 4.3: ตรวจสอบ schema
```sql
-- ดูตารางทั้งหมด
SELECT table_name FROM information_schema.tables WHERE table_schema='public' ORDER BY table_name;
```

ควรเห็น:
```
admin_users, audit_logs, companies, drivers, fuel_logs,
maintenance_intervals, maintenance_logs, mileage_logs,
system_settings, trucks
```

### Step 4.4: สร้าง Storage Buckets
1. https://supabase.com/dashboard/project/YOUR_REF/storage/buckets
2. กด **New bucket** สร้าง 3 buckets:
   - `mileage-photos` — public: **off**
   - `fuel-bills` — public: **off**
   - `driver-licenses` — public: **off**

---

## 5. Set Environment Secrets

### Step 5.1: สร้าง Encryption Key
Generate AES-GCM 256-bit key:
```bash
# PowerShell
[Convert]::ToBase64String((1..32 | ForEach-Object { Get-Random -Maximum 256 }))

# หรือ Node.js
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

Copy ค่าที่ได้ (44 ตัวอักษร base64) เก็บไว้ — เรียกว่า `PII_ENCRYPTION_KEY`

### Step 5.2: ตั้ง Secrets ใน Supabase
ที่ https://supabase.com/dashboard/project/YOUR_REF/functions/secrets:

| Key | Value | จำเป็น? |
|-----|-------|--------|
| `PII_ENCRYPTION_KEY` | (จาก Step 5.1) | ✅ |
| `LIFF_CHANNEL_ID` | (จาก LINE — ดู Step 7) | ✅ |
| `ALLOWED_ORIGINS` | `https://YOUR_GITHUB_USERNAME.github.io,https://localhost` | ✅ |
| `LINE_ADMIN_GROUP_ID` | (Optional, สำหรับ LINE notify) | ⚠️ |
| `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET` | (Optional — ใช้ Supabase Storage ก่อนได้) | ⚠️ |

หรือใช้ CLI:
```bash
supabase secrets set PII_ENCRYPTION_KEY="ค่าที่ generate มา"
supabase secrets set LIFF_CHANNEL_ID="1234567890"
supabase secrets set ALLOWED_ORIGINS="https://your-username.github.io,https://localhost"
```

---

## 6. Deploy Edge Functions

```bash
cd supabase-system
supabase functions deploy admin-api --no-verify-jwt --project-ref YOUR_PROJECT_REF
supabase functions deploy liff-api  --no-verify-jwt --project-ref YOUR_PROJECT_REF
```

ตรวจ deploy สำเร็จ:
```bash
curl https://YOUR_REF.supabase.co/functions/v1/admin-api/health
# (ถ้าไม่มี /health endpoint — ลอง /me ที่จะ return 401)
```

---

## 7. Setup LINE LIFF

### Step 7.1: สร้าง LINE Provider
1. https://developers.line.biz/console/
2. กด **Create new provider** → ตั้งชื่อบริษัท

### Step 7.2: สร้าง LINE Login Channel
1. ใน provider → กด **Create channel**
2. เลือก **LINE Login**
3. กรอก:
   - **Channel name**: ชื่อระบบ
   - **App types**: ✅ Web app
4. กด Create

### Step 7.3: เก็บ Channel ID
ที่ channel → **Basic settings** tab → เก็บ **Channel ID** (เลข ~10 หลัก)
- เอาไปใส่ใน Supabase secret `LIFF_CHANNEL_ID` (Step 5)

### Step 7.4: สร้าง LIFF App
ที่ channel → **LIFF** tab → กด **Add**:
- **LIFF app name**: `Truck Tracker`
- **Size**: Full
- **Endpoint URL**: `https://YOUR_GITHUB_USERNAME.github.io/REPO_NAME/liff.html`
  (Step 8 จะตั้ง GitHub Pages — ใช้ URL ที่จะได้)
- **Scope**: ✅ `profile` ✅ `openid`
- **Bot link feature**: optional (ถ้ามี LINE Official Account)
- กด Add

### Step 7.5: เก็บ LIFF ID
หลังสร้าง → เก็บ **LIFF ID** (รูปแบบ `1234567890-AbCdEfGh`)
- เอาไปใส่ใน `liff.html` (Step 9)

### Step 7.6: ทดสอบ LIFF URL
URL จะเป็น `https://liff.line.me/YOUR_LIFF_ID`
เปิดในมือถือผ่าน LINE → ควรเปิด LIFF (อาจจะยังไม่ทำงานเพราะ HTML ยังไม่อัพ)

---

## 8. Setup GitHub Pages

### Step 8.1: สร้าง GitHub repo
1. https://github.com/new
2. Repo name: เช่น `truck-tracker` (อะไรก็ได้)
3. **Public** (สำหรับ GitHub Pages ฟรี)
4. ✅ Add a README file
5. Create

### Step 8.2: Clone + Upload HTML
```bash
git clone https://github.com/YOUR_USERNAME/truck-tracker.git
cd truck-tracker

# Copy HTML จาก source code มา root ของ repo
cp /path/to/supabase-system/static/*.html .

git add .
git commit -m "Initial deploy"
git push origin main
```

### Step 8.3: เปิด GitHub Pages
1. ที่ repo → **Settings** → **Pages**
2. **Source**: Deploy from branch
3. **Branch**: `main` / `(root)`
4. กด **Save**
5. รอ 1-2 นาที → URL จะปรากฏ เช่น `https://your-username.github.io/truck-tracker/`

### Step 8.4: อัพเดท LIFF endpoint
กลับไป LINE Developer Console → LIFF app → แก้ **Endpoint URL**:
```
https://your-username.github.io/truck-tracker/liff.html
```

---

## 9. แก้ URL ใน HTML

### Step 9.1: แก้ `admin.html`
หา line ประมาณ 1090-1100 (ตัวแปร `API_BASE`):
```javascript
const API_BASE = "https://joevecrppgctnemaqgan.supabase.co/functions/v1/admin-api";
```
เปลี่ยนเป็น:
```javascript
const API_BASE = "https://YOUR_PROJECT_REF.supabase.co/functions/v1/admin-api";
```

### Step 9.2: แก้ `liff.html`
หา line ประมาณ 1000-1010:
```javascript
const LIFF_ID  = "2007XXXXXX-XXXXXXXX";
const API_BASE = "https://joevecrppgctnemaqgan.supabase.co/functions/v1/liff-api";
```
เปลี่ยนทั้ง 2 ตัว:
```javascript
const LIFF_ID  = "YOUR_LIFF_ID_จาก_Step_7.5";
const API_BASE = "https://YOUR_PROJECT_REF.supabase.co/functions/v1/liff-api";
```

### Step 9.3: Push อีกครั้ง
```bash
git add .
git commit -m "Configure API URLs"
git push origin main
```

---

## 10. สร้าง Admin User คนแรก

### Step 10.1: Hash password
ใช้ bcrypt online tool หรือ Node.js:
```javascript
// Node.js (npm install bcrypt)
const bcrypt = require('bcrypt');
const hash = bcrypt.hashSync('your_password_here', 10);
console.log(hash);
```

หรือใช้ online: https://bcrypt-generator.com/ (cost 10)

### Step 10.2: Insert admin user
ใน Supabase SQL Editor:
```sql
INSERT INTO admin_users (username, password_hash, role, is_active)
VALUES ('admin', 'PASTE_BCRYPT_HASH_HERE', 'superadmin', true);
```

### Step 10.3: ตั้งค่าระบบ initial
```sql
-- ตั้งรหัสลงทะเบียน (คนขับใช้สมัครใน LIFF)
INSERT INTO system_settings (key, value) VALUES ('registration_code', 'JTG2026')
ON CONFLICT (key) DO UPDATE SET value = EXCLUDED.value;

-- ค่ามาตรฐานน้ำมัน (กม./ลิตร)
INSERT INTO system_settings (key, value) VALUES
  ('fuel_efficiency_min', '3'),
  ('fuel_efficiency_max', '20'),
  ('ocr_enabled', 'false')
ON CONFLICT (key) DO UPDATE SET value = EXCLUDED.value;
```

### Step 10.4: เพิ่มบริษัทเจ้าของรถ
```sql
INSERT INTO companies (name, short_name, address, tax_id) VALUES
  ('บริษัท ของคุณ จำกัด', 'BCK', 'ที่อยู่...', '0105500000000');
```

---

## 11. ทดสอบระบบ

### Step 11.1: เข้าหน้า Admin
1. เปิด https://your-username.github.io/truck-tracker/admin.html
2. Login: username = `admin`, password = ที่ตั้งใน Step 10
3. ควรเห็น dashboard

### Step 11.2: เพิ่มรถ
1. แท็บ **🚚 รถ** → กด **➕ เพิ่มรถ**
2. กรอกข้อมูล → Save

### Step 11.3: ทดสอบ LIFF
1. เปิด LINE บนมือถือ → เปิด LIFF URL: `https://liff.line.me/YOUR_LIFF_ID`
2. ครั้งแรก = หน้าลงทะเบียน
3. กรอก:
   - **รหัสลงทะเบียน**: `JTG2026` (หรือที่ตั้งใน Step 10.3)
   - ข้อมูลส่วนตัว
   - รูปใบขับขี่
   - เลือกประเภท + รถ
4. กด Submit
5. กลับมาที่ admin → แท็บ **👤 คนขับ** → ควรเห็นคนขับใหม่

### Step 11.4: ทดสอบบันทึกไมล์
ใน LIFF → กด **📋 ส่งไมล์** → ถ่ายรูป + กรอกเลข → Submit
แล้วดูใน admin → **📋 ไมล์รายวัน**

---

## 12. Optional: Cloudflare R2

ถ้าต้องการ scale storage แยก (Supabase free มี 1GB):

### Step 12.1: สร้าง R2 Bucket
1. https://dash.cloudflare.com → R2
2. Create bucket → ตั้งชื่อ
3. Settings → CORS:
```json
[{
  "AllowedOrigins": ["*"],
  "AllowedMethods": ["GET","PUT","POST"],
  "AllowedHeaders": ["*"]
}]
```

### Step 12.2: Generate API Token
1. R2 → Manage R2 API Tokens → Create token
2. Permissions: Object Read & Write
3. Bucket: เลือก bucket ที่สร้าง
4. เก็บ:
   - Access Key ID
   - Secret Access Key
   - Endpoint URL (รูปแบบ `https://ACCOUNT_ID.r2.cloudflarestorage.com`)

### Step 12.3: ตั้งใน Supabase secrets
```bash
supabase secrets set R2_ACCOUNT_ID="..."
supabase secrets set R2_ACCESS_KEY_ID="..."
supabase secrets set R2_SECRET_ACCESS_KEY="..."
supabase secrets set R2_BUCKET="bucket-name"
```

ระบบจะ auto-upload fuel bills + driver licenses ไป R2 (mileage logs ยังอยู่ใน Supabase)

---

## 13. Customize ตามบริษัท

### 13.1: ชื่อโกดัง (Warehouse)
ค่า default ในระบบ: `ชัยนาท`, `ลพบุรี`

เปลี่ยนใน:
- `admin.html` — search `ชัยนาท` → replace ชื่อโกดังของคุณ
- `liff.html` — search `ชัยนาท` → replace
- DB:
```sql
-- ถ้ามีข้อมูลเก่า ใช้ UPDATE
UPDATE trucks  SET warehouse = 'YOUR_NEW_NAME' WHERE warehouse = 'ชัยนาท';
UPDATE drivers SET warehouse = 'YOUR_NEW_NAME' WHERE warehouse = 'ชัยนาท';
```

### 13.2: Routes (สาย VAN)
ค่า default: `V01` - `V12`

เปลี่ยนใน:
- `liff.html` — search `V01` → ดูใน `<option value="V01">`
- `liff-api/index.ts` — search `VALID_ROUTES` → แก้ array

### 13.3: ระยะการเปลี่ยนถ่ายมาตรฐาน
เข้า admin → **🛠 แผนซ่อม** → กดปุ่ม **ตั้งค่าระยะ** → แก้ตัวเลขตามต้องการ

### 13.4: ค่ามาตรฐานน้ำมัน
```sql
UPDATE system_settings SET value = '4' WHERE key = 'fuel_efficiency_min';
UPDATE system_settings SET value = '15' WHERE key = 'fuel_efficiency_max';
```

### 13.5: รหัสลงทะเบียน
admin → **⚙️ ตั้งค่าทั่วไป** → แก้ "รหัสลงทะเบียน" → ส่งให้คนขับใช้

---

## 🆘 Troubleshooting

### "Login failed"
- เช็ค password hash ถูกต้องมั้ย (re-generate ด้วย bcrypt cost 10)
- เช็ค admin_users.is_active = true
- เช็ค edge function deployed สำเร็จ + ALLOWED_ORIGINS ใส่ URL ตรงมั้ย

### "LIFF ค้างที่ Loading"
- เช็ค LIFF_ID ใน liff.html ถูกต้อง
- เช็ค LIFF Endpoint URL ใน LINE Console ตรงกับ GitHub Pages URL
- เช็ค LIFF_CHANNEL_ID ใน Supabase secret

### "Failed to fetch"
- เช็ค ALLOWED_ORIGINS ในรวม URL ของ GitHub Pages
- เช็ค Supabase Functions deployed สำเร็จ
- เช็คใน browser DevTools → Network tab → ดู error response

### "Database error: column does not exist"
- รัน migrations ขาดบางอัน — ลองรัน inline migrations ใน Step 4.2

### "Migration: relation already exists"
- ปกติ — migration ใช้ `IF NOT EXISTS` clause ที่จะข้ามถ้ามีอยู่แล้ว ปลอดภัย

---

## 📚 เอกสารเพิ่มเติม

- **PROJECT_KNOWLEDGE.md** — ข้อมูลครบทุกอย่างของระบบ
- **CLAUDE.md** — Quick reference สำหรับ Claude AI

ถ้าใช้ Claude AI ช่วยพัฒนาเพิ่ม → อัพ `PROJECT_KNOWLEDGE.md` เข้า Claude Project

---

## ✅ Checklist สรุป

- [ ] สมัคร Supabase + LINE Developer + GitHub
- [ ] ติดตั้ง Git + Supabase CLI
- [ ] Clone source code
- [ ] สร้าง Supabase project + เก็บ ref + service key
- [ ] รัน migrations ทั้งหมด + inline migrations
- [ ] สร้าง 3 Storage buckets
- [ ] Generate PII_ENCRYPTION_KEY
- [ ] ตั้ง Supabase secrets (PII_KEY, LIFF_CHANNEL_ID, ALLOWED_ORIGINS)
- [ ] Deploy Edge Functions (admin-api, liff-api)
- [ ] สร้าง LINE Channel + LIFF app
- [ ] สร้าง GitHub repo + Push HTML
- [ ] เปิด GitHub Pages
- [ ] อัพเดท Endpoint URL ใน LINE
- [ ] แก้ API_BASE + LIFF_ID ใน HTML → Push อีกครั้ง
- [ ] Hash admin password + INSERT user
- [ ] ตั้ง registration_code + บริษัท
- [ ] ทดสอบ login admin
- [ ] ทดสอบ LIFF — register driver
- [ ] ทดสอบส่งไมล์ + ดูใน admin
- [ ] (Optional) Setup Cloudflare R2
- [ ] Customize ชื่อโกดัง + routes + intervals

---

## 🎉 เสร็จแล้ว!

ระบบพร้อมใช้งานสำหรับบริษัทของคุณ — สามารถ:
- เพิ่มรถได้ไม่จำกัด
- คนขับลงทะเบียนผ่าน LINE LIFF
- บันทึกไมล์/น้ำมัน/ซ่อมประจำวัน
- ดูรายงาน + แจ้งเตือนใกล้ครบกำหนด
- ตรวจสอบบิล + ส่งบัญชี
- Export PDF/ZIP รายเดือน

**📞 ติดปัญหา?** ดู `PROJECT_KNOWLEDGE.md` → Common Bugs section
