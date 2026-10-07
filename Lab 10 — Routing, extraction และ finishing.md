## Lab 10 — Routing, extraction และ finishing

Lab นี้นำ checkpoint ที่ผ่าน **Lab 9 — Placement, CTS และ timing closure** มาสร้างเส้นทางเดินสายจริง ตรวจ connectivity และ antenna สกัด parasitic resistance/capacitance ออกเป็น SPEF วิเคราะห์ timing หลัง routing และสร้าง layout ที่พร้อมเข้าสู่การตรวจขั้นสุดท้าย

**ผลลัพธ์ที่ต้องการ:** Routed database, DEF, netlist, SPEF, post-route STA reports และ GDS ที่มีประวัติการสร้างสอดคล้องกัน พร้อมรายการตรวจ finishing ที่ระบุชัดว่าอะไรผ่าน อะไรยังไม่รัน และอะไรต้องแก้

> ตัวอย่างคำสั่งใช้ `librelane/config.json` ให้ปรับ path และชื่อ steps ตามโครงการ NEORV32 จริง ขั้นตอนนี้เป็นคู่มือปฏิบัติ ไม่ใช่ผลการรันหรือการรับรอง sign-off ของโครงการ

---

### 10.1 วัตถุประสงค์การเรียนรู้

เมื่อจบ Lab ผู้เรียนควรสามารถ:

1. ตรวจความพร้อมของ design ก่อน routing
2. แยก global routing กับ detailed routing
3. วิเคราะห์ congestion, pin-access problems และ routing violations
4. ตรวจ antenna และ connectivity หลังเดินสาย
5. ตรวจความถูกต้องของ SPEF และการจับคู่ extraction corners
6. อ่าน setup/hold reports ที่ใช้ extracted parasitics
7. ทำ finishing ตามข้อกำหนดของ PDK/template
8. สร้างชุด artifacts ที่ตรวจสอบย้อนหลังและส่งต่อได้

### 10.2 ความหมายของแต่ละขั้นตอน

| ขั้นตอน | หน้าที่ | ผลลัพธ์สำคัญ |
|---|---|---|
| Global routing | วางแผนเส้นทางและจัดสรร routing resources | Routing guides, congestion assessment |
| Detailed routing | สร้าง wires/vias ตาม geometry และกฎ routing | Routed ODB/DEF และ router DRC report |
| Antenna checking/repair | ตรวจความเสี่ยง gate damage ระหว่าง fabrication | Antenna report และผล repair |
| Connectivity checking | ตรวจการเชื่อมต่อหลัง routing | Disconnected-pin/net reports |
| Parasitic extraction | สกัด R/C จาก geometry ตาม extraction model | SPEF |
| Post-route STA | วิเคราะห์ timing ด้วย routed parasitics | Setup/hold/electrical reports |
| Finishing/stream-out | เตรียม physical structures และ export layout | GDS และ finishing records |
| Final verification | ตรวจ layout และวงจรตาม methodology | DRC/LVS/antenna/density/STA matrix |

LibreLane Classic มี steps สำหรับ global routing, detailed routing, antenna checks, connectivity checks, filler insertion, RC extraction และ post-PNR STA ก่อน stream-out แต่ template ของ IHP อาจเพิ่มหรือปรับลำดับขั้นตอน จึงต้องใช้ flow ที่ล็อกไว้ในโครงการเป็นหลัก [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/reference/flows.html?utm_source=chatgpt.com)

---

### 10.3 สิ่งที่ต้องเตรียมจาก Lab 9

| Input | จุดตรวจ |
|---|---|
| Post-CTS หรือ post-global-route ODB | ตรงกับ checkpoint ที่เลือก |
| Updated netlist | รวม buffers/cells ที่เพิ่มจาก CTS และ repair |
| SDC | เป็นชุดที่ตรวจผ่านใน Lab 7 และใช้ใน Lab 9 |
| Timing reports | มี setup/hold ครบตาม required corners |
| Technology LEF และ cell/macro LEF | ตรงกับ PDK revision |
| Macro GDS | มีครบสำหรับ stream-out |
| RC extraction models | รองรับ SG13G2 และ corners ที่ต้องใช้ |
| Pad/PDN database | Connectivity และ geometry ผ่านเกณฑ์ก่อนหน้า |

**ข้อควรระวังสำหรับ NEORV32 full-chip**

- Clock network ต้องผ่านจาก clock pad ไปถึง core sinks ครบ
- Reset network ต้องรักษาการเชื่อมต่อและ reset-release methodology
- SRAM macro ต้องมี LEF, GDS, timing model และ netlist view ที่เหมาะสม
- Power domains ของ core และ I/O ต้องตรงกันทุก view
- Cells ที่เพิ่มจาก timing repair ต้องมี power connections ครบ

---

### 10.4 ขั้นตอนที่ 1 — ล็อก input และเลือก checkpoint

สร้างพื้นที่เก็บรายงาน:

```bash
mkdir -p reports/lab10

python3 -m librelane --help \
  > reports/lab10/librelane-help.txt

python3 - <<'PY' > reports/lab10/tool-version.txt
from importlib.metadata import version
print("LibreLane:", version("librelane"))
PY

openroad -version \
  >> reports/lab10/tool-version.txt

sha256sum librelane/config.json \
  > reports/lab10/config.sha256
```

บันทึก SDC ทุกไฟล์ที่ใช้งานจริงด้วย ไม่ใช้เฉพาะ SDC ที่คัดลอกไว้เก่า

กำหนด Lab 9 run:

```bash
export LAB9_RUN="librelane/runs/<ชื่อ-run-Lab9>"

test -d "$LAB9_RUN" || {
  echo "Lab 9 run directory not found"
  exit 1
}

rg --files "$LAB9_RUN" \
  | rg 'state_out\.json$|\.odb$|\.sdc$|metrics.*\.json$'
```

เลือกจุดเริ่มให้ตรงกับงานที่ทำแล้ว:

| สถานะ Lab 9 | จุดเริ่ม Lab 10 |
|---|---|
| จบหลัง CTS/timing repair | เริ่ม global routing |
| จบหลัง global routing แต่ยังไม่ได้ทำ repair/checks ที่ตามมา | เริ่ม step ถัดไปตาม flow |
| จบครบช่วงก่อน detailed routing | เริ่ม detailed routing |

**ห้ามข้าม antenna repair, legalization หรือ STA steps ระหว่าง global routing กับ detailed routingโดยไม่ตรวจว่าได้รันแล้ว**

```bash
export ROUTE_INPUT_STATE="<checkpoint-ที่เลือก>/state_out.json"

test -f "$ROUTE_INPUT_STATE" || {
  echo "Routing input checkpoint not found"
  exit 1
}
```

---

### 10.5 ขั้นตอนที่ 2 — ตรวจ routing configuration

ค้นหาค่าที่เกี่ยวข้อง:

```bash
rg -n \
  'ROUT|GRT|DRT|ANTENNA|RCX|SPEF|CORNER|FILL|GDS' \
  librelane/config.json
```

ตรวจ resolved configuration และ PDK configuration เพิ่มเติมสำหรับค่าที่ไม่ได้กำหนดในไฟล์นี้

| กลุ่มค่า | สิ่งที่ต้องยืนยัน |
|---|---|
| Signal routing layers | ใช้ชั้นโลหะที่ PDK อนุญาต |
| Clock routing layers | สอดคล้องกับ clock RC assumptions |
| Routing capacity adjustments | มีที่มาและไม่ซ่อน congestion |
| Macro/pad obstructions | Router รับรู้พื้นที่ต้องห้ามครบ |
| Via rules | มาจาก technology ของ SG13G2 |
| Antenna properties | Cell/macro views มีข้อมูลที่ต้องใช้ |
| RC models | เป็นของ stack และ PDK revision เดียวกัน |
| Stream-out views | GDS ของ cells/macros ครบ |

ตัวอย่าง flags ของ Classic flow:

```json
{
  "RUN_DRT": true,
  "RUN_ANTENNA_REPAIR": true,
  "RUN_FILL_INSERTION": true,
  "RUN_SPEF_EXTRACTION": true,
  "RUN_MCSTA": true
}
```

ตัวอย่างนี้เป็น **ส่วนที่เพิ่มใน configuration เดิม** ไม่ใช่ configuration ที่ใช้รันได้โดยลำพัง และต้องตรวจการรองรับกับ flow/template ที่ใช้จริง [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/reference/flows.html?utm_source=chatgpt.com)

---

### 10.6 ขั้นตอนที่ 3 — รัน global routing

หากเริ่มจาก checkpoint ก่อน global routing:

```bash
python3 -m librelane \
  --run-tag lab10_grt_baseline \
  --with-initial-state "$ROUTE_INPUT_STATE" \
  --from OpenROAD.GlobalRouting \
  --to OpenROAD.GlobalRouting \
  librelane/config.json
```

การหยุดที่ global routing ใช้สำหรับตรวจ baseline เท่านั้น หลังจากนั้นต้องรัน steps ที่เหลือตาม flow ก่อนเข้า detailed routing

ตรวจ log และ reports:

```bash
export GRT_RUN="librelane/runs/lab10_grt_baseline"

rg -n -i \
  'overflow|congestion|capacity|demand|unrout|error|warning' \
  "$GRT_RUN" \
  -g '*.log' -g '*.rpt'
```

#### อ่าน congestion อย่างไร

Routing congestion เกิดเมื่อจำนวนเส้นทางที่ต้องผ่านบริเวณหนึ่งมากกว่า resources ที่ใช้ได้ในบริเวณนั้น

ต้องตรวจ:

1. Overflow ต่อ layer
2. ตำแหน่ง hotspot
3. ความสัมพันธ์กับ macro pins และ channels
4. บริเวณที่ PDN ลด routing resources
5. Bus จำนวนมากที่ต้องข้าม core
6. Clock branches ที่แข่งขันพื้นที่กับ signal routing

**Total overflow ต่ำไม่ได้แปลว่าไม่มีปัญหา pin access** เพราะ hotspot เล็กใกล้ macro หรือ pad อาจทำให้ detailed routing ล้มเหลวได้

#### แนวทางแก้ congestion

| สาเหตุ | แนวทางแก้ |
|---|---|
| Standard cells หนาแน่น | ลด density หรือเพิ่ม usable core area |
| Macro channel แคบ | ปรับ macro spacing/orientation |
| Macro pins กระจุกตัว | ปรับตำแหน่งและทิศทาง macro |
| Bus ข้าม core จำนวนมาก | ปรับ placement หรือ architecture หากจำเป็น |
| PDN กิน resources มาก | ทบทวน PDN โดยยังรักษา electrical requirements |
| Routing obstructions ผิด | แก้ LEF/configuration ให้ตรงกับ physical intent |
| Buffer insertion มาก | ทบทวน timing repair และ floorplan |

เมื่อแก้ floorplan หรือ placement ต้องย้อนรันจากขั้นที่ได้รับผลกระทบ ไม่ใช้ routed database เดิมเป็นหลักฐานของ design ที่เปลี่ยนแล้ว

---

### 10.7 ขั้นตอนที่ 4 — ตรวจ antenna ก่อน detailed routing

Antenna violation เกี่ยวข้องกับการสะสมประจุบน conductive geometry ที่เชื่อมกับ gate ระหว่างกระบวนการผลิต ไม่ใช่ปัญหาคลื่นวิทยุ

ตรวจรายงาน:

```bash
rg -n -i \
  'antenna|diode|violat|repair' \
  "$GRT_RUN" \
  -g '*.log' -g '*.rpt'
```

สำหรับแต่ละ violation ให้บันทึก:

- Net name
- Gate/pin ที่ได้รับผลกระทบ
- Layer
- Antenna ratio และ allowable limit
- วิธี repair
- ผลตรวจหลัง repair

วิธีแก้อาจใช้ diode insertion หรือ routing modification ตามความสามารถของ flow และกฎ PDK

**ต้องตรวจ cells/macros ที่ไม่มี antenna properties ด้วย** เพราะ “ไม่พบ violations” อาจเกิดจากขอบเขตการตรวจไม่ครบ

หลัง diode insertion ให้ตรวจ:

1. Placement legality
2. Diode connectivity และ supplies ตาม cell definition
3. Timing/load ที่เปลี่ยน
4. Routing updates
5. Antenna report รอบใหม่

---

### 10.8 ขั้นตอนที่ 5 — รัน detailed routing

เลือก checkpoint ที่ผ่านขั้นเตรียมก่อน detailed routingครบแล้ว:

```bash
export PRE_DRT_STATE="<checkpoint-ก่อน-detailed-routing>/state_out.json"

python3 -m librelane \
  --run-tag lab10_drt_baseline \
  --with-initial-state "$PRE_DRT_STATE" \
  --from OpenROAD.DetailedRouting \
  --to OpenROAD.DetailedRouting \
  librelane/config.json
```

กำหนด run:

```bash
export DRT_RUN="librelane/runs/lab10_drt_baseline"
```

ตรวจ:

```bash
rg -n -i \
  'violation|short|spacing|unrout|pin.access|iteration|error|warning' \
  "$DRT_RUN" \
  -g '*.log' -g '*.rpt'
```

#### สิ่งที่ต้องอ่านจาก router

| รายการ | ความหมาย |
|---|---|
| Initial/final violation count | ความคืบหน้าการแก้กฎ routing |
| Unrouted connections | การเชื่อมต่อที่ยังทำไม่ครบ |
| Wirelength | ปริมาณสายที่สร้างจริง |
| Via count | ความซับซ้อนของการเปลี่ยนชั้น |
| Pin-access failures | Router เข้าถึง pin ไม่ได้ |
| Iteration limit | หยุดเพราะถึงขีดจำกัดหรือแก้ครบจริง |
| Final DRC markers | ตำแหน่งปัญหาที่ต้องตรวจ |

**Exit code สำเร็จไม่ใช่หลักฐานว่าทุก connection และทุก DRC rule ผ่าน** ต้องตรวจ reports และ checker steps หลัง routingด้วย

---

### 10.9 ขั้นตอนที่ 6 — ตรวจ routed layout ด้วย GUI

เปิด ODB/DEF หลัง detailed routing โดยโหลด technology และ LEF ให้ตรงกัน

ตรวจตามลำดับ:

1. แสดง routing layers และ vias
2. เปิด DRC markers
3. เลือก nets ที่มี violations
4. ตรวจบริเวณ macro pins
5. ตรวจ clock network
6. ตรวจ pad-to-core routes
7. ตรวจ crossing ใกล้ PDN rings/straps
8. ตรวจช่องทางที่มี congestion จาก global routing

เก็บภาพอย่างน้อย:

- Full-chip overview
- Core routing
- SRAM/macro pin access หากมี
- Clock branch ตัวอย่าง
- DRC hotspot ก่อน–หลังแก้

#### การวิเคราะห์ตัวอย่าง

**กรณี short ระหว่าง nets**

ตรวจว่ามาจาก:

- Router geometry
- PDN/interface geometry
- LEF pin/obstruction ที่ไม่ตรงกับ GDS
- Macro boundary
- Stream-out layer mapping

**กรณี pin access ไม่ได้**

ตรวจ:

- Pin shape เล็กหรืออยู่ชิด obstruction
- Macro orientation
- Channel width
- Routing layers ที่อนุญาต
- Via landing/enclosure

ไม่เพิ่ม routing iterations อย่างเดียวหากสาเหตุเป็น geometry หรือ configuration ที่ไม่ถูกต้อง

---

### 10.10 ขั้นตอนที่ 7 — รัน checks หลัง detailed routing

รันต่อผ่าน antenna และ connectivity checks ตาม flow จริง

| Check | สิ่งที่ต้องพิสูจน์ |
|---|---|
| Post-route antenna | ไม่มี unresolved antenna violations |
| Router DRC checker | ไม่มี unresolved router violations |
| Disconnected pins | Required pins เชื่อมครบ |
| Signal connectivity | ไม่มี required signal nets ขาด |
| Power connectivity | Cells ที่เพิ่มใหม่ได้รับ supply |
| Placement legality | Physical cells และ repair cells legal |

LibreLane มี `Odb.ReportDisconnectedPins` และ checker ที่เกี่ยวข้องหลัง detailed routing แต่ต้องอ่านรายละเอียดเพื่อแยก pins ที่ตั้งใจไม่ต่อจาก pins ที่ขาดโดยผิดพลาด

**ห้ามใช้การ ignore ทั้ง module เพื่อทำให้ connectivity report ผ่านโดยยังไม่ตรวจ pin-level intent**

สำหรับ SG13G2 full-chip ให้ตรวจเป็นพิเศษ:

- Core VDD/VSS
- I/O supplies
- Power/ground pads
- SRAM power pins
- Clock buffers
- Antenna diodes
- Filler/tap cells ตาม physical-cell definitions

---

### 10.11 ขั้นตอนที่ 8 — แยกประเภท finishing ให้ถูกต้อง

คำว่า “fill” อาจหมายถึงคนละอย่าง:

| ประเภท | หน้าที่ | เกี่ยวข้องกับ |
|---|---|---|
| Standard-cell filler | เติมช่องว่างใน cell rows ตาม library requirements | Wells/rails และ physical continuity |
| Tap/endcap | จัดการ well/substrate contacts และ row boundaries | Latch-up/physical rules |
| Pad fillers/corners | เติมและปิดโครงสร้าง pad ring | I/O ring continuity |
| Dummy metal fill | ปรับ metal density ตาม manufacturing rules | Density และ parasitic impact |
| Seal ring | โครงสร้างรอบ die ตามข้อกำหนด fabrication | Mechanical/process protection |

**Standard-cell filler insertion ไม่ได้พิสูจน์ว่า metal density ผ่าน**

สำหรับ tap/endcap ที่ flow ใส่ไว้ตั้งแต่ floorplan ให้ตรวจว่าถูกต้องและยังอยู่ครบ ไม่ใส่ซ้ำโดยอัตโนมัติ

Finishing ต้องใช้ cells, dimensions, layers และ scripts จาก PDK/template revision ที่กำหนด ห้ามสร้าง seal ring หรือ dummy fill จากการคาดเดา

---

### 10.12 ขั้นตอนที่ 9 — เตรียม parasitic extraction

ก่อนรัน RCX ต้องยืนยัน:

1. Routed ODB เป็นเวอร์ชันล่าสุด
2. Netlist ตรงกับ database
3. Extraction rules เป็นของ SG13G2
4. Layer stack และ units ถูกต้อง
5. Required extraction corners ครบ
6. กำหนดความสัมพันธ์ระหว่าง RC corners กับ timing corners
7. ขอบเขต macro parasitics ชัดเจน
8. วิธีจัดการ metal fill สอดคล้องกับ methodology

OpenRCX สกัด parasitic resistance และ capacitance โดยใช้ routed geometry และ extraction models และสามารถเขียนผลเป็น SPEF ได้ [OpenROAD documentation](https://openroad.readthedocs.io/en/latest/main/src/rcx/README.html?utm_source=chatgpt.com)

#### อย่าสับสน PVT corner กับ RC corner

| ชนิด corner | อธิบาย |
|---|---|
| Cell PVT | Process, voltage, temperature ของ cell timing models |
| RC extraction | เงื่อนไข interconnect resistance/capacitance |
| Analysis scenario | การจับคู่ mode, cell libraries, RC model และ constraints |

ไม่ควรอนุมานว่า RC corner หนึ่งเหมาะกับทุก cell cornerโดยไม่มี mapping จาก methodology

---

### 10.13 ขั้นตอนที่ 10 — รัน RC extraction และ post-route STA

เลือก checkpoint ที่รวม routing และ physical-cell insertion ที่ต้องอยู่ก่อน RCX แล้ว:

```bash
export PRE_RCX_STATE="<checkpoint-ก่อน-RCX>/state_out.json"

python3 -m librelane \
  --run-tag lab10_rcx_sta \
  --with-initial-state "$PRE_RCX_STATE" \
  --from OpenROAD.RCX \
  --to OpenROAD.STAPostPNR \
  librelane/config.json
```

ตรวจว่า steps เหล่านี้อยู่ใน flow ที่ใช้และถูกเปิดใช้งาน

```bash
export RCX_RUN="librelane/runs/lab10_rcx_sta"

rg --files "$RCX_RUN" \
  | rg '\.spef(\.gz)?$|\.sdc$|\.rpt$|metrics.*\.json$'
```

ตรวจ log:

```bash
rg -n -i \
  'spef|parasit|corner|unannotated|not.found|unmatched|error|warning' \
  "$RCX_RUN" \
  -g '*.log' -g '*.rpt'
```

**Acceptance เบื้องต้น**

- มี SPEF สำหรับ required extraction corners
- Extraction log ไม่มี unresolved errors
- STA โหลด SPEF ที่ตั้งใจใช้จริง
- ไม่มี unexplained net-name mismatch
- Required timing nets มี parasitic annotation ตามขอบเขตการวิเคราะห์

การมีไฟล์ SPEF อย่างเดียวไม่ได้พิสูจน์ว่า STA ใช้ SPEF นั้นครบ

---

### 10.14 ขั้นตอนที่ 11 — ตรวจโครงสร้าง SPEF

สำหรับ SPEF ที่ไม่บีบอัด:

```bash
export SPEF_FILE="<path-to-design.spef>"

rg -n \
  '^\*(SPEF|DESIGN|DIVIDER|DELIMITER|BUS_DELIMITER|T_UNIT|C_UNIT|R_UNIT|NAME_MAP)' \
  "$SPEF_FILE"

rg -c '^\*D_NET' "$SPEF_FILE"
```

ตรวจ:

| Field | จุดตรวจ |
|---|---|
| `*DESIGN` | ตรงกับ top-level design |
| `*T_UNIT` | Timing units ถูกต้อง |
| `*C_UNIT` | Capacitance units ถูกต้อง |
| `*R_UNIT` | Resistance units ถูกต้อง |
| Name delimiters | สอดคล้องกับ netlist naming |
| `*NAME_MAP` | การ map names ใช้งานได้ |
| `*D_NET` sections | มี extracted nets ตามขอบเขตที่คาดหวัง |
| `*CAP` / `*RES` | มีข้อมูลตาม extraction model |

ชื่อ net อาจแปลงเป็นหมายเลขผ่าน `*NAME_MAP` ดังนั้นค้นหาชื่อ clock net อย่างเดียวอาจไม่พบ section โดยตรง

**จำนวน `*D_NET` ไม่จำเป็นต้องเท่ากับจำนวน nets ใน netlist** เพราะ power nets, internal macro nets หรือ nets ที่อยู่นอกขอบเขต extraction อาจถูกจัดการต่างกัน ต้องใช้ annotation report และ methodology ประกอบ

สำหรับ SRAM macro ให้แยก:

- External interconnect parasitics: สกัดจาก top-level routing
- Internal macro timing: ใช้ macro timing model
- Internal physical extraction: อยู่ในขอบเขตการตรวจ macro ตาม methodology

ต้องหลีกเลี่ยงทั้งการนับซ้ำและการละเลย delay

---

### 10.15 ขั้นตอนที่ 12 — วิเคราะห์ post-route timing

เริ่มจาก reports ของ `OpenROAD.STAPostPNR` แล้วเปรียบเทียบกับ Lab 9

| Metric | Post-CTS | Post-GRT | Post-RCX |
|---|---:|---:|---:|
| Setup worst slack | … | … | … |
| Setup TNS | … | … | … |
| Hold worst slack | … | … | … |
| Violating endpoints | … | … | … |
| Max slew violations | … | … | … |
| Max capacitance violations | … | … | … |
| Clock latency/skew ของ paths ที่เลือก | … | … | … |

เปรียบเทียบใน **scenario เดียวกัน** ได้แก่ clock period, SDC, PVT/RC corner และ analysis settings

#### อ่าน critical path หลัง extraction

1. ตรวจ startpoint/endpoint
2. ตรวจ analysis corner
3. ตรวจ launch/capture clock paths
4. ตรวจ data cell delay
5. ตรวจ net delay
6. ตรวจ required time และ uncertainty
7. ตรวจ slack และ adjustment ที่ tool ใช้
8. เปิดเส้นทางนั้นใน GUI

หาก timing แย่ลง ให้ระบุว่าเกิดจาก:

- Wire resistance
- Increased capacitance
- Route detour
- Via count
- Clock insertion/skew
- Buffer/cell changes จาก repair
- Constraints หรือ corner mapping ที่เปลี่ยน

#### หาก setup/hold ไม่ผ่าน

ย้อนแก้ตามสาเหตุ:

| ปัญหา | จุดแก้ที่เป็นไปได้ |
|---|---|
| Long route/congestion | Placement/floorplan/routing |
| Logic delay สูง | Synthesis/RTL |
| Hold path สั้น | Timing repair บน data path |
| Clock imbalance | CTS และ sink distribution |
| Constraints ผิด | SDC แล้วรัน optimization/analysis ที่เกี่ยวข้องใหม่ |

**หลัง ECO ต้อง reroute, re-extract และ rerun STA** การแก้ netlist แล้วใช้ SPEF เก่าไม่ใช่ผล timing ของ design ใหม่

---

### 10.16 ขั้นตอนที่ 13 — ทำ finishing ตาม template

จัดทำ finishing checklist ก่อน stream-out:

| รายการ | วิธีตรวจ | ผล |
|---|---|---|
| Standard-cell fillers | Cell count และ layout inspection | … |
| Tap/endcap | Spacing/placement ตาม rules | … |
| Pad fillers/corners | Ring continuity และ orientation | … |
| Power ring/straps | Connectivity และ geometry | … |
| Seal ring | Template/PDK implementation | … |
| Dummy metal fill | Approved fill flow และ density checks | … |
| Layer mapping | Stream-out configuration | … |
| Macro GDS merge | Cell names และ hierarchy | … |

**ลำดับ fill กับ extraction ต้องกำหนดชัดเจน**

ถ้า finishing เพิ่มหรือเปลี่ยน metal geometry ที่ extraction/sign-off methodology ต้องพิจารณา ให้ทำ extraction และ timing analysis ใหม่ด้วยวิธีที่รองรับผลของ geometry นั้น

หาก extraction flow ไม่รวม post-fill effects ให้ระบุข้อจำกัดในรายงาน ไม่เรียกผล pre-fill ว่า final post-fill timing โดยอัตโนมัติ

---

### 10.17 ขั้นตอนที่ 14 — Stream-out GDS

ใช้ stream-out steps ของ flow/template พร้อม technology mapping ที่กำหนด

ตรวจ:

1. Top cell ถูกต้อง
2. Standard-cell GDS ครบ
3. Macro GDS ครบ
4. Pad cells/fillers/corners ครบ
5. ไม่มี unresolved cell references
6. Layer/datatype mapping ถูกต้อง
7. Units และ die dimensions ถูกต้อง
8. Seal ring/fill อยู่ในขอบเขตที่ต้องการ

หาก flow สร้าง GDS ผ่านหลาย engines ให้ใช้การเปรียบเทียบ/XOR ตามที่ PDK รองรับและ methodology กำหนด ไม่สมมติว่า engine ทุกตัวรองรับ SG13G2 revision เดียวกัน

**GDS ที่เปิดดูได้ยังไม่พิสูจน์ว่า DRC/LVS ผ่าน**

---

### 10.18 ขั้นตอนที่ 15 — ตรวจ GDS และ verification reports

ตรวจ GDS ด้วย KLayout:

- Top-level hierarchy
- Die/core dimensions
- Pad ring
- SRAM/macros
- Routing layers/vias
- Seal ring
- Fill geometry
- Missing/empty cells
- Unexpected geometry นอก die

แยก checks:

| Check | ตรวจอะไร | ไม่ทดแทน |
|---|---|---|
| Router DRC | กฎที่ router ตรวจได้ | Foundry/PDK GDS DRC ทั้งหมด |
| GDS DRC | Geometry ตาม deck ที่ใช้ | LVS และ timing |
| LVS | Layout connectivity เทียบ schematic/netlist | Timing และ density |
| Antenna | Process antenna rules | DRC/LVS ทั้งหมด |
| Density | Density windows/rules | Connectivity |
| STA | Timing ภายใต้ scenarios ที่กำหนด | Physical verification |
| PDN/IR checks | Power integrity ภายใต้ assumptions | Signal STA |

ให้บันทึก deck revision และ options ของแต่ละ check โดยเฉพาะ full-chip, macro exclusions และ hierarchical checking

**“ไม่ได้รัน” ต้องระบุเป็น `NOT RUN` ไม่บันทึกเป็น `PASS`**

---

### 10.19 ขั้นตอนที่ 16 — จัดชุด artifacts สำหรับส่งต่อ

เก็บ artifacts จาก checkpoint ที่สัมพันธ์กัน:

| Artifact | จุดประสงค์ |
|---|---|
| Final ODB | Physical database |
| Routed DEF | Placement/routing exchange |
| Updated logical netlist | วงจรหลัง implementation |
| Powered netlist | Supply-aware reference ตาม flow |
| SDC | Constraints ที่ใช้วิเคราะห์ |
| SPEF ทุก required corner | Extracted parasitics |
| GDS | Layout สำหรับ final verification |
| STA reports | Timing evidence |
| DRC/LVS/antenna/density reports | Physical verification evidence |
| Resolved configuration | Reproducibility |
| Tool/PDK/source revisions | Provenance |

ตัวอย่างจัดโฟลเดอร์:

```bash
mkdir -p deliverables/lab10/layout
mkdir -p deliverables/lab10/netlist
mkdir -p deliverables/lab10/timing
mkdir -p deliverables/lab10/reports
mkdir -p deliverables/lab10/provenance
```

คัดลอกเฉพาะ artifacts ที่ตรวจแล้วว่าเป็นชุดเดียวกัน แล้วสร้าง checksum:

```bash
python3 - <<'PY'
from pathlib import Path
import hashlib

root = Path("deliverables/lab10")
manifest = root / "SHA256SUMS"

lines = []
for path in sorted(root.rglob("*")):
    if not path.is_file() or path == manifest:
        continue
    digest = hashlib.sha256()
    with path.open("rb") as stream:
        for block in iter(lambda: stream.read(1024 * 1024), b""):
            digest.update(block)
    lines.append(f"{digest.hexdigest()}  {path.relative_to(root).as_posix()}")

manifest.write_text("\n".join(lines) + "\n", encoding="utf-8")
PY
```

Checksum ยืนยันว่า bytes ไม่เปลี่ยน แต่ไม่ได้ยืนยันความถูกต้องทางวิศวกรรม ต้องใช้ reports และ provenance ประกอบ

---

### 10.20 ปัญหาที่พบบ่อย

| อาการ | สิ่งที่ตรวจอันดับแรก | แนวทาง |
|---|---|---|
| Global routing overflow สูง | Hotspot, density, macro channels | แก้ floorplan/placement |
| Detailed routing ค้างที่ violations | Marker locations และ pin access | แก้สาเหตุ geometry ก่อนเพิ่ม iterations |
| Antenna repair ไม่ครบ | Properties, diode availability, affected nets | ตรวจ PDK views และ repair flow |
| Disconnected pad pins | Net names, pad models, physical connections | ตรวจ pin-level mapping |
| SPEF ไม่มี clock nets ที่คาดหวัง | Name mapping และ extraction scope | ตรวจ annotation ใน STA |
| STA โหลด SPEF ไม่ตรง | Netlist/checkpoint และ naming | ใช้ artifacts จาก run เดียวกัน |
| Post-route hold ล้มเหลว | Min paths และ clock skew | Repair แล้ว reroute/re-extract |
| GDS ขาด SRAM | Macro GDS paths และ cell names | แก้ stream-out merge |
| Router DRC ผ่าน แต่ GDS DRC ไม่ผ่าน | Rules ที่ต่างกัน, macro boundaries, finishing | แก้ตาม GDS deck |
| LVS ไม่ตรงหลัง finishing | Reference netlist และ physical-cell handling | ตรวจ methodology และ cell views |
| Density ไม่ผ่านหลัง filler insertion | ชนิด fill ที่ใช้งาน | ใช้ metal-fill flow ที่ถูกต้อง |
| ECO แล้ว timing ดูผิดปกติ | SPEF เก่าและ artifact mismatch | Re-extract และ STA ใหม่ |

---

### 10.21 เกณฑ์ผ่าน Lab

| Gate | Acceptance |
|---|---|
| Routing completeness | Required connections เดินสายครบ |
| Router DRC | ไม่มี unresolved violations |
| Connectivity | ไม่มี unexplained disconnected required pins/nets |
| Antenna | ผ่านขอบเขตและ rules ที่กำหนด |
| Extraction | Required RC models/corners ครบและไม่มี unresolved errors |
| SPEF annotation | ไม่มี unexplained missing/mismatched timing-net annotation |
| Post-route setup/hold | ผ่านทุก required scenario หรือระบุเป็นปัญหาคงค้าง |
| Electrical timing checks | ไม่มี unresolved slew/capacitance violations |
| Finishing | ทำครบตาม PDK/template methodology |
| Stream-out | GDS ครบ ถูก top cell และไม่มี missing references |
| Verification status | ระบุ PASS/FAIL/NOT RUN/WAIVED พร้อมหลักฐาน |
| Artifact consistency | ODB/DEF/netlist/SPEF/GDS มี provenance สอดคล้องกัน |

หากมี approved waiver ให้บันทึก rule, location, เหตุผล, ผู้อนุมัติ และขอบเขตที่ครอบคลุม ไม่ใช้ waiver แทนการตรวจที่ยังไม่ได้รัน

---

### 10.22 งานที่ต้องส่ง

1. Routed ODB และ DEF
2. Updated netlists และ SDC
3. SPEF ทุก required extraction corner
4. Post-route STA reports และ critical-path analysis
5. GDS หลัง finishing ที่กำหนด
6. Routing/antenna/connectivity reports
7. DRC/LVS/density status matrix
8. ภาพ layout และตัวอย่างปัญหาก่อน–หลังแก้
9. Artifact manifest และ checksums
10. รายการข้อจำกัดและงานที่ต้องตรวจใน Lab ถัดไป

แบบฟอร์มสรุป:

```text
Design/top cell:
Source revision:
PDK/template revision:
LibreLane/OpenROAD version:
Input checkpoint:
Final routed checkpoint:

Routing completeness:
Router DRC:
Connectivity:
Antenna:
Extraction models/corners:
SPEF annotation:
Worst setup slack/scenario:
Worst hold slack/scenario:
Electrical violations:

Filler/tap/endcap status:
Pad-ring finishing:
Seal-ring status:
Metal-fill/density status:
GDS DRC:
LVS:

Final artifacts:
Known limitations:
Remaining actions:
```

### 10.23 คำถามท้าย Lab

1. Global routing guides ต่างจาก detailed routing geometry อย่างไร?
2. เหตุใด total congestion ต่ำจึงยังเกิด pin-access failure ได้?
3. Router DRC ผ่านแล้วเหตุใดต้องตรวจ GDS DRC อีก?
4. PVT corner ต่างจาก RC extraction corner อย่างไร?
5. เหตุใดไฟล์ SPEF มีอยู่จึงยังไม่พิสูจน์ว่า STA annotation ครบ?
6. Standard-cell filler ต่างจาก dummy metal fill อย่างไร?
7. หลัง ECO เหตุใดจึงต้อง reroute และ re-extract?
8. Post-fill geometry อาจกระทบ parasitics อย่างไร?
9. GDS เปิดดูได้ต่างจาก layout ที่พร้อม sign-off อย่างไร?
10. จะพิสูจน์อย่างไรว่า GDS, netlist และ SPEF เป็น design revision เดียวกัน?