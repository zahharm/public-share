# คู่มือการใช้งาน IPD Dashboard API

> **Version:** 1.0.0 | **Updated:** 2026-05-21

---

## สารบัญ

1. [ภาพรวม](#1-ภาพรวม)
2. [Base URL](#2-base-url)
3. [การยืนยันตัวตน (Authentication)](#3-การยืนยันตัวตน-authentication)
4. [รูปแบบ Response](#4-รูปแบบ-response)
5. [รหัส Error](#5-รหัส-error)
6. [Endpoints](#6-endpoints)
   - [Health Check](#61-get-health)
   - [Readiness Check](#62-get-ready)
   - [รับใหม่วันนี้](#63-get-apiv1ipdadmissions-today)
   - [ผู้ป่วยปัจจุบัน](#64-get-apiv1ipdcurrent-patients)
   - [จำหน่ายวันนี้](#65-get-apiv1ipddischarges-today)
   - [การเคลื่อนย้ายเตียง](#66-get-apiv1ipdbed-moves-today)
7. [Cache & Rate Limit](#7-cache--rate-limit)
8. [ตัวอย่างการใช้งาน](#8-ตัวอย่างการใช้งาน)

---

## 1. ภาพรวม

IPD Dashboard API ให้ข้อมูลสถิติผู้ป่วยใน (IPD) แบบ Real-time จากฐานข้อมูล HosXP PostgreSQL
ออกแบบมาสำหรับแสดงผลบน Dashboard โดยเฉพาะ

**คุณสมบัติหลัก:**
- ข้อมูล Real-time จากฐานข้อมูล HosXP
- Redis Cache ลดภาระ DB
- Bearer Token Authentication
- Rate Limiting ป้องกันการใช้งานเกินกำหนด
- รองรับชื่อ Ward ภาษาไทย UTF-8

---

## 2. Base URL

| Environment | URL |
|-------------|-----|
| **Production** | `https://uttaradit-hosp.moph.go.th/api-hos` |
| **Development** | `http://localhost:8000` |
| **Internal** | `http://172.17.1.240:8000` |

**Swagger UI (ทดสอบ API):**
- Production: `https://uttaradit-hosp.moph.go.th/api-hos/api-docs`
- Development: `http://localhost:8000/api-docs`

---

## 3. การยืนยันตัวตน (Authentication)

ทุก endpoint ต้องใส่ **Bearer Token** ใน HTTP Header ยกเว้น `/health` และ `/ready`

### วิธีที่ 1 — Authorization Header (แนะนำ)
```
Authorization: Bearer <API_KEY>
```

### วิธีที่ 2 — X-API-Key Header
```
X-API-Key: <API_KEY>
```

### API Key

| Environment | Key |
|-------------|-----|
| Production | `ipd-prod-key-2026` |
| Development | `ipd-dev-key-2026` |

### ตัวอย่าง curl
```bash
curl -H "Authorization: Bearer ipd-prod-key-2026" \
     https://uttaradit-hosp.moph.go.th/api-hos/api/v1/ipd/admissions-today
```

### วิธีใช้ใน Swagger UI
1. เปิด `https://uttaradit-hosp.moph.go.th/api-hos/api-docs`
2. คลิกปุ่ม **Authorize 🔓** (มุมขวาบน)
3. ใส่ `ipd-prod-key-2026` ในช่อง **Value**
4. คลิก **Authorize** → **Close**
5. ทดสอบ endpoint ได้เลย (token จะถูกส่งอัตโนมัติ)

---

## 4. รูปแบบ Response

### Success Response
```json
{
  "success": true,
  "timestamp": "2026-05-21T13:00:00.000000+00:00",
  "data": { ... },
  "message": "OK"
}
```

### Error Response
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "คำอธิบาย error"
  }
}
```

### Response Headers ที่สำคัญ
| Header | ความหมาย |
|--------|----------|
| `X-Cache: HIT` | ข้อมูลมาจาก Redis Cache (เร็ว) |
| `X-Cache: MISS` | ข้อมูลมาจาก Database |
| `X-Request-ID` | รหัส Request สำหรับ Debugging |

---

## 5. รหัส Error

| HTTP Status | Code | ความหมาย |
|-------------|------|----------|
| 401 | `MISSING_TOKEN` | ไม่มี Authorization header |
| 401 | `INVALID_API_KEY` | API Key ไม่ถูกต้อง |
| 429 | `RATE_LIMITED` | เรียก API เกินกำหนด (200 req/60s) |
| 504 | `DB_TIMEOUT` | Database query ช้าเกิน 8 วินาที |
| 500 | `INTERNAL_ERROR` | Server error (ดู X-Request-ID สำหรับ support) |

---

## 6. Endpoints

---

### 6.1 `GET /health`

ตรวจสอบว่า server กำลัง run อยู่หรือไม่ (**ไม่ต้อง token**)

**Request:**
```bash
curl https://uttaradit-hosp.moph.go.th/api-hos/health
```

**Response:**
```json
{
  "status": "ok",
  "app": "IPD Dashboard API",
  "version": "1.0.0",
  "environment": "production",
  "timestamp": "2026-05-21T13:00:00+00:00"
}
```

---

### 6.2 `GET /ready`

ตรวจสอบว่า Database และ Redis พร้อมใช้งาน (**ไม่ต้อง token**)

**Request:**
```bash
curl https://uttaradit-hosp.moph.go.th/api-hos/ready
```

**Response:**
```json
{
  "status": "ok",
  "timestamp": "2026-05-21T13:00:00+00:00",
  "checks": {
    "database": "ok",
    "redis": "ok"
  }
}
```

---

### 6.3 `GET /api/v1/ipd/admissions-today`

**จำนวนผู้ป่วยรับใหม่วันนี้ แยกตาม Ward**

- ข้อมูลจาก: ตาราง `ipt` กรอง `regdate = วันนี้`
- เฉพาะ Ward ที่ Active (`ward_active = 'Y'`) เท่านั้น
- **Cache:** 5 นาที

**Request:**
```bash
curl -H "Authorization: Bearer ipd-prod-key-2026" \
     https://uttaradit-hosp.moph.go.th/api-hos/api/v1/ipd/admissions-today
```

**Response:**
```json
{
  "success": true,
  "timestamp": "2026-05-21T13:00:00.000000+00:00",
  "data": {
    "items": [
      {
        "ward": "01",
        "ward_name": "อายุรกรรมชาย 1",
        "regdate": "2026-05-21",
        "total_admissions": 7
      },
      {
        "ward": "04",
        "ward_name": "อายุรกรรมหญิง 2",
        "regdate": "2026-05-21",
        "total_admissions": 3
      }
    ],
    "total": 120
  },
  "message": "OK"
}
```

**คำอธิบาย Fields:**
| Field | Type | ความหมาย |
|-------|------|----------|
| `ward` | string | รหัส Ward |
| `ward_name` | string | ชื่อ Ward (ภาษาไทย) |
| `regdate` | string (date) | วันที่รับ (YYYY-MM-DD) |
| `total_admissions` | integer | จำนวนรับใหม่ |
| `total` | integer | รวมทุก Ward |

---

### 6.4 `GET /api/v1/ipd/current-patients`

**จำนวนผู้ป่วยที่ยังอยู่ในโรงพยาบาล ณ ขณะนี้ แยกตาม Ward**

- ข้อมูลจาก: ตาราง `ipt` กรอง `confirm_discharge = 'N'`
- **Cache: 1 นาที** (อัปเดตบ่อยที่สุด — สำหรับข้อมูล Real-time)

**Request:**
```bash
curl -H "Authorization: Bearer ipd-prod-key-2026" \
     https://uttaradit-hosp.moph.go.th/api-hos/api/v1/ipd/current-patients
```

**Response:**
```json
{
  "success": true,
  "timestamp": "2026-05-21T13:00:00.000000+00:00",
  "data": {
    "items": [
      {
        "ward": "01",
        "ward_name": "อายุรกรรมชาย 1",
        "count_an": 36
      },
      {
        "ward": "19",
        "ward_name": "หออภิบาลผู้ป่วยวิกฤตอายุรกรรม",
        "count_an": 8
      }
    ],
    "total": 513
  },
  "message": "OK"
}
```

**คำอธิบาย Fields:**
| Field | Type | ความหมาย |
|-------|------|----------|
| `ward` | string | รหัส Ward |
| `ward_name` | string | ชื่อ Ward (ภาษาไทย) |
| `count_an` | integer | จำนวนผู้ป่วยที่ยังอยู่ |
| `total` | integer | รวมทุก Ward |

---

### 6.5 `GET /api/v1/ipd/discharges-today`

**จำนวนผู้ป่วยจำหน่ายวันนี้ แยกประเภทจำหน่าย (เสียชีวิต / อื่นๆ) ตาม Ward**

- ข้อมูลจาก: ตาราง `ipt` กรอง `dchdate = วันนี้` และ `confirm_discharge = 'Y'`
- `dchstts = '09'` = เสียชีวิต
- **Cache:** 5 นาที

**Request:**
```bash
curl -H "Authorization: Bearer ipd-prod-key-2026" \
     https://uttaradit-hosp.moph.go.th/api-hos/api/v1/ipd/discharges-today
```

**Response:**
```json
{
  "success": true,
  "timestamp": "2026-05-21T13:00:00.000000+00:00",
  "data": {
    "items": [
      {
        "regdate": "2026-05-20",
        "ward": "04",
        "ward_name": "อายุรกรรมหญิง 2",
        "total_discharges": 3,
        "count_dead": 1,
        "count_other": 2
      },
      {
        "regdate": "2026-05-21",
        "ward": "09",
        "ward_name": "ศัลยกรรมชาย",
        "total_discharges": 5,
        "count_dead": 0,
        "count_other": 5
      }
    ],
    "total_discharges": 110,
    "total_dead": 4
  },
  "message": "OK"
}
```

**คำอธิบาย Fields:**
| Field | Type | ความหมาย |
|-------|------|----------|
| `regdate` | string (date) | วันที่รับเข้า (ไม่ใช่วันจำหน่าย) |
| `ward` | string | รหัส Ward |
| `ward_name` | string | ชื่อ Ward (ภาษาไทย) |
| `total_discharges` | integer | จำนวนจำหน่ายทั้งหมด |
| `count_dead` | integer | จำนวนเสียชีวิต |
| `count_other` | integer | จำนวนจำหน่ายประเภทอื่น |
| `total_discharges` (root) | integer | รวมจำหน่ายทุก Ward |
| `total_dead` (root) | integer | รวมเสียชีวิตทุก Ward |

> **หมายเหตุ:** `regdate` คือวันที่ผู้ป่วยรับเข้า ซึ่งอาจเป็นวันก่อนหน้า (ผู้ป่วย Admit มาหลายวันแล้วจึงจำหน่าย)

---

### 6.6 `GET /api/v1/ipd/bed-moves-today`

**จำนวนการเคลื่อนย้ายเตียง / ย้าย Ward วันนี้**

- ข้อมูลจาก: ตาราง `iptbedmove` กรอง `movedate = วันนี้`
- `รับย้าย` = รับผู้ป่วยโอนมา (Transfer In)
- `ย้ายตึก` = ส่งผู้ป่วยไปอีก Ward (Transfer Out)
- **Cache:** 5 นาที

**Request:**
```bash
curl -H "Authorization: Bearer ipd-prod-key-2026" \
     https://uttaradit-hosp.moph.go.th/api-hos/api/v1/ipd/bed-moves-today
```

**Response:**
```json
{
  "success": true,
  "timestamp": "2026-05-21T13:00:00.000000+00:00",
  "data": {
    "items": [
      {
        "nward": "01",
        "ward_name": "อายุรกรรมชาย 1",
        "count_receive": 3,
        "count_move": 1,
        "total_moves": 4
      },
      {
        "nward": "19",
        "ward_name": "หออภิบาลผู้ป่วยวิกฤตอายุรกรรม",
        "count_receive": 5,
        "count_move": 2,
        "total_moves": 7
      }
    ],
    "total_moves": 39
  },
  "message": "OK"
}
```

**คำอธิบาย Fields:**
| Field | Type | ความหมาย |
|-------|------|----------|
| `nward` | string | รหัส Ward ปัจจุบัน |
| `ward_name` | string | ชื่อ Ward (ภาษาไทย) |
| `count_receive` | integer | รับโอนมา (Transfer In) |
| `count_move` | integer | ส่งออกไป (Transfer Out) |
| `total_moves` | integer | รวมการเคลื่อนย้ายของ Ward นี้ |
| `total_moves` (root) | integer | รวมทุก Ward |

---

## 7. Cache & Rate Limit

### Cache TTL
| Endpoint | TTL | เหตุผล |
|----------|-----|--------|
| `admissions-today` | **5 นาที** | ข้อมูลเปลี่ยนช้า |
| `current-patients` | **1 นาที** | Real-time census |
| `discharges-today` | **5 นาที** | ข้อมูลเปลี่ยนช้า |
| `bed-moves-today` | **5 นาที** | ข้อมูลเปลี่ยนช้า |

ดูว่าข้อมูลมาจาก Cache หรือ DB ได้จาก Response Header:
```
X-Cache: HIT    ← มาจาก Redis (เร็วมาก ~5ms)
X-Cache: MISS   ← มาจาก DB (~50-200ms)
```

### Rate Limit
- **200 requests / 60 วินาที** ต่อ API Key
- เกินแล้ว → HTTP 429 `RATE_LIMITED`
- Reset อัตโนมัติหลัง 60 วินาที

---

## 8. ตัวอย่างการใช้งาน

### JavaScript (fetch)
```javascript
const API_BASE = 'https://uttaradit-hosp.moph.go.th/api-hos';
const API_KEY  = 'ipd-prod-key-2026';

async function getAdmissionsToday() {
  const res = await fetch(`${API_BASE}/api/v1/ipd/admissions-today`, {
    headers: { 'Authorization': `Bearer ${API_KEY}` }
  });
  const json = await res.json();
  if (json.success) {
    console.log('รับใหม่วันนี้:', json.data.total, 'ราย');
    console.log('รายละเอียด:', json.data.items);
  }
}

async function getCurrentPatients() {
  const res = await fetch(`${API_BASE}/api/v1/ipd/current-patients`, {
    headers: { 'Authorization': `Bearer ${API_KEY}` }
  });
  const json = await res.json();
  if (json.success) {
    console.log('ผู้ป่วยปัจจุบัน:', json.data.total, 'ราย');
  }
}

// เรียกพร้อมกันทีเดียว (Parallel)
async function loadDashboard() {
  const [admissions, patients, discharges, moves] = await Promise.all([
    fetch(`${API_BASE}/api/v1/ipd/admissions-today`,  { headers: { Authorization: `Bearer ${API_KEY}` } }).then(r => r.json()),
    fetch(`${API_BASE}/api/v1/ipd/current-patients`,  { headers: { Authorization: `Bearer ${API_KEY}` } }).then(r => r.json()),
    fetch(`${API_BASE}/api/v1/ipd/discharges-today`,  { headers: { Authorization: `Bearer ${API_KEY}` } }).then(r => r.json()),
    fetch(`${API_BASE}/api/v1/ipd/bed-moves-today`,   { headers: { Authorization: `Bearer ${API_KEY}` } }).then(r => r.json()),
  ]);

  return {
    admissionsTotal:  admissions.data.total,
    patientsTotal:    patients.data.total,
    dischargesTotal:  discharges.data.total_discharges,
    deadTotal:        discharges.data.total_dead,
    movesTotal:       moves.data.total_moves,
  };
}
```

### Python (requests)
```python
import requests

API_BASE = "https://uttaradit-hosp.moph.go.th/api-hos"
HEADERS  = {"Authorization": "Bearer ipd-prod-key-2026"}

def get_dashboard():
    endpoints = {
        "admissions": f"{API_BASE}/api/v1/ipd/admissions-today",
        "patients":   f"{API_BASE}/api/v1/ipd/current-patients",
        "discharges": f"{API_BASE}/api/v1/ipd/discharges-today",
        "moves":      f"{API_BASE}/api/v1/ipd/bed-moves-today",
    }
    results = {}
    for name, url in endpoints.items():
        r = requests.get(url, headers=HEADERS, timeout=15)
        r.raise_for_status()
        results[name] = r.json()["data"]
    return results

data = get_dashboard()
print(f"รับใหม่วันนี้    : {data['admissions']['total']} ราย")
print(f"ผู้ป่วยปัจจุบัน  : {data['patients']['total']} ราย")
print(f"จำหน่ายวันนี้    : {data['discharges']['total_discharges']} ราย")
print(f"  - เสียชีวิต   : {data['discharges']['total_dead']} ราย")
print(f"ย้าย Ward วันนี้ : {data['moves']['total_moves']} ครั้ง")
```

### PHP (curl)
```php
<?php
$API_BASE = 'https://uttaradit-hosp.moph.go.th/api-hos';
$API_KEY  = 'ipd-prod-key-2026';

function callApi(string $path): array {
    global $API_BASE, $API_KEY;
    $ch = curl_init("$API_BASE$path");
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER     => ["Authorization: Bearer $API_KEY"],
        CURLOPT_TIMEOUT        => 15,
        CURLOPT_SSL_VERIFYPEER => true,
    ]);
    $response = curl_exec($ch);
    curl_close($ch);
    return json_decode($response, true);
}

$admissions = callApi('/api/v1/ipd/admissions-today');
$patients   = callApi('/api/v1/ipd/current-patients');
$discharges = callApi('/api/v1/ipd/discharges-today');

echo "รับใหม่วันนี้  : " . $admissions['data']['total'] . " ราย\n";
echo "ผู้ป่วยปัจจุบัน: " . $patients['data']['total']   . " ราย\n";
echo "จำหน่ายวันนี้  : " . $discharges['data']['total_discharges'] . " ราย\n";
echo "เสียชีวิต     : " . $discharges['data']['total_dead']        . " ราย\n";
```

### curl (ทดสอบจาก Terminal)
```bash
BASE="https://uttaradit-hosp.moph.go.th/api-hos"
KEY="ipd-prod-key-2026"

# ดูข้อมูลทั้งหมดพร้อมกัน
for endpoint in admissions-today current-patients discharges-today bed-moves-today; do
  echo "=== $endpoint ==="
  curl -s -H "Authorization: Bearer $KEY" \
       "$BASE/api/v1/ipd/$endpoint" | python3 -m json.tool
  echo ""
done
```

---

## สรุป Endpoints

| Method | Path | ต้อง Token | Cache | ความหมาย |
|--------|------|-----------|-------|----------|
| GET | `/health` | ไม่ | - | Server alive? |
| GET | `/ready` | ไม่ | - | DB + Redis ready? |
| GET | `/api/v1/ipd/admissions-today` | ✅ | 5 นาที | รับใหม่วันนี้ |
| GET | `/api/v1/ipd/current-patients` | ✅ | 1 นาที | ผู้ป่วยที่ยังอยู่ |
| GET | `/api/v1/ipd/discharges-today` | ✅ | 5 นาที | จำหน่ายวันนี้ |
| GET | `/api/v1/ipd/bed-moves-today` | ✅ | 5 นาที | ย้าย Ward วันนี้ |

---

*สอบถามเพิ่มเติม ดู Swagger UI ที่ `https://uttaradit-hosp.moph.go.th/api-hos/api-docs`*
