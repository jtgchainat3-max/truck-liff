# 🚛 Truck Service Tracker — Full Project Knowledge

**สำหรับอัพโหลดเข้า Claude Project** เพื่อให้ Claude session ใหม่เข้าใจระบบทั้งหมด

> 📅 อัพเดทล่าสุด: 2026-05-20
> 🏢 บริษัท: JTG Chainat (โกดังชัยนาท + ลพบุรี)
> 🌐 Live URLs:
> - **Admin**: https://jtgchainat3-max.github.io/truck-liff/admin.html
> - **LIFF**: https://jtgchainat3-max.github.io/truck-liff/liff.html
> - **Supabase**: https://supabase.com/dashboard/project/joevecrppgctnemaqgan
> - **GitHub Repo**: https://github.com/jtgchainat3-max/truck-liff

---

## 📋 1. ภาพรวมระบบ

### Use Case
บริษัทขนส่ง JTG ใช้ระบบนี้เพื่อ:
1. **คนขับ** บันทึกไมล์เช้า-เย็น ผ่าน LINE LIFF (ถ่ายรูปเลขไมล์ + GPS)
2. **คนขับ** บันทึกการเติมน้ำมัน (รูปบิล + รูปเลขไมล์)
3. **คนขับ** บันทึกการซ่อมบำรุง (รูปบิล + รูปเลขไมล์ + ตำแหน่งล้อ)
4. **แอดมิน** ดูสรุปไมล์รายวัน, น้ำมัน, ซ่อม, แผนซ่อม, เอกสารภาษี/ประกัน
5. **แอดมิน** ตรวจสอบบิล + ส่งบัญชี (export PDF/ZIP รายเดือน)
6. **แอดมิน** ตั้งระยะเปลี่ยนถ่ายต่อประเภทรถ → ระบบแจ้งเตือนใกล้ครบกำหนด

### Roles
- **superadmin** — ทำทุกอย่าง รวม delete, manage users, export
- **admin** — ทำได้ตาม tab_permissions ที่ superadmin กำหนด
- **driver** (LIFF only, ไม่ login admin) — บันทึกข้อมูลของตัวเอง

### Driver Types
- **van** — VAN มีรถประจำ วิ่งสาย JTG V01-V12
- **transport** — ขนส่ง เลือกรถได้ทุกวัน
- **admin** — แอดมิน กรอกย้อนหลังได้ (เลือกวันที่เอง, skip GPS check, skip mileage order check)

---

## 🏗 2. Tech Stack

| Layer | Technology |
|-------|-----------|
| Database | Supabase PostgreSQL |
| Backend | Supabase Edge Functions (Deno runtime, TypeScript) |
| Frontend | Static HTML + vanilla JS (no framework) |
| Hosting | GitHub Pages (HTML), Supabase (Functions) |
| Image Storage | Supabase Storage + Cloudflare R2 (S3-compatible) |
| Auth (LIFF) | LINE LIFF SDK + LINE ID Token verification (OAuth) |
| Auth (Admin) | Username/password + bcrypt + session token |
| LINE Notify | LINE Messaging API |

### Why no framework?
- ระบบเล็ก ไม่ต้อง build step
- HTML แก้แล้ว push ตรง → GitHub Pages deploy ทันที 1-2 นาที
- ลด complexity

### Key Constraints
- ห้ามใช้ external CDN libs (ยกเว้น LINE LIFF SDK + JSZip)
- Bangkok timezone (Asia/Bangkok, UTC+7) — มี helper `todayBKK()` ทุก function
- ทุก user input ต้อง `esc()` ก่อน interpolate (กัน XSS)
- รูปทุกใบเก็บเป็น **signed URL** ที่หมดอายุ — ไม่ public

---

## 🗄 3. Database Schema

### Tables (สำคัญ)

#### `trucks`
```sql
id                    SERIAL PRIMARY KEY
plate_number          TEXT NOT NULL
province              VARCHAR(50)
registration_name     TEXT  -- ชื่อในเล่มทะเบียน
owner_name            TEXT  -- ชื่อผู้ครอบครอง
warehouse             TEXT  -- 'ชัยนาท' | 'ลพบุรี'
truck_type            TEXT  -- 'pickup' | '6wheel' | '10wheel' | 'forklift' | 'motorcycle' | 'other'
brand                 TEXT
model                 TEXT
year                  INTEGER
color                 VARCHAR(20)
is_active             BOOLEAN DEFAULT true
is_pickable           BOOLEAN DEFAULT true   -- แสดงใน driver picker?
is_driver_available   BOOLEAN DEFAULT true   -- รับ driver ใหม่ได้?
reserved_for_driver_id INTEGER REFERENCES drivers(id)  -- รถประจำตัว
company_id            INTEGER REFERENCES companies(id)
tax_due_date          DATE
insurance_due_date    DATE
compulsory_insurance_due_date DATE
last_mileage_recorded INTEGER  -- cache สำหรับ schedule (auto-update)
last_mileage_at       TIMESTAMPTZ
notes                 TEXT
```

#### `drivers`
```sql
id                  SERIAL PRIMARY KEY
line_user_id        TEXT UNIQUE NOT NULL  -- จาก LINE LIFF
name                TEXT NOT NULL
phone               TEXT
driver_type         TEXT CHECK (driver_type IN ('van','transport','admin'))
route_code          TEXT  -- 'V01'...'V12' สำหรับ van
truck_id            INTEGER REFERENCES trucks(id)
warehouse           TEXT
id_card_number      TEXT  -- AES-GCM encrypted
address             TEXT
license_number      TEXT  -- AES-GCM encrypted
license_image_path  TEXT  -- supabase storage path
license_image_storage TEXT  -- 'supabase' | 'r2'
license_expiry_date DATE
is_owner            BOOLEAN DEFAULT false
is_active           BOOLEAN DEFAULT true
```

#### `mileage_logs` (ไมล์เช้า/เย็น)
```sql
id              SERIAL PRIMARY KEY
truck_id        INTEGER REFERENCES trucks(id)
driver_id       INTEGER REFERENCES drivers(id)
log_type        TEXT  -- 'morning' | 'evening'
log_date        DATE
mileage         INTEGER
latitude        NUMERIC
longitude       NUMERIC
accuracy        NUMERIC
image_path      TEXT
image_storage   TEXT
route_override  TEXT  -- ถ้า VAN ขับแทนสายอื่น
anomaly_flag    BOOLEAN
anomaly_note    TEXT
created_at      TIMESTAMPTZ DEFAULT NOW()
```

#### `fuel_logs` (เติมน้ำมัน)
```sql
id                    SERIAL PRIMARY KEY
truck_id              INTEGER REFERENCES trucks(id)
driver_id             INTEGER REFERENCES drivers(id)
log_date              DATE
mileage               INTEGER
liters                NUMERIC(20,10)  -- precision สูง (10 ตำแหน่ง)
price_per_liter       NUMERIC
total_cost            NUMERIC  -- calculated หรือ entered
total_cost_entered    INTEGER  -- ยอดที่คนขับกดจริง (integer, ไม่มีทศนิยม)
km_since_last_fuel    INTEGER  -- delta จาก last fuel
km_per_liter          NUMERIC  -- = km_since_last / liters
station_name          TEXT
latitude, longitude   NUMERIC
image_path            TEXT  -- รูปเลขไมล์
image_storage         TEXT
bill_image_path       TEXT  -- รูปบิล
bill_image_storage    TEXT
verified_at           TIMESTAMPTZ
verified_by           TEXT
mileage_match         BOOLEAN  -- ตรวจ: ไมล์ตรงบิลมั้ย
bill_received         BOOLEAN
bill_received_at      TIMESTAMPTZ
ocr_status            TEXT  -- 'pending' | 'matched' | 'mismatch' | 'failed' (currently DISABLED)
ocr_mismatches        JSONB
replaces_id           INTEGER  -- ถ้าส่งบิลใหม่แทนใบเดิม
superseded_by_id      INTEGER
admin_override_by     TEXT
anomaly_flag          BOOLEAN
anomaly_note          TEXT
created_at            TIMESTAMPTZ
```

#### `maintenance_logs` (ซ่อมบำรุง — รองรับ multi-item per session)
```sql
id                    SERIAL PRIMARY KEY
truck_id              INTEGER
driver_id             INTEGER
maintenance_type      TEXT  -- ดูใน maintenance_intervals
maintenance_date      DATE
mileage_at_service    INTEGER
cost                  NUMERIC  -- ค่าใช้จ่ายของ item นี้
labor_cost            NUMERIC  -- DEPRECATED — ไม่ใช้แล้ว (labor เป็น item แยก)
total_cost_entered    NUMERIC  -- เก็บแค่ head row — ยอดในบิลจริง
shop_name             TEXT
notes                 TEXT
tire_positions        TEXT[]  -- ['FL','FR','RL','RR'] สำหรับงานล้อ
service_session_id    UUID    -- ผูก rows ในการเข้าอู่ครั้งเดียวกัน
is_session_head       BOOLEAN DEFAULT true  -- รูป + GPS เก็บแค่ head row
bill_image_path       TEXT
bill_image_storage    TEXT
mileage_image_path    TEXT
mileage_image_storage TEXT
latitude, longitude   NUMERIC
next_service_mileage  INTEGER  -- DEPRECATED — ไม่ใช้แล้ว (compute จาก interval)
next_service_date     DATE     -- DEPRECATED
verified_by           TEXT
verified_at           TIMESTAMPTZ
mileage_match         BOOLEAN
verification_note     TEXT
bill_received         BOOLEAN
bill_received_at      TIMESTAMPTZ
created_at            TIMESTAMPTZ
```

**Multi-item Pattern:**
- 1 ครั้งเข้าอู่ = 1 `service_session_id` (UUID)
- N items = N rows ใน table
- Row แรก (is_session_head=true) เก็บรูปทั้งหมด + GPS
- Row อื่น (is_session_head=false) เก็บแค่ type + cost + notes + tire_positions

#### `maintenance_intervals` (ตารางช่วงเปลี่ยน — PK composite)
```sql
truck_type_group     TEXT NOT NULL DEFAULT 'all'
                     -- 'all' | 'pickup' | 'truck' | 'forklift' | 'motorcycle' | 'other'
maintenance_type     TEXT NOT NULL  -- 'oil_change' | 'gear_oil' | ฯลฯ
label_th             TEXT NOT NULL
icon                 TEXT DEFAULT '🔧'  -- emoji
interval_km          INTEGER  -- ทุก N กม. (หรือ ชม. ถ้า forklift)
interval_months      INTEGER  -- ทุก N เดือน
alert_km_before      INTEGER DEFAULT 500
alert_days_before    INTEGER DEFAULT 14
notes                TEXT
updated_at           TIMESTAMPTZ
PRIMARY KEY (truck_type_group, maintenance_type)
```

**Standard maintenance_types:**
- `oil_change` — เปลี่ยนน้ำมันเครื่อง 🛢 (10000 กม. / 6 เดือน)
- `gear_oil` — น้ำมันเกียร์ ⚙️ (80000 / 24)
- `differential_oil` — เฟืองท้าย 🔩 (60000 / 24)
- `brake_fluid` — น้ำมันเบรค 💧 (40000 / 24)
- `coolant` — น้ำหม้อน้ำ 🌡 (ตั้งเอง)
- `tire_rotate` — สลับยาง 🔄 (10000 / 6)
- `tire_replace` — เปลี่ยนยาง 🛞 (80000 / 60)
- `tire_patch` — ปะยาง 🔧 (no interval)
- `brake` — เปลี่ยน/ตรวจเบรก 🛑 (40000 / 12)
- `battery` — แบตเตอรี่ 🔋 (no km / 24)
- `wheel_bearing` — ลูกปืนล้อ 🔩 (no interval)
- `wheel_rim` — กะทะล้อ 🛞 (no interval)
- `labor` — ค่าแรง 💼 (special — no schedule)
- `other` — อื่นๆ 🔧 (มีช่อง "ซ่อมอะไร")

**Truck type → group mapping:**
```javascript
pickup, 4wheel  → 'pickup'
6wheel, 10wheel → 'truck'    // รถบรรทุก
forklift        → 'forklift' // ใช้ "ชม." แทน "กม."
motorcycle      → 'motorcycle'
other           → 'all' (fallback)
```

**Lookup logic:**
- ลอง group ที่ตรง → ถ้าไม่มี override → fallback ใช้ `all`

#### `admin_users`
```sql
id              SERIAL PRIMARY KEY
username        TEXT UNIQUE
password_hash   TEXT  -- bcrypt
role            TEXT  -- 'superadmin' | 'admin'
tab_permissions JSONB  -- ['summary','daily','fuel',...] หรือ null = ใช้ role default
is_active       BOOLEAN
created_at      TIMESTAMPTZ
last_login_at   TIMESTAMPTZ
```

#### `companies`, `system_settings`, `audit_logs`
- `companies` — บริษัทเจ้าของรถ (name, short_name, address, tax_id)
- `system_settings` — key-value (registration_code, ocr_enabled, fuel_efficiency_min/max)
- `audit_logs` — log การกระทำของ admin (PII auto-redacted)

---

## 🔌 4. API Endpoints

### admin-api (require X-Auth header)

#### Auth & User
- `POST /login` — username + password → return token
- `GET /me` — current user info + role + tab_permissions
- `POST /logout`

#### Trucks
- `GET /trucks` — รายการรถทั้งหมด
- `POST /add-truck` — เพิ่มรถ
- `PUT /update-truck` — แก้ไขรถ (mass-assignment whitelist)

#### Drivers
- `GET /drivers` — รายการคนขับ
- `PUT /drivers/:id` — แก้คนขับ (whitelist)
- `POST /update-driver` — alias
- `DELETE /drivers/:id` — ลบ + cleanup license image (superadmin)

#### Mileage / Fuel / Maintenance
- `GET /daily?date=YYYY-MM-DD` — ไมล์รายวัน (เช้า/เย็น)
- `PUT /mileage-logs/:id` — แก้ไมล์
- `DELETE /mileage-logs/:id` — superadmin only
- `GET /fuel-summary?date_from&date_to&truck_id` — สรุปน้ำมัน
- `PUT /fuel-logs/:id/verify` — ตรวจสอบบิล
- `DELETE /fuel-logs/:id` — superadmin only
- `GET /maintenance?truck_id&type` — ประวัติซ่อม
- `DELETE /maintenance-logs/:id` — superadmin only

#### Maintenance Intervals & Schedule
- `GET /maintenance-intervals?group=<group>` — ตารางช่วงเปลี่ยน
- `POST /maintenance-intervals` — เพิ่มประเภทใหม่ (superadmin)
- `PUT /maintenance-intervals/:type?group=<group>` — แก้/upsert
- `DELETE /maintenance-intervals/:type?group=<group>` — ลบ
- `GET /maintenance-schedule` — สถานะแผนซ่อมต่อรถ

#### Verify / Accounting
- `GET /verify-queue?truck_id` — บิลรอตรวจ
- `PUT /fuel/:id/verify-mileage` — verify ไมล์ในบิล
- `GET /accounting?date_from&date_to&type` — บิลส่งบัญชี
- `GET /export-bills?year&month&type` — รายงานเดือน

#### Documents (ภาษี/ประกัน)
- `GET /documents-status` — รายการเอกสารใกล้หมด

#### Anomalies
- `GET /anomalies?days` — ผิดปกติ
- `PUT /:table-logs/:id/resolve-anomaly` — ล้าง flag

#### Companies
- `GET /companies`, `POST/PUT/DELETE /companies`

#### Admin Users
- `GET /admin-users`, `POST /admin-users`
- `PUT /admin-users/:id/permissions` — เปลี่ยน tab_permissions
- `DELETE /admin-users/:id` (superadmin only)

#### Settings
- `GET /settings`, `PUT /settings`

#### Summary
- `GET /summary` — dashboard numbers

### liff-api (no auth — uses LIFF id_token + line_user_id)

- `POST /register` — ลงทะเบียนคนขับ (multipart, รูปใบขับขี่)
- `GET /driver-status?line_user_id=` — เช็คสถานะ + ข้อมูลคนขับ
- `GET /smart-match?line_user_id=` — แนะนำรถที่ควรขับ
- `POST /submit-mileage` — ส่งไมล์ (multipart, รูปเลขไมล์)
- `POST /submit-fuel` — ส่งบิลน้ำมัน (multipart, 2 รูป: เลขไมล์+บิล)
- `POST /submit-maintenance` — ส่งซ่อม (multipart, items=JSON array)
- `GET /maintenance-types` — รายการประเภทซ่อม (สำหรับ dropdown)

---

## 🚀 5. Deployment Workflow

### A. Database Migration (รัน SQL ใน Supabase Dashboard)

ไม่มี CLI migration — paste SQL ใน:
https://supabase.com/dashboard/project/joevecrppgctnemaqgan/sql/new

Migrations ทั้งหมดอยู่ใน `supabase/migrations/` เรียงตามวันที่

### B. Deploy Edge Functions

```powershell
Set-Location "D:\google drive\CLAUDE\truck service tracker\truck service tracker\supabase-system"
supabase functions deploy admin-api --no-verify-jwt --project-ref joevecrppgctnemaqgan
supabase functions deploy liff-api  --no-verify-jwt --project-ref joevecrppgctnemaqgan
```

**⚠️ ใช้ `--no-verify-jwt` เสมอ** เพราะ LIFF + admin มี auth เอง (ไม่ใช่ Supabase JWT)

### C. Push HTML to GitHub Pages

```powershell
$src = "D:\google drive\CLAUDE\truck service tracker\truck service tracker\supabase-system\static"
$dst = "D:\github\truck-liff"
Copy-Item "$src\admin.html" "$dst\admin.html" -Force
Copy-Item "$src\liff.html"  "$dst\liff.html"  -Force
Set-Location $dst
git add admin.html liff.html
git commit -m "your message"
git push origin main
```

GitHub Pages auto-deploys 1-2 นาทีหลัง push

---

## 🔐 6. Credentials & Tokens

### GitHub Token
- เก็บใน: `C:\Users\i-col\.github_token` (file)
- + Windows Credential Manager (target: `git:https://github.com`, user: `jtgchainat3-max`)
- Token ใช้ scope `repo` — Generate ใหม่ที่ https://github.com/settings/tokens

### Supabase
- Project ref: `joevecrppgctnemaqgan`
- ชื่อ project: `Trucktracker`
- Linked แล้วผ่าน `supabase link` — ไม่ต้อง link อีก
- Service role key เก็บใน Edge Function secrets

### LINE
- LIFF Channel ID เก็บใน Edge Function env: `LIFF_CHANNEL_ID`
- LINE Notify token (ถ้าใช้) เก็บใน Edge Function env

### Encryption (PII)
- AES-GCM 256-bit key เก็บใน Edge Function env: `PII_ENCRYPTION_KEY`
- **Fail-loud** — โยน error ถ้าไม่มี key (ไม่ fallback plaintext)

### Cloudflare R2
- Bucket: เก็บรูปบิล + รูปใบขับขี่ (high-value images)
- Supabase Storage: เก็บรูปไมล์รายวัน (disposable — auto-cleanup 3 เดือน)
- AWS4 signing แบบ manual ใน security.ts

---

## 🐛 7. Common Bugs & Patterns

### Bug 1: "ไม่เห็นรวม session"
**สาเหตุ:** admin-api `/maintenance` ไม่ select `service_session_id` field
**Fix:** เพิ่ม field ใน `.select()` clause

### Bug 2: "next_due_km ผิด"
**สาเหตุ:** Logic เก่า honor `next_service_mileage` ใน log (LIFF เก่าให้กรอก)
**Fix:** ลบ override → ใช้ baseline + interval_km เท่านั้น

### Bug 3: "Input ในตั้งค่าระยะว่าง"
**สาเหตุ:** เช็ค `_overridden` แทน `group === 'all'`
**Fix:** ใช้ `const showValue = group === "all" || _overridden`

### Bug 4: "submit-maintenance: null value in image_storage"
**สาเหตุ:** Non-head rows มี image_path=null แต่ column NOT NULL
**Fix:** ALTER TABLE DROP NOT NULL หรือ default `"supabase"`

### Bug 5: "fuel km_since_last_fuel มั่ว"
**สาเหตุ:** DB value ผิดจาก backdate insert
**Fix:** Compute on-the-fly ใน admin display (sort by mileage asc per truck)

### Bug 6: "PowerShell cwd reset"
**สาเหตุ:** Tool wrapper reset cwd ทุก command
**Fix:** ใช้ `Set-Location` ทุก command — ห้าม pipe เป็น chain

### Bug 7: "Forklift แสดง 'กม.' แทน 'ชม.'"
**สาเหตุ:** Hardcode "กม." ใน label
**Fix:** ใช้ `mileageUnit(truck_type)` helper

### Bug 8: "Login brute force"
**สาเหตุ:** ไม่มี rate limit
**Fix:** `checkRateLimit(supabase, 'login:${ip}:${username}', 5, 300)` — 5 ครั้ง/5 นาที

---

## 📋 8. UI Tabs (Admin)

### Main menu structure (รวมเป็น 9 ตัว — มี submenu)
```
📊 ภาพรวม
📋 ไมล์รายวัน
⛽ น้ำมัน
🔧 ซ่อมบำรุง ▾
  ├ 🔧 ประวัติซ่อม
  └ 🛠 แผนซ่อม
🔍 ตรวจสอบบิล ▾
  ├ 🔍 ตรวจสอบบิล
  └ 📒 ส่งบัญชี
⚠️ ผิดปกติ
📋 ภาษี/ประกัน
📁 ข้อมูล ▾
  ├ 🚛 รถ
  └ 👤 คนขับ
⚙️ ตั้งค่า ▾
  ├ 🔑 ผู้ใช้ระบบ
  ├ 🏢 บริษัท
  └ ⚙️ ตั้งค่าทั่วไป
```

### Tab IDs (data-tab)
- `summary, daily, fuel, maintenance, maintsched, verify, accounting, anomalies, documents, trucks, drivers, adminusers, companies, settings`

### Permission System
- Default per role: `DEFAULT_TABS_BY_ROLE[role]`
- Custom override per user: `admin_users.tab_permissions` (JSONB array)

---

## 🎨 9. LIFF Screens

```
screenLoading        — กำลังโหลด
screenBlocked        — บล็อก (เปิดผ่าน LINE เท่านั้น)
screenRegister       — ลงทะเบียนคนขับ (รูปใบขับขี่)
screenMenu           — เมนูหลัก
screenMileage        — บันทึกไมล์ (เช้า/เย็น)
screenFuel           — เติมน้ำมัน
screenMaintenance    — งานซ่อม (multi-item form)
screenDriver         — ข้อมูลคนขับ + ปุ่มสนับสนุน
```

### Maintenance Multi-Item Pattern
- Shared: เลขไมล์ + ชื่ออู่ + รูปเลขไมล์ + รูปบิล + GPS
- Per item: type + cost + tire_positions + notes
- ปุ่ม "➕ เพิ่มรายการซ่อม" → clone item template
- Submit: items = JSON array → POST submit-maintenance
- "ค่าแรง" เป็น type แยก (type=labor)

### Admin Backdate
- ถ้า `driver_type === "admin"` → แสดงการ์ดสีม่วง "📅 วันที่ของบันทึก"
- input type=date → ส่ง `log_date` ใน FormData
- Backend: ถ้า driver เป็น admin + log_date valid → ใช้แทน todayBKK()
- + skip GPS check + skip mileage order check

---

## 🔧 10. Common Code Patterns

### Bangkok Timezone Helper
```typescript
// supabase/functions/_shared/datetime.ts
import { todayBKK, nowBKK, timeBKK, isPhotoFresh } from "../_shared/datetime.ts";

todayBKK()  // "2026-05-20"
nowBKK()    // "2026-05-20T15:30:45+07:00"
timeBKK(ts) // "15:30"
isPhotoFresh(ts, 10)  // boolean — รูปไม่เก่ากว่า 10 นาที
```

### XSS Protection (Frontend)
```javascript
function esc(s) {
  if (s == null) return "";
  return String(s).replace(/[&<>"']/g, c => ({
    "&":"&amp;","<":"&lt;",">":"&gt;","\"":"&quot;","'":"&#39;"
  }[c]));
}
// Use: ${esc(r.driver_name)} not ${r.driver_name}
```

### Mileage Unit Helper
```javascript
function mileageUnit(truckType) {
  return truckType === "forklift" ? "ชม." : "กม.";
}
function mileageUnitLong(truckType) {
  return truckType === "forklift" ? "ชั่วโมง" : "กิโลเมตร";
}
```

### Multi-item Maintenance Submit (LIFF)
```javascript
const items = [];
for (const item of document.querySelectorAll(".maint-item")) {
  items.push({
    type: item.querySelector(".m-type").value,
    cost: parseFloat(item.querySelector(".m-cost").value) || null,
    tire_positions: [...item.querySelectorAll(".tire-btn.selected")].map(b => b.dataset.code),
    notes: item.querySelector(".m-notes").value || null,
  });
}
fd.append("items", JSON.stringify(items));
```

### Maintenance Schedule Lookup (Backend)
```typescript
const TRUCK_TYPE_TO_GROUP = {
  pickup: "pickup", "4wheel": "pickup",
  "6wheel": "truck", "10wheel": "truck",
  forklift: "forklift",
  motorcycle: "motorcycle",
  other: "other",
};
function getIntervalForTruck(truck, type) {
  const g = TRUCK_TYPE_TO_GROUP[truck.truck_type] || "all";
  return intervalsByGroup[g]?.[type] || intervalsByGroup["all"]?.[type] || null;
}
```

### Update last_mileage_recorded (atomic)
```typescript
supabase.from("trucks").update({
  last_mileage_recorded: mileage,
  last_mileage_at: new Date().toISOString(),
}).eq("id", truckId)
  .or(`last_mileage_recorded.is.null,last_mileage_recorded.lt.${mileage}`)
  .then(() => {});
```

### Session Grouping (Frontend)
```javascript
const sessions = [];
const sessionMap = {};
for (const r of data) {
  const sid = r.service_session_id || `single_${r.id}`;
  if (!sessionMap[sid]) {
    sessionMap[sid] = { session_id: r.service_session_id, items: [], head: null, total_cost: 0 };
    sessions.push(sessionMap[sid]);
  }
  const s = sessionMap[sid];
  s.items.push(r);
  s.total_cost += r.cost || 0;
  if (r.is_session_head || !s.head) s.head = r;
}
```

---

## 📦 11. Migration History (รัน QSL เรียงตามนี้)

ทุก migration อยู่ใน `supabase/migrations/` เรียงตามไฟล์

| File | Purpose |
|------|---------|
| `20240101_init.sql` | Initial schema (trucks, drivers, mileage_logs, fuel_logs, maintenance_logs, admin_users) |
| `20260518_truck_enhancements.sql` | + province, color, is_driver_available |
| `20260520_maintenance_constraints.sql` | + CHECK constraints |
| `20260524_truck_types.sql` | truck_type: 4wheel → pickup, + forklift/other |
| `20260525_owner_and_pickable.sql` | + is_owner (driver), is_pickable (truck) |
| `20260531_truck_reserved.sql` | + reserved_for_driver_id |
| `20260601_tab_permissions.sql` | + admin_users.tab_permissions JSONB |
| `20260602_perf_indexes.sql` | 7 indexes สำหรับ query speed |
| `20260603_maintenance_intervals.sql` | New table maintenance_intervals |
| `20260604_admin_driver_type.sql` | driver_type += 'admin' |
| `20260605_maintenance_v2.sql` | + service_session_id, labor_cost, is_session_head, last_mileage_recorded, icon, coolant |

### Latest unfiled migrations (รัน inline):
1. **Maintenance v3 (cost-only per item):**
```sql
INSERT INTO maintenance_intervals (maintenance_type, label_th, icon) VALUES
  ('wheel_bearing','เปลี่ยนลูกปืนล้อ','🔩'),
  ('wheel_rim','เปลี่ยนกะทะล้อ','🛞'),
  ('tire_patch','ปะยาง','🔧'),
  ('labor','ค่าแรง','💼')
ON CONFLICT (maintenance_type) DO NOTHING;
ALTER TABLE maintenance_logs ADD COLUMN IF NOT EXISTS total_cost_entered NUMERIC;
```

2. **Composite PK for groups:**
```sql
ALTER TABLE maintenance_intervals ADD COLUMN IF NOT EXISTS truck_type_group TEXT NOT NULL DEFAULT 'all';
DO $$ BEGIN
  IF EXISTS (SELECT 1 FROM pg_constraint WHERE conname = 'maintenance_intervals_pkey') THEN
    ALTER TABLE maintenance_intervals DROP CONSTRAINT maintenance_intervals_pkey;
  END IF;
END $$;
ALTER TABLE maintenance_intervals ADD CONSTRAINT maintenance_intervals_pkey
  PRIMARY KEY (truck_type_group, maintenance_type);
CREATE INDEX IF NOT EXISTS idx_maint_intervals_type ON maintenance_intervals(maintenance_type);
```

3. **Storage nullable (fix multi-item insert):**
```sql
ALTER TABLE maintenance_logs
  ALTER COLUMN mileage_image_storage DROP NOT NULL,
  ALTER COLUMN bill_image_storage    DROP NOT NULL;
```

---

## 🛡 12. Security Implementations

| Feature | Implementation |
|---------|----------------|
| LIFF ID token verify | `_shared/security.ts` → `verifyLiffToken(idToken, channelId)` |
| PII encryption | AES-GCM 256-bit, fail-loud (`encryptPII()`, `decryptPII()`) |
| Mass assignment | Whitelist ใน update endpoints ทุกตัว |
| XSS | `esc()` ทุก user-data interpolation (มี 40+ ที่ในแต่ละไฟล์) |
| CORS | `ALLOWED_ORIGINS.includes(origin)` exact match |
| Rate limit | Sliding window — login 5/5min, submit 20/min |
| Brute force | Login lockout per (IP, username) |
| Signed URLs | สำหรับรูปทุกใบ — หมดอายุได้ (กัน leak) |
| Audit log | PII redaction ก่อน insert |
| Role-based UI | `super-only` CSS class — hide delete buttons |

---

## 🗂 13. File Structure

```
truck service tracker/
└── truck service tracker/
    └── supabase-system/
        ├── CLAUDE.md                       ← Quick reference
        ├── PROJECT_KNOWLEDGE.md            ← This file (full knowledge)
        ├── supabase/
        │   ├── config.toml
        │   ├── functions/
        │   │   ├── admin-api/
        │   │   │   └── index.ts            ← Backend admin (1900+ lines)
        │   │   ├── liff-api/
        │   │   │   └── index.ts            ← Backend LIFF (1300+ lines)
        │   │   └── _shared/
        │   │       ├── security.ts         ← Auth, encryption, R2, rate limit
        │   │       ├── datetime.ts         ← Bangkok timezone
        │   │       ├── ocr.ts              ← OCR helpers (disabled by user choice)
        │   │       └── line-notify.ts      ← LINE Messaging API
        │   └── migrations/
        │       └── 20*.sql                 ← Migrations ตามวันที่
        └── static/
            ├── admin.html                  ← Admin UI (4500+ lines)
            ├── liff.html                   ← LIFF UI (2400+ lines)
            └── gps-test.html               ← GPS debug page

D:\github\truck-liff\                       ← GitHub repo (HTML files only)
├── admin.html
├── liff.html
└── gps-test.html
```

---

## ⚡ 14. Key Decisions / Why's

1. **No labor_cost per item** — ค่าแรงเป็น type แยก (`type=labor`)
2. **No next_service_mileage** — compute จาก interval + baseline
3. **OCR disabled** — user found AI too inaccurate (เคยอ่านชื่อบริษัทผิด)
4. **Liters precision 10 decimal places** — ป้องกัน rounding loss
5. **total_cost_entered = integer** — สรุปบัญชีตรง ไม่มี decimal สับสน
6. **Multi-item shared image** — 1 บิล/รูป แต่หลาย items (efficient storage)
7. **Compute km_since_last_fuel on-the-fly** — DB value อาจผิดจาก backdate
8. **truck_type_group composite PK** — flexibility per truck class
9. **Admin can backdate** — สำหรับกรอกข้อมูลเก่า
10. **Forklift uses hours** — `mileage_at_service` เก็บค่าเดียวกัน แต่ UI label เปลี่ยน

---

## 🛠 15. Active Features Status

| Feature | Status |
|---------|--------|
| Driver registration via LIFF | ✅ Working |
| Mileage logging (morning/evening) | ✅ |
| Fuel logging + bill | ✅ |
| Multi-item maintenance | ✅ |
| Maintenance schedule per group | ✅ |
| Bill verification | ✅ |
| Monthly export (PDF + ZIP) | ✅ |
| Tax/insurance reminders | ✅ |
| Anomaly detection | ✅ |
| Admin backdate | ✅ |
| Per-truck-type intervals | ✅ |
| Custom maintenance types | ✅ (admin can add) |
| Role-based tab permissions | ✅ |
| LINE notify on anomaly | ⚠️ Partial |
| OCR auto-verify fuel bills | ❌ Disabled |

---

## 💡 16. Tips สำหรับ Claude session ใหม่

1. **เปิดอ่าน `CLAUDE.md` ก่อนเสมอ** — quick reference
2. **PowerShell quirks สำคัญ:**
   - ไม่มี `&&` / `||` — ใช้ `;` หรือ `if ($?) {...}`
   - cwd reset ทุก command — ใช้ `Set-Location` ทุกครั้ง
3. **HTML deploy:**
   - Local: `D:\google drive\CLAUDE\truck service tracker\truck service tracker\supabase-system\static\`
   - Copy → `D:\github\truck-liff\`
   - git commit + push
4. **Edge Function deploy:** ใช้ `--no-verify-jwt` เสมอ
5. **Database changes:** บอก user ไปรันใน Supabase SQL Editor
6. **เวลาแก้ค่าใน UI:** ทดสอบให้ครบทั้ง normal flow + edge cases (admin backdate, forklift hours, multi-item)
7. **`esc()` ทุก user-data interpolation** — ห้ามลืม
8. **Forklift = ชม. ไม่ใช่ กม.** — ใช้ `mileageUnit()` helper เสมอ

---

## 🔗 17. Quick Reference Links

- **Supabase SQL Editor**: https://supabase.com/dashboard/project/joevecrppgctnemaqgan/sql/new
- **Supabase Functions**: https://supabase.com/dashboard/project/joevecrppgctnemaqgan/functions
- **Supabase Storage**: https://supabase.com/dashboard/project/joevecrppgctnemaqgan/storage/buckets
- **GitHub Repo**: https://github.com/jtgchainat3-max/truck-liff
- **GitHub Actions**: https://github.com/jtgchainat3-max/truck-liff/actions
- **Admin Live**: https://jtgchainat3-max.github.io/truck-liff/admin.html
- **LIFF Live**: https://jtgchainat3-max.github.io/truck-liff/liff.html
- **LIFF URL (in LINE)**: ดูใน LINE Developer Console
- **GitHub Token Settings**: https://github.com/settings/tokens

---

**📝 หมายเหตุสำหรับ Claude:**
- ระบบนี้ใช้งานจริงในบริษัท — ทุก deploy ต้อง careful
- ห้าม destructive ops (DROP TABLE, DELETE without WHERE) โดยไม่ confirm
- ห้าม push secrets ไป GitHub
- ทุก migration ต้อง test ใน mind ก่อนให้ user รัน
- เก็บ user data ปลอดภัย (PII encrypt + audit redact)
