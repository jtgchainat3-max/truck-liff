# Truck Service Tracker — JTG Chainat

ระบบติดตามรถบรรทุก/ขนส่ง สำหรับบริษัทขนส่ง JTG (โกดังชัยนาท + ลพบุรี) — รองรับการบันทึกไมล์รายวัน, เติมน้ำมัน, ซ่อมบำรุง, แผนซ่อม, และเอกสารภาษี/ประกัน

## 🏗 Architecture

### Tech Stack
- **Backend**: Supabase (PostgreSQL + Edge Functions / Deno runtime)
- **Frontend**:
  - **LIFF** (`static/liff.html`) — คนขับใช้ผ่าน LINE app
  - **Admin** (`static/admin.html`) — หลังบ้านสำหรับ admin
  - **GPS Test** (`static/gps-test.html`) — utility สำหรับทดสอบ GPS
- **Storage**: Supabase Storage + Cloudflare R2 (S3-compatible) — แยกตามประเภทไฟล์
- **Hosting**: GitHub Pages (repo `jtgchainat3-max/truck-liff`)
- **LINE**: LIFF SDK + ID token verification + LINE Notify

### Edge Functions
ทุก function deploy ด้วย `--no-verify-jwt` (LIFF + admin มี auth เอง)

| Function | หน้าที่ |
|----------|--------|
| `admin-api` | หลังบ้าน — login, CRUD รถ/คนขับ/บิล/ซ่อม, รายงาน, export |
| `liff-api` | LIFF — รับส่งไมล์/น้ำมัน/ซ่อม จากคนขับผ่าน LINE |
| `_shared/` | Helper modules (security, datetime, ocr, line-notify) |

### Database Schema (Key Tables)

```
trucks                    — รถทั้งหมด
  - plate_number, province, registration_name, owner_name
  - warehouse (ชัยนาท/ลพบุรี), truck_type (pickup/6wheel/10wheel/forklift/motorcycle)
  - brand, model, year, color
  - is_active, is_pickable, is_driver_available, reserved_for_driver_id
  - last_mileage_recorded, last_mileage_at  (cache)
  - tax_due_date, insurance_due_date, compulsory_insurance_due_date

drivers                   — คนขับ
  - line_user_id, name, phone
  - driver_type: van | transport | admin
  - id_card_number (encrypted), license_number (encrypted), address
  - license_image_path, license_expiry_date
  - truck_id (FK), warehouse, route_code, is_owner, is_active

mileage_logs              — ไมล์เช้า/เย็น
  - truck_id, driver_id, log_type (morning/evening)
  - log_date, mileage, latitude, longitude
  - image_path, image_storage, route_override
  - anomaly_flag, anomaly_note

fuel_logs                 — บันทึกน้ำมัน
  - truck_id, driver_id, log_date
  - mileage, liters, price_per_liter, total_cost, total_cost_entered
  - km_since_last_fuel, km_per_liter
  - station_name, image_path (เลขไมล์), bill_image_path
  - verified_at, mileage_match, bill_received, replaces_id
  - anomaly_flag, anomaly_note
  - ocr_status, ocr_mismatches

maintenance_logs          — บันทึกซ่อม (รองรับ multi-item)
  - truck_id, driver_id, maintenance_type, maintenance_date
  - mileage_at_service, cost, labor_cost, total_cost_entered
  - shop_name, notes, tire_positions
  - service_session_id (UUID — ผูก rows ในการซ่อมครั้งเดียวกัน)
  - is_session_head (รูปเก็บแค่ head row)
  - bill_image_path, mileage_image_path
  - latitude, longitude
  - verified_by, mileage_match

maintenance_intervals     — ระยะเปลี่ยนถ่าย (PK: truck_type_group + maintenance_type)
  - truck_type_group: all | pickup | truck | forklift | motorcycle | other
  - maintenance_type: oil_change | gear_oil | differential_oil | brake_fluid
                     | coolant | tire_rotate | tire_replace | tire_patch
                     | brake | battery | wheel_bearing | wheel_rim
                     | labor | other (+ custom types ที่ admin เพิ่ม)
  - label_th, icon, interval_km, interval_months
  - alert_km_before, alert_days_before, notes

admin_users               — ผู้ใช้หลังบ้าน
  - username, password_hash, role (superadmin | admin)
  - tab_permissions (JSONB array — กำหนดแท็บที่เห็นได้)

companies                 — บริษัทเจ้าของรถ
  - name, short_name, address, tax_id

system_settings           — ตั้งค่าระบบ (key-value)
  - registration_code, ocr_enabled, fuel_efficiency_min/max ฯลฯ

audit_logs                — log การกระทำของ admin (PII redacted)
```

### Truck Type Groups (สำหรับ maintenance_intervals)

| truck_type | group | หน่วย |
|-----------|-------|------|
| pickup, 4wheel | `pickup` | กม. |
| 6wheel, 10wheel | `truck` | กม. |
| forklift | `forklift` | **ชม.** (ไม่ใช่กิโล) |
| motorcycle | `motorcycle` | กม. |
| other | fallback `all` | กม. |

Lookup logic: ใช้ group ที่ตรง → fallback to `all` group ถ้าไม่มี override

### Driver Types
- `van` — มีรถประจำ (วิ่งสาย JTG V01-V12)
- `transport` — ขนส่ง เลือกรถได้ทุกวัน
- `admin` — แอดมิน กรอกย้อนหลังได้ (เลือกวันที่เอง + skip mileage_order check + skip GPS check)

## 🚀 Deployment

### Edge Functions
```powershell
Set-Location "D:\google drive\CLAUDE\truck service tracker\truck service tracker\supabase-system"
supabase functions deploy admin-api --no-verify-jwt --project-ref joevecrppgctnemaqgan
supabase functions deploy liff-api  --no-verify-jwt --project-ref joevecrppgctnemaqgan
```

### Frontend (GitHub Pages)
```powershell
$src = "D:\google drive\CLAUDE\truck service tracker\truck service tracker\supabase-system\static"
$dst = "D:\github\truck-liff"
Copy-Item "$src\admin.html" "$dst\admin.html" -Force
Copy-Item "$src\liff.html"  "$dst\liff.html"  -Force
Set-Location $dst
git add admin.html liff.html
git commit -m "<message>"
git push origin main
```

GitHub Pages auto-deploys after push (1-2 นาที). URL: `https://jtgchainat3-max.github.io/truck-liff/`

### Database Migrations
**ไม่มี migration CLI** — paste SQL ใน Supabase Dashboard SQL Editor → Run
- URL: https://supabase.com/dashboard/project/joevecrppgctnemaqgan/sql/new
- ทุก migration อยู่ใน `supabase/migrations/` (ไฟล์ตามวันที่)

## 🔐 Security

### Implemented
- ✅ LIFF ID token verification (LINE OAuth)
- ✅ AES-GCM 256-bit encryption สำหรับ PII (id_card, license_number) — fail-loud (โยน error ถ้าไม่มี key)
- ✅ Mass assignment whitelist บน update endpoints
- ✅ XSS protection (`esc()` ทุก user-data interpolation)
- ✅ CORS exact-match (whitelist origins)
- ✅ Rate limiting (sliding window) — login 5/5min, submit 20/min
- ✅ Brute force protection — login lockout
- ✅ Signed URLs สำหรับรูปทุกใบ (Supabase + R2)
- ✅ Audit log มี PII redaction
- ✅ Role-based tab permissions (per-user override)
- ✅ `super-only` CSS class — ลบบิล/ลบไมล์/ลบ session ทำได้แค่ superadmin

### Token & Credentials
- GitHub token เก็บใน `C:\Users\i-col\.github_token` + Windows Credential Manager
- Supabase project ref: `joevecrppgctnemaqgan`
- Supabase Project: **Trucktracker** (linked already via CLI)

## 📋 Common Tasks

### เพิ่มประเภทซ่อมใหม่
1. Admin หน้า "ตั้งค่าระยะ" → กด ➕ เพิ่มประเภทใหม่
2. ระบบจะ insert เข้า `maintenance_intervals` (group=all)
3. LIFF dropdown auto-load จาก API ทันที (no code change)

### เพิ่ม Forklift-specific interval
1. หน้าตั้งค่าระยะ → dropdown เลือก "🏗 Forklift"
2. กรอกระยะ → บันทึก
3. ระบบ insert override row (truck_type_group=forklift)
4. รถ forklift จะใช้ค่านี้แทน 'all'

### เพิ่ม Driver Type ใหม่
1. Migration: ALTER TABLE drivers DROP/ADD CHECK constraint
2. liff.html: เพิ่ม `<option>` ใน regType + การ์ดประเภท
3. LIFF + admin-api: handle driver_type ใหม่ตาม use case

### Multi-item Maintenance Submission
1. LIFF ส่ง `items` = JSON array (cost, type, tire_positions, notes)
2. Backend: generate `service_session_id` UUID
3. INSERT N rows — row 0 = `is_session_head=true` (เก็บรูป + GPS), rest = `is_session_head=false`
4. Admin display: group rows by `service_session_id` → แสดงเป็น 1 session

### Baseline Logic (next_due คำนวณยังไง)
```
last_service = maintenance_logs row ล่าสุดของ (truck, type)
baseline = last_service.mileage_at_service ?? first_recorded_mileage
next_due_km = baseline + interval_km
status = overdue (dueInKm < 0) | alert (dueInKm <= alert_km_before) | ok | unknown
```

**ไม่ honor `next_service_mileage` ใน log อีกแล้ว** (LIFF เก่ามี field นี้ — ลบไปแล้ว)

## 🐛 Common Bugs & Fixes

### "ไม่เห็นรวม session ในประวัติซ่อม"
- เช็คว่า `service_session_id` มี select ใน admin-api `/maintenance` endpoint
- เช็คว่า frontend grouping logic ทำงาน (`sessionMap[sid]`)

### "next_due_km ผิด (ต่างจาก interval)"
- ดูว่ามี `next_service_mileage` ใน log เก่ามั้ย — เป็นค่าที่ LIFF เก่าบังคับให้กรอก
- Logic ใหม่ไม่ honor field นี้แล้ว — ถ้ายัง ต้องลบ override ใน code

### "Input ในตั้งค่าระยะว่าง"
- เช็ค render condition — ต้อง `group === "all" || _overridden`
- ไม่ใช้แค่ `_overridden` (เพราะ all group ไม่มี flag นี้)

### "submit-maintenance พัง: null value in image_storage"
- column `mileage_image_storage` + `bill_image_storage` ต้อง nullable
- หรือใช้ default `"supabase"` แม้ path เป็น null

### "fuel km_since_last_fuel มั่ว"
- ค่าใน DB stored at insert time — อาจผิดถ้ามี backdate
- ทาง fix: compute on-the-fly จาก data set ในหน้าหลังบ้าน (sort by mileage asc per truck, delta from previous)

## 🗂 File Structure

```
supabase-system/
├── CLAUDE.md                              ← คุณกำลังอ่านไฟล์นี้
├── supabase/
│   ├── functions/
│   │   ├── admin-api/index.ts             ← Backend หลังบ้าน
│   │   ├── liff-api/index.ts              ← Backend LIFF
│   │   └── _shared/
│   │       ├── security.ts                ← encryption, CORS, rate limit, auth
│   │       ├── datetime.ts                ← Bangkok timezone helpers
│   │       ├── ocr.ts                     ← OCR helpers (disabled — user choice)
│   │       └── line-notify.ts             ← LINE notification
│   └── migrations/
│       ├── 20240101_init.sql              ← initial schema
│       ├── 20260518_truck_enhancements.sql
│       ├── 20260520_maintenance_constraints.sql
│       ├── 20260524_truck_types.sql       ← pickup/forklift types
│       ├── 20260525_owner_and_pickable.sql
│       ├── 20260531_truck_reserved.sql
│       ├── 20260601_tab_permissions.sql
│       ├── 20260602_perf_indexes.sql
│       ├── 20260603_maintenance_intervals.sql  ← ตารางช่วงเปลี่ยน
│       ├── 20260604_admin_driver_type.sql ← admin driver type
│       └── 20260605_maintenance_v2.sql    ← session_id + labor_cost + groups
└── static/
    ├── admin.html                         ← หลังบ้าน UI
    ├── liff.html                          ← LIFF UI
    └── gps-test.html                      ← GPS debug utility
```

## 🔗 External Resources

- **Supabase Dashboard**: https://supabase.com/dashboard/project/joevecrppgctnemaqgan
- **GitHub Repo**: https://github.com/jtgchainat3-max/truck-liff
- **GitHub Pages**: https://jtgchainat3-max.github.io/truck-liff/
- **LINE Developer Console**: (LIFF + Channel ID env vars)

## 💡 Notes สำหรับ Claude ในการแก้ไขครั้งต่อๆ ไป

1. **Database changes ต้องบอก user รัน SQL ใน Supabase SQL Editor เสมอ** — ไม่มี CLI migration
2. **HTML changes ต้อง copy → push GitHub** — มี script ready ในส่วน Deployment ด้านบน
3. **Edge Function ต้อง deploy ด้วย `--no-verify-jwt`** เสมอ
4. **PowerShell quirks**:
   - ไม่มี `&&` / `||` — ใช้ `;` หรือ `if ($?) {...}`
   - cwd reset ระหว่าง commands — ใช้ `Set-Location` ทุกครั้ง
5. **`esc()` ทุก user-data interpolation** — กัน XSS
6. **Forklift truck_type ใช้ "ชม." ไม่ใช่ "กม."** — ใช้ `mileageUnit()` helper
7. **Token ของ GitHub อยู่ใน Windows Credential Manager + `~/.github_token` file** — `git push` ใช้ได้เลยไม่ต้องส่ง token
