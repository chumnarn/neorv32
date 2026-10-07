## Lab 9 — Placement, CTS และ timing closure

Lab นี้นำผลจาก **Lab 8 — Floorplan และ PDN** มาจัดวาง standard cells สร้าง clock tree และปรับปรุง timing ของ NEORV32 บน IHP SG13G2 ก่อนเข้าสู่ detailed routing

**เป้าหมายหลัก:** ได้ placement ที่ถูกต้องตามกฎทางกายภาพ มี clock tree ที่เชื่อมต่อครบ และมีหลักฐานตรวจ setup/hold ทุก corner ที่กำหนด โดยเข้าใจว่าผลหลัง CTS ยังใช้ parasitic estimation จึงต้องยืนยันอีกครั้งหลัง routing และ extraction

> **ขอบเขตของตัวอย่าง:** ชื่อไฟล์ configuration, run directory และ clock ให้ปรับตามโครงการจริง ตัวอย่างใช้ `librelane/config.json` และ clock period 10 ns เพื่ออธิบายวิธีทำงาน ไม่ได้ยืนยันว่า NEORV32 configuration ของอาจารย์จะผ่าน 100 MHz

### 9.1 วัตถุประสงค์การเรียนรู้

เมื่อจบ Lab ผู้เรียนควรสามารถ:

1. ตรวจความพร้อมของ floorplan/PDN ก่อน placement
2. แยก global placement, detailed placement และ timing optimization
3. อ่าน timing report และแยกปัญหา logic delay, wire delay และ clock skew
4. ตรวจ clock tree หลัง CTS ทั้ง connectivity และ timing
5. แก้ setup/hold โดยเลือกจุดแก้ตามสาเหตุ
6. เปรียบเทียบผลหลายรอบด้วย constraints และ corner ชุดเดียวกัน
7. เตรียม checkpoint และรายงานส่งต่อให้ routing

---

### 9.2 ทำความเข้าใจลำดับการทำงาน

ลำดับที่เกี่ยวข้องใน LibreLane Classic flow ได้แก่ global placement, design repair, detailed placement, CTS, STA และ timing repair หลัง CTS จากนั้นเข้าสู่ global routing และการตรวจเพิ่มเติม ชื่อและลำดับจริงต้องตรวจจาก flow ของรุ่นที่ติดตั้ง เพราะโครงการอาจใช้ custom flow จาก template [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/reference/flows.html?utm_source=chatgpt.com)

| ช่วง | หน้าที่ | สิ่งที่ต้องตรวจ |
|---|---|---|
| Global placement | จัดตำแหน่ง cells โดยพิจารณาการเชื่อมต่อและพื้นที่ | ความหนาแน่น, wirelength, hotspot |
| Design repair | ปรับขนาด cells หรือเพิ่ม buffers เพื่อแก้ electrical violations | Slew, capacitance, fanout |
| Detailed placement | ย้าย cells ลงตำแหน่ง legal บน rows/sites | Overlap, orientation, displacement |
| CTS | สร้างเครือข่ายกระจาย clock | Sinks, latency, skew, clock slew |
| Post-CTS timing repair | ปรับ data paths หลังมี clock tree | Setup, hold, area overhead |
| Global routing assessment | ประเมินเส้นทางเดินสายและ parasitics ให้ใกล้ความจริงขึ้น | Congestion และ timing ที่เปลี่ยนไป |

**Global placement ผ่านไม่ได้แปลว่า placement legal แล้ว** ส่วน detailed placement มีหน้าที่ทำให้ตำแหน่ง cells ถูกต้องตาม rows และ sites โดยพยายามลดการย้ายตำแหน่งจากผล global placement [OpenROAD documentation](https://openroad.readthedocs.io/en/latest/main/src/dpl/README.html?utm_source=chatgpt.com)

---

### 9.3 ข้อมูลที่ต้องเตรียมจาก Lab ก่อนหน้า

| ข้อมูล | ต้นทาง | จุดตรวจ |
|---|---|---|
| Technology-mapped netlist | Lab 5 | ไม่มี unresolved logic ที่ต้องนำไปวาง |
| Pad instances และ pin mapping | Lab 6 | ตำแหน่งและ orientation ถูกต้อง |
| PNR SDC | Lab 7 | Clock, I/O delays และ exceptions ครบ |
| Floorplan database | Lab 8 | Rows, core area และ macros ถูกต้อง |
| PDN | Lab 8 | Connectivity และ geometry ผ่านเกณฑ์ Lab 8 |
| Liberty/LEF | Environment/PDK | ตรงกับ PDK revision ที่ล็อกไว้ |
| Timing corner list | Lab 7 | มี mapping ระหว่าง corner และ libraries ชัดเจน |

สำหรับ full-chip ต้องตรวจเพิ่มเติมว่า:

- Pad ring และ macros ที่ตั้งใจให้ fixed มีสถานะ fixed จริง
- Clock port ผ่าน input pad ไปถึง clock network ของ core
- Core supplies และ I/O supplies เชื่อมตาม schematic/PDK
- Placement ไม่ใช้พื้นที่ pad ring หรือ macro obstruction
- หาก memory ถูก synthesize เป็น standard cells ต้องนับพื้นที่และ routing demand ตาม implementation จริง

**ห้ามแก้ปัญหา clock ไม่เข้าถึง core ด้วยการสร้าง clock ใหม่ภายในโดยไม่มีเหตุผลด้าน timing methodology** เพราะอาจทำให้ช่วง clock pad-to-core หลุดจากการวิเคราะห์

---

### 9.4 ขั้นตอนที่ 1 — บันทึก baseline และตรวจเครื่องมือ

ทำงานใน environment เดียวกับ Lab 8:

```bash
mkdir -p reports/lab9

python3 -m librelane --help \
  > reports/lab9/librelane-help.txt

python3 - <<'PY' > reports/lab9/tool-version.txt
from importlib.metadata import version
print("LibreLane:", version("librelane"))
PY

openroad -version \
  >> reports/lab9/tool-version.txt

git rev-parse HEAD \
  > reports/lab9/project-commit.txt

git status --short \
  > reports/lab9/project-status.txt
```

หากโครงการไม่ได้อยู่ใน Git ให้บันทึก source hashes แทน

ตรวจ options ของ CLI:

```bash
rg -n -- \
  '--from|--to|--with-initial-state|--run-tag' \
  reports/lab9/librelane-help.txt
```

ตรวจ configuration ที่เกี่ยวข้อง:

```bash
rg -n \
  'CLOCK|SDC|CORNER|DENSITY|CTS|RESIZER|DIE_AREA|CORE_AREA' \
  librelane/config.json
```

**จุดตรวจ**

- `CLOCK_PERIOD` ตรงกับ `create_clock` ใน SDC
- `CLOCK_PORT` ตรงกับ port ของ top-level จริง
- PNR ใช้ SDC ที่ผ่านการตรวจใน Lab 7
- ไม่มี deprecated/unknown configuration warning ที่ยังไม่ได้ประเมิน
- Libraries และ RC settings เป็นของ IHP SG13G2

> บางค่าได้รับผ่าน PDK configuration หรือ template จึงไม่ปรากฏใน `config.json` ต้องตรวจ resolved configuration และ log ของ run ด้วย

---

### 9.5 ขั้นตอนที่ 2 — เลือก checkpoint จาก Lab 8

กำหนด run ที่ตรวจผ่านแล้ว:

```bash
export LAB8_RUN="librelane/runs/<ชื่อ-run-Lab8>"

test -d "$LAB8_RUN" || {
  echo "Lab 8 run directory not found"
  exit 1
}

rg --files "$LAB8_RUN" \
  | rg 'state_out\.json$|metrics.*\.json$|\.odb$|\.def$'
```

เลือก `state_out.json` จาก step **สุดท้ายก่อน global placement** ของ flow จริง ไม่เลือกจากชื่อ PDN เพียงอย่างเดียว เพราะบาง flow มี step เตรียม database หลัง PDN อีกหลายขั้น

```bash
export LAB8_STATE="$LAB8_RUN/<step-สุดท้ายก่อน-placement>/state_out.json"

test -f "$LAB8_STATE" || {
  echo "Checkpoint not found"
  exit 1
}
```

ก่อน resume ให้ตรวจว่า database, netlist, SDC และ macro views ใน checkpoint ยังเข้าถึงได้

**กฎเลือก checkpoint**

| สิ่งที่เปลี่ยน | ต้องเริ่มใหม่จาก |
|---|---|
| RTL, memory configuration หรือ synthesis mapping | Synthesis หรือก่อนหน้านั้น |
| Core area, macro position, pad location หรือ PDN | Floorplan และขั้นที่ได้รับผลกระทบ |
| Placement density | ก่อน global placement |
| CTS parameters | ก่อน CTS |
| Post-CTS repair parameters | ก่อน timing repair |
| SDC | ก่อน optimization ที่ต้องใช้ constraint ใหม่ พร้อมยืนยันว่าโหลด SDC ใหม่จริง |

การ resume จาก database เดิมโดยไม่ตรวจ provenance อาจทำให้รายงานอ้างถึง netlist หรือ SDC คนละรุ่นกับที่ตั้งใจใช้

---

### 9.6 ขั้นตอนที่ 3 — รัน placement baseline

เมื่อ CLI รุ่นที่ใช้รองรับ options ต่อไปนี้ และ flow มี step names ตามตัวอย่าง:

```bash
python3 -m librelane \
  --run-tag lab9_place_baseline \
  --with-initial-state "$LAB8_STATE" \
  --from OpenROAD.GlobalPlacement \
  --to OpenROAD.DetailedPlacement \
  librelane/config.json
```

หาก custom flow ใช้ชื่อหรือขอบเขตต่างกัน ให้ใช้ชื่อจาก flow นั้น

หลังรัน ให้กำหนด path ของ run ตามผลจริง:

```bash
export PLACE_RUN="librelane/runs/lab9_place_baseline"

rg --files "$PLACE_RUN" \
  | rg 'globalplacement|detailedplacement|repairdesign|metrics'
```

ค้นหาปัญหา:

```bash
rg -n -i \
  'error|warning|overflow|density|congestion|overlap|unplaced|illegal' \
  "$PLACE_RUN" \
  -g '*.log' -g '*.rpt'
```

#### สิ่งที่ต้องอ่านจาก global placement

1. จำนวน standard cells และพื้นที่รวม
2. Target density และผลการกระจาย cells
3. Placement overflow/convergence
4. Estimated wirelength
5. ปัญหา high-fanout nets
6. Timing ก่อนและหลัง design repair หาก flow มีรายงาน

**อย่าสับสน placement overflow กับ routing overflow** ค่าแรกเกี่ยวกับการกระจายพื้นที่ cells ส่วนค่าหลังเกี่ยวกับ demand เทียบกับ routing capacity

#### สิ่งที่ต้องอ่านจาก detailed placement

- จำนวน unplaced instances
- Illegal overlap
- การวางตรง rows/sites
- Macro/pad overlap
- Cells ที่ย้ายไกลจาก global placement

**Acceptance ของช่วง placement**

- ไม่มี unplaced standard cells ที่ต้องใช้งาน
- ไม่มี illegal overlaps
- Fixed pads/macros อยู่ตำแหน่งที่กำหนด
- รายงาน legalization ไม่มี unresolved errors
- PDN/connectivity checks ที่ได้รับผลกระทบยังผ่าน

---

### 9.7 ขั้นตอนที่ 4 — ตรวจ placement ด้วย GUI

เปิด ODB หลัง detailed placement ด้วยวิธี GUI ที่โครงการใช้ โดยโหลด technology และ LEF ให้ครบ

ตรวจทีละบริเวณ:

| บริเวณ | สิ่งที่ตรวจ |
|---|---|
| ขอบ core | Cells เบียด pad ring หรือ PDN ring หรือไม่ |
| รอบ SRAM/macros | ช่องว่างและ pin access เพียงพอหรือไม่ |
| ใจกลาง core | มี cell density hotspot หรือไม่ |
| กลุ่ม CPU datapath | ตำแหน่ง endpoints ของ critical paths |
| Memory interface | Bus ยาวหรือกระจายข้าม core หรือไม่ |
| Clock/reset distribution | Nets fanout สูงและระยะทางกระจาย sinks |

สำหรับ NEORV32 ให้ใช้ timing report ระบุชื่อ instance จริงก่อนเลือกดู ALU, register file, load/store หรือ memory logic เพราะ hierarchy อาจเปลี่ยนหลัง export VHDL และ synthesis

**การทดลอง density**

ตัวอย่างค่าที่ใช้ทดลองใน Lab:

```json
"PL_TARGET_DENSITY_PCT": 35
```

ให้ทดลอง 35%, 40% และ 45% โดยคง floorplan, RTL, SDC และ corner เดิม ตัวแปรในเอกสาร LibreLane ปัจจุบันใช้ชื่อ `PL_TARGET_DENSITY_PCT`; ตรวจชื่อและหน่วยกับรุ่นที่ติดตั้งก่อนใช้ [librelane.readthedocs.io](https://librelane.readthedocs.io/en/stable/reference/step_config_vars.html?utm_source=chatgpt.com)

ค่าชุดนี้เป็นตัวอย่างทดลอง ไม่ใช่ค่าที่รับประกันว่าเหมาะกับทุก NEORV32 configuration

- Density ต่ำอาจช่วยให้มีพื้นที่แทรก buffers แต่เพิ่ม wirelength
- Density สูงอาจลดระยะทางบาง paths แต่เพิ่ม congestion
- Density ที่เหมาะสมต้องประเมินร่วมกับพื้นที่ macros, PDN obstructions และ routing capacity

---

### 9.8 ขั้นตอนที่ 5 — วิเคราะห์ timing ก่อน CTS

ก่อน CTS clock network ยังไม่มีรูปแบบสุดท้าย ผล setup จึงใช้เพื่อระบุ data-path bottlenecks ส่วน hold ต้องตรวจอีกครั้งหลัง clock propagation

ให้รวบรวม critical paths อย่างน้อย 10 paths ต่อ path group และ corner ที่สำคัญ:

| ข้อมูล | ใช้ตอบคำถาม |
|---|---|
| Startpoint/endpoint | ปัญหาอยู่ระหว่าง registers หรือ I/O |
| Clock/path group | เป็น domain ใด |
| Corner | ตรวจที่ operating condition ใด |
| Cell delay | Logic depth หรือ drive strength เป็นปัญหาหรือไม่ |
| Net delay | Placement/wirelength เป็นปัญหาหรือไม่ |
| Fanout/capacitance | โหลดมากเกินไปหรือไม่ |
| Required time | Constraint มีเหตุผลหรือไม่ |
| Slack | ขาด timing budget เท่าไร |

ใน OpenROAD/OpenSTA session ที่โหลด design, Liberty, SDC และ parasitic model ถูกต้องแล้ว สามารถใช้คำสั่งรายงาน:

```tcl
report_checks \
  -path_delay max \
  -format full_clock_expanded \
  -group_path_count 10

report_checks \
  -path_delay min \
  -format full_clock_expanded \
  -group_path_count 10

report_worst_slack -max
report_worst_slack -min
report_tns
```

ต้องตรวจว่าคำสั่งและ options รองรับใน OpenSTA รุ่นที่ติดตั้ง

> เปิด ODB อย่างเดียวไม่เพียงพอสำหรับ STA เพราะต้องมี timing libraries, constraints และ parasitic model ด้วย วิธีที่ปลอดภัยสำหรับ Lab คือเริ่มจาก reports และ loading scripts ที่ flow สร้าง

#### วิธีจัดประเภท critical path

**กรณี A: Cell delay สูง**

ตรวจ:

- Logic depth
- Mux chain
- Arithmetic logic
- Register-file read selection
- Cell drive strength
- Memory ที่ถูกแปลงเป็น logic

แนวทางแก้: ปรับ synthesis/RTL, เลือก implementation ของ memory หรือพิจารณา pipeline โดยรักษาข้อกำหนดการทำงาน

**กรณี B: Net delay สูง**

ตรวจ:

- Endpoints อยู่ไกลกัน
- Fanout สูง
- Macro บังคับให้เดินสายอ้อม
- Congestion ทำให้ route ยาว

แนวทางแก้: placement, macro location, buffering และพื้นที่สำหรับ routing

**กรณี C: Required time ผิดปกติ**

กลับไปตรวจ SDC ก่อนปรับ physical design เช่น clock period, I/O delays, clock relationship และ exceptions

---

### 9.9 ขั้นตอนที่ 6 — เตรียม CTS

ก่อน CTS ต้องตอบคำถามต่อไปนี้ได้:

1. Clock root อยู่ที่ port/pin ใด?
2. Clock ผ่าน pad หรือ buffer ใดก่อนเข้า core?
3. มี clocks กี่ชุด?
4. มี generated clock หรือ clock-gating cells หรือไม่?
5. CTS ใช้ buffer cells ใดจาก PDK?
6. Clock sinks ที่คาดหวังมีกี่ตัว?
7. มี sequential cells ที่ไม่มี clock หรือไม่?

จำนวน CTS sinks อาจไม่เท่ากับจำนวน flip-flops ทั้งหมด เนื่องจาก gating, macros หรือหลาย clock domains จึงต้องอธิบายส่วนต่างได้

**เริ่มจากค่า CTS ที่ PDK/template รองรับ** และตรวจว่า:

- Buffer cells มี Liberty และ LEF ที่ตรงกัน
- Cell names อยู่ใน library จริง
- ไม่มี clock cell ที่ถูกห้ามใช้โดยไม่ได้ตั้งใจ
- Clock RC settings ใช้ routing layers ของ SG13G2
- Clock path ผ่าน I/O pad มี timing model หากอยู่ในขอบเขตการวิเคราะห์

CTS ของ LibreLane รับ database ที่ผ่าน detailed placement และมีการทำ detailed placement อีกครั้งเพื่อรองรับ cells ใหม่ [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/reference/step_config_vars.html?utm_source=chatgpt.com)

---

### 9.10 ขั้นตอนที่ 7 — รัน CTS และ timing repair

เลือก checkpoint หลัง detailed placement:

```bash
export PLACE_STATE="$PLACE_RUN/<step-detailed-placement>/state_out.json"

test -f "$PLACE_STATE" || {
  echo "Placement checkpoint not found"
  exit 1
}
```

รัน:

```bash
python3 -m librelane \
  --run-tag lab9_cts_baseline \
  --with-initial-state "$PLACE_STATE" \
  --from OpenROAD.CTS \
  --to OpenROAD.ResizerTimingPostCTS \
  librelane/config.json
```

หากต้องการ STA report หลัง repair ให้รันต่อถึง STA step ที่ตามหลัง repair ใน flow จริง แทนการหยุดที่ repair เพียงอย่างเดียว

ค่าที่เกี่ยวข้องใน Classic flow:

```json
"RUN_CTS": true,
"RUN_POST_CTS_RESIZER_TIMING": true
```

LibreLane มี timing optimization หลัง CTS และมีตัวเลือกเพิ่มเติมหลัง global routing; ต้องตรวจการเปิดใช้จาก resolved configuration จริง [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/usage/timing_closure/index.html?utm_source=chatgpt.com)

---

### 9.11 ขั้นตอนที่ 8 — ตรวจ clock tree หลัง CTS

#### A. Connectivity

- Clock sinks ที่ต้องใช้งานได้รับ clock ครบ
- ไม่มี floating clock pins
- ไม่มี disconnected clock branches
- Clock-gating output และ macro clock pins ถูกจัดการตาม methodology
- Clock input pad เชื่อมกับ root ของ core clock tree

#### B. Physical correctness

- Clock buffers legal และไม่มี overlap
- ไม่มี buffer อยู่ใน obstruction
- Power pins ของ cells ใหม่ได้รับ supply
- Placement และ connectivity checks หลัง CTS ผ่าน

#### C. Clock quality

บันทึก:

- จำนวน clock buffers
- จำนวน sinks
- Tree depth หากมีรายงาน
- Insertion delay/latency
- Skew
- Clock slew
- Clock capacitance violations

**Skew ต่ำไม่ได้รับประกัน timing ผ่าน** ต้องตรวจ launch/capture clock paths ของ timing paths จริง และแยกค่ารวมของ tree ออกจาก path-specific skew

#### D. STA clock model

ตรวจว่า STA หลัง CTS ใช้ propagated clocks ตาม methodology ของ flow

สำหรับการตรวจใน session แยก คำสั่ง:

```tcl
set_propagated_clock [all_clocks]
```

ทำให้ clock delays ถูกวิเคราะห์ผ่าน network ที่มีอยู่ แต่ต้องโหลด design และ parasitic model ถูกต้องก่อน คำสั่งนี้ไม่ได้สร้าง clock tree หรือเติม parasitics ให้เอง

---

### 9.12 ขั้นตอนที่ 9 — คำนวณ setup และ hold จากรายงาน

สำหรับ single-cycle path ใน clock domain เดียวกัน กำหนด:

- $$L$$: Clock arrival time ที่ launch register
- $$C$$: Clock arrival time ที่ capture register
- $$T$$: Clock period
- $$D_{\max}$$, $$D_{\min}$$: Data-path delay รวม clock-to-Q
- $$t_{\text{setup}}$$, $$t_{\text{hold}}$$: Sequential-cell timing checks
- $$U_s$$, $$U_h$$: Setup/hold uncertainty

แบบจำลองพื้นฐาน:

$$Slack_{\text{setup}} = T+(C-L)-D_{\max}-t_{\text{setup}}-U_s$$

$$Slack_{\text{hold}} = D_{\min}-(C-L)-t_{\text{hold}}-U_h$$

สมการนี้ยังไม่แสดงรายละเอียด derating, clock reconvergence adjustment และ edge relationships ต้องใช้ STA report เป็นคำตอบอ้างอิงสำหรับ design จริง

#### ตัวอย่าง setup

กำหนด:

- $$T=10.00$$ ns
- $$L=0.80$$ ns
- $$C=0.95$$ ns
- $$D_{\max}=9.70$$ ns
- $$t_{\text{setup}}=0.10$$ ns
- $$U_s=0.25$$ ns

$$
Slack_{\text{setup}} = 10.00+0.15-9.70-0.10-0.25 = 0.10\ \text{ns}
$$

ผ่านในเงื่อนไขตัวอย่างนี้

#### ตัวอย่าง hold

ใช้:

- $$D_{\min}=0.12$$ ns
- $$C-L=0.15$$ ns
- $$t_{\text{hold}}=0.05$$ ns
- $$U_h=0.03$$ ns

$$Slack_{\text{hold}} = 0.12-0.15-0.05-0.03 = -0.11\ \text{ns}$$

ต้องเพิ่ม data-path delay อย่างน้อย 0.11 ns สำหรับเงื่อนไขนี้ และตรวจใหม่ทุก corner หลังแก้

**ข้อสังเกต:** Capture clock ที่มาช้ากว่า launch clock ช่วย setup แต่ทำให้ hold ยากขึ้น ส่วนการเพิ่ม clock period โดยทั่วไปไม่ได้แก้ same-edge hold violation

---

### 9.13 ขั้นตอนที่ 10 — แก้ setup violations

ทำตามลำดับ:

1. **ยืนยัน constraint**  
   ตรวจ clock, path group, I/O budget และ exceptions

2. **เลือก path ที่เป็นปัญหาจริง**  
   ดูหลาย endpoints ไม่ดูเฉพาะ worst path เดียว

3. **แยก cell delay และ net delay**  
   ใช้ incremental delay ในรายงาน ไม่ประเมินจากจำนวน gates อย่างเดียว

4. **เลือกการแก้ตามสาเหตุ**

| สาเหตุ | แนวทางแก้ | ผลกระทบที่ต้องตรวจ |
|---|---|---|
| Cell drive อ่อน | Resize cell | Area และ input capacitance เพิ่ม |
| Fanout สูง | Buffer หรือแบ่งโหลด | Area, routing และ hold |
| Wire ยาว | ปรับ placement/floorplan | Legalization และ congestion |
| Logic depth มาก | ปรับ RTL/synthesis | Functional equivalence |
| Memory logic ใหญ่ | ประเมิน macro/architecture | Interface, latency และ mapping |
| Timing budget ไม่สมจริง | ทบทวน specification | ต้องมีเหตุผลและอนุมัติการเปลี่ยนเป้าหมาย |

5. **รัน STA ทุก required corner ใหม่**
6. **ตรวจ hold และ electrical violations**
7. **บันทึก area/buffer overhead**

OpenROAD resizer รองรับการปรับขนาด cells และ buffering เพื่อซ่อม design/timing แต่ผลลัพธ์ขึ้นกับ constraints, libraries และพื้นที่ที่มีอยู่ [OpenROAD documentation](https://openroad.readthedocs.io/en/latest/main/src/rsz/README.html?utm_source=chatgpt.com)

**ห้ามใส่ false path หรือ multicycle path เพื่อทำให้รายงานผ่านโดยไม่มีหลักฐานจากการทำงานของวงจร**

---

### 9.14 ขั้นตอนที่ 11 — แก้ hold violations

หลัง CTS ให้ตรวจ min-delay paths ทุก required corner

1. เลือก worst hold paths
2. อ่าน launch/capture clock arrival times
3. ตรวจว่า violation เกิดจาก data path สั้นหรือ skew สูง
4. ตรวจ input `-min` delay หากเป็น input-to-register path
5. ใช้ timing repair เพิ่ม delay บน data branch ที่เหมาะสม
6. ตรวจ setup ใหม่หลังเพิ่ม buffers
7. ตรวจ placement และ supplies ของ cells ที่เพิ่ม
8. ตรวจ corner อื่นและ paths ที่ใช้ branch ร่วมกัน

| สาเหตุ | แนวทาง |
|---|---|
| Data path สั้นมาก | เพิ่ม delay/buffer บน data branch |
| Capture clock มาช้าเกินไป | ตรวจ tree imbalance และ sink grouping |
| Input min delay ผิด | แก้ตาม external-interface specification |
| Clock relationship ผิด | แก้ clocks/generated clocks |
| Hold repair แทรก cells ไม่ได้ | ตรวจพื้นที่ว่าง, legalization และ repair limits |

**ไม่ควรเพิ่ม clock uncertainty เพื่อซ่อม hold** เพราะ uncertainty เป็น timing budget ที่ต้องมีที่มา และอาจทำให้ constraints เข้มขึ้น

สำหรับ asynchronous reset ต้องคงการตรวจ recovery/removal และ reset-release methodology ที่กำหนดใน Lab 7 ไม่ยกเว้นทุก reset path โดยอัตโนมัติ

---

### 9.15 ขั้นตอนที่ 12 — ประเมินหลัง global routing

ผลหลัง placement/CTS เป็น checkpoint ระหว่างทาง การประเมินด้วย global routing ช่วยเห็น routing demand และผล timing เมื่อประมาณ parasitics จากเส้นทางที่ใกล้ implementation มากขึ้น

รันต่อจาก checkpoint ก่อน global routing ตาม flow ของโครงการ:

```bash
python3 -m librelane \
  --run-tag lab9_grt_assessment \
  --with-initial-state "<checkpoint-ก่อน-global-routing>/state_out.json" \
  --from OpenROAD.GlobalRouting \
  --to "<STA-step-หลัง-global-routing>" \
  librelane/config.json
```

ชื่อ STA step ให้เลือกจาก flow จริง ไม่อ้างอิงเลข stage เพราะเปลี่ยนได้ตามรุ่นและ configuration

ตรวจ:

- Routing overflow/congestion ต่อ layer
- Unroutable regions หรือ macro pin-access problems
- Setup/hold ที่แย่ลง
- Max slew/max capacitance
- Area และ buffer count หลัง repair

หาก post-CTS ผ่านแต่ global-route timing ไม่ผ่าน ให้จัดสถานะ **ต้องแก้ต่อ** และวิเคราะห์ว่า wirelength, detour หรือ congestion เพิ่ม delay ส่วนใด

---

### 9.16 ขั้นตอนที่ 13 — ทดลอง timing closure อย่างมีระบบ

เปลี่ยนทีละกลุ่มปัจจัยและใช้ชื่อ run แยกกัน:

| Run | สิ่งที่เปลี่ยน | สิ่งที่คงเดิม |
|---|---|---|
| R0 | Baseline | RTL/SDC/PDK/corners |
| R1 | Placement density | RTL/SDC/floorplan/corners |
| R2 | Floorplan หรือ macro position | RTL/SDC/corners |
| R3 | CTS parameters | Placement checkpoint/SDC/corners |
| R4 | Timing repair parameters | Pre-repair checkpoint/SDC/corners |

บันทึกผล:

| Run | Stage | Corner | Setup WNS | Setup TNS | Hold WNS | Violating endpoints | Area | Clock buffers | Congestion |
|---|---|---|---:|---:|---:|---:|---:|---:|---|
| R0 | Post-CTS | … | … | … | … | … | … | … | … |
| R0 | Post-GRT | … | … | … | … | … | … | … | … |

WNS บอก path ที่แย่ที่สุด ส่วน TNS และจำนวน violating endpoints ช่วยบอกขอบเขตปัญหา ต้องระบุว่าแต่ละ metric เป็นค่าเฉพาะ corner หรือค่ารวมจาก flow

**หลักเลือก run:** เลือก run ที่ผ่านข้อกำหนดครบและมี margin/physical quality เหมาะสม ไม่เลือกจาก setup WNS เพียงค่าเดียว

---

### 9.17 ปัญหาที่พบบ่อย

| อาการ | สิ่งที่ตรวจเป็นอันดับแรก |
|---|---|
| Placement ไม่ converge | พื้นที่ cells เทียบกับ usable core area, density, macro obstructions |
| Detailed placement ล้มเหลว | Overlap, rows/sites, fixed instances และพื้นที่ว่าง |
| CTS ไม่พบ clock | Clock port, SDC และ connectivity ผ่าน pad |
| Sinks หายไปบางส่วน | Clock gating, generated clocks, macro timing model |
| Clock buffer count สูงมาก | Sink distribution, constraints และ library choices |
| Setup ดีขึ้นแต่ hold แย่ลง | Positive skew และ data paths ที่สั้น |
| Hold repair เพิ่ม cells มาก | จำนวน paths, skew, constraints และ repair limits |
| STA ผ่านแต่มี unconstrained endpoints | Clock/I/O coverage และ exceptions |
| Post-CTS ผ่าน แต่ post-GRT ไม่ผ่าน | Congestion, route detour และ parasitic estimation |
| Reset recovery/removal ไม่ผ่าน | Reset release, synchronization และ methodology |

---

### 9.18 เกณฑ์ผ่าน Lab

แยก **placement/CTS completion** ออกจาก **timing readiness**

| รายการ | เกณฑ์ผ่าน |
|---|---|
| Placement legality | ไม่มี unresolved overlap/unplaced instances |
| Fixed objects | Pads/macros ตรงกับ approved floorplan |
| Clock connectivity | Required sinks ครบและไม่มี disconnected branches |
| Clock electrical checks | ไม่มี unresolved slew/capacitance violations |
| Constraint coverage | ไม่มี unexplained unconstrained endpoints |
| Setup | Slack ≥ 0 ทุก required corner/path group สำหรับ stage ที่ประเมิน |
| Hold | Slack ≥ 0 ทุก required corner/path group สำหรับ stage ที่ประเมิน |
| Reset checks | ผ่านหรือมี documented exception ตาม methodology |
| Physical integrity หลัง repair | Cells legal และ supply connectivity ถูกต้อง |
| Routing readiness | ไม่มี unresolved congestion ที่ขัดขวางขั้นถัดไป |
| Reproducibility | ระบุ source, PDK, tools, config, SDC และ checkpoint ได้ |

หาก constraints ออกแบบให้เผื่อ margin ไว้แล้ว ต้องระบุ margin นั้นในรายงาน ไม่เพิ่มเงื่อนไขตัวเลขโดยไม่มีที่มา

**ผลผ่าน Lab นี้เป็นการผ่าน checkpoint ก่อน detailed routing การประกาศ final timing closure ต้องใช้ผลหลัง routing/extraction ตาม sign-off methodology**

---

### 9.19 งานที่ต้องส่ง

1. Configuration และ SDC ของ run ที่เลือก
2. Placement/CTS checkpoints พร้อม provenance
3. ภาพ placement และ clock tree
4. Setup/hold reports ทุก required corner
5. Critical-path analysis อย่างน้อย 3 setup paths และ 3 hold paths
6. ตารางเปรียบเทียบอย่างน้อย 3 runs
7. รายการ cells/area ที่เพิ่มจาก CTS และ timing repair
8. ปัญหาคงค้างและเงื่อนไขส่งต่อ routing

รูปแบบสรุป:

```text
Design:
RTL commit/hash:
PDK revision:
LibreLane/OpenROAD version:
Configuration/SDC hash:
Clock definitions:
Required corners:

Placement legality:
Clock connectivity:
Constraint coverage:
Worst setup slack and corner:
Worst hold slack and corner:
Electrical violations:
Recovery/removal status:
Global-route congestion:
Selected checkpoint:
Remaining issues:
```

### 9.20 คำถามท้าย Lab

1. เหตุใด global placement ผ่านแล้วจึงต้องทำ detailed placement?
2. เหตุใด timing ก่อน CTS และหลัง CTS จึงต่างกัน?
3. Positive skew ช่วย setup แต่กระทบ hold อย่างไร?
4. เหตุใดเพิ่ม clock period จึงไม่แก้ same-edge hold violation?
5. เมื่อ cell delay สูง ควรเริ่มแก้ที่ placement หรือ logic?
6. การแทรก hold buffers กระทบ setup และ congestion อย่างไร?
7. เหตุใด timing ผ่านแต่มี unconstrained endpoints จึงยังรับรองผลไม่ได้?
8. ผล post-CTS กับ final extracted timing ต่างกันอย่างไร?