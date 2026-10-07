## Lab 5 — Technology synthesis ใน LibreLane

Lab นี้นำ Verilog ที่ export จาก VHDL ใน Lab 3 และผ่าน structural synthesis check ใน Lab 4 มาสังเคราะห์เป็น **gate-level netlist ที่ใช้ standard cells ของ IHP SG13G2** จากนั้นตรวจว่า netlist มีโครงสร้างครบถ้วน ไม่มีวงจรที่ mapping ไม่สำเร็จ และไม่มีปัญหา driver หรือ combinational loop ที่ยังไม่ได้แก้ไข

**ผลลัพธ์หลัก:** mapped netlist, synthesis reports, resolved configuration และหลักฐานที่เชื่อมโยงผล synthesis กลับไปยัง source และ environment ที่ใช้

> ตัวอย่างใช้ชื่อ top module `neorv32_asic_top` และไฟล์ `build/export/neorv32_asic_top.v` ให้แทนด้วยชื่อและตำแหน่งจริงจาก Lab 3 ก่อนรัน ชื่อดังกล่าวเป็นข้อตกลงของตัวอย่าง ไม่ใช่ชื่อ top ที่กำหนดโดย upstream NEORV32

### 5.1 วัตถุประสงค์

เมื่อจบ Lab ผู้เรียนต้องสามารถ:

1. อธิบายความต่างระหว่าง generic synthesis กับ technology synthesis
2. ตรวจว่า LibreLane ใช้ PDK และ standard-cell library ที่ถูกต้อง
3. กำหนด synthesis input และ clock target ได้
4. รันเฉพาะช่วง synthesis และ structural checks
5. อ่านรายงาน cell count, area, memory implementation และ synthesis warnings
6. ตรวจว่า CPU, ROM และ RAM ไม่ถูก optimize ออกโดยไม่ตั้งใจ
7. จัดทำ synthesis handoff สำหรับขั้นตอน physical implementation

### 5.2 หลักการที่ต้องเข้าใจก่อนเริ่ม

#### 5.2.1 Generic synthesis กับ technology synthesis

| ประเด็น | Generic synthesis ใน Lab 4 | Technology synthesis ใน Lab 5 |
|---|---|---|
| เป้าหมาย | ตรวจโครงสร้างและการแปลง RTL | สร้าง netlist สำหรับ technology ที่เลือก |
| Cell ที่พบ | Generic logic, registers และ memories | Standard cells และ macros ที่อนุญาต |
| Technology library | อาจยังไม่ใช้ Liberty ของ PDK | ใช้ Liberty และ mapping rules ของ PDK |
| Area | จำนวน generic resources | พื้นที่ cell ตาม library |
| Timing | ยังไม่ใช่ timing ของ technology เป้าหมาย | ใช้ timing model ช่วยเลือกวงจร |
| การนำไปใช้ต่อ | ตรวจ RTL และ synthesis feasibility | ส่งเข้า floorplan และ implementation |

ตัวอย่างเชิงแนวคิด:

```text
VHDL
  ↓ GHDL export
Verilog
  ↓ elaboration และ generic optimization
generic logic / registers / memories
  ↓ technology mapping
SG13G2 standard cells + approved macros
```

คำสั่ง RTL เช่น `+`, `==` และ `if` ไม่ได้สัมพันธ์กับ standard cell เพียงตัวเดียวเสมอไป เครื่องมือสามารถปรับโครงสร้างและรวม logic ก่อนเลือก cell ที่ใช้จริง

#### 5.2.2 หน้าที่ของ Liberty

Liberty ให้ข้อมูลของ standard cells เช่น:

- Boolean function
- Timing arcs
- Setup และ hold constraints
- Input capacitance
- Cell area
- ข้อจำกัด transition และ capacitance

PDK configuration ของ LibreLane เป็นส่วนที่เชื่อม flow เข้ากับ library และไฟล์ technology ที่เกี่ยวข้อง จึงต้องใช้ **PDK ที่เตรียมสำหรับ LibreLane แล้ว** ไม่ใช่ตรวจเพียงว่ามี directory ของ IHP Open PDK อยู่เท่านั้น [LibreLane Documentation](https://librelane.readthedocs.io/en/stable/usage/about_pdks.html?utm_source=chatgpt.com)

#### 5.2.3 Synthesis ผ่านยังไม่เท่ากับ timing closure

ผลของ Lab นี้ยืนยันได้ว่า netlist ถูกสร้างและตรวจโครงสร้างตามเกณฑ์ที่กำหนด แต่ยังไม่ยืนยัน:

- Timing หลัง placement และ routing
- Clock skew หลัง CTS
- Timing จาก extracted interconnect
- DRC, LVS หรือ PDN connectivity
- Functional equivalence ระหว่าง RTL กับ mapped netlist

ต้องตรวจเรื่องเหล่านี้ใน Lab ถัดไปตาม methodology ของโครงการ

---

### 5.3 สิ่งที่ต้องเตรียม

| รายการ | เงื่อนไขก่อนเริ่ม |
|---|---|
| Verilog export | ได้จาก Lab 3 และ export สำเร็จ |
| Structural checks | ผ่านเกณฑ์ Lab 4 |
| Top module | ชื่อตรงกับ module ใน exported Verilog |
| NEORV32 configuration | บันทึก generics/features ที่ใช้ export |
| ROM content | ตรงกับ firmware หรือ ROM smoke test ที่ตรวจแล้ว |
| LibreLane environment | เรียก CLI และเครื่องมือ synthesis ได้ |
| Built PDK | มี configuration และ standard-cell libraries |
| Memory plan | ระบุว่า ROM/RAM เป็น logic, registers หรือ macro |
| Source manifest | มี hash หรือ revision ของ input |

**อย่าเปลี่ยน generics ของ NEORV32 ระหว่าง Lab 3 กับ Lab 5 โดยไม่ export และตรวจ Lab 4 ใหม่** เพราะ generics อาจเปลี่ยน ISA features, peripheral set และขนาด memory

### 5.4 ข้อตกลงของ Lab

เริ่มต้นด้วยการ harden **core-level top** ที่ผ่าน Lab 4 ก่อน หากโครงการแยก core กับ pad wrapper ให้ใช้ core top ใน Lab นี้

หากต้อง synthesis full-chip top ที่ instantiate IO pads อยู่แล้ว ต้องเตรียม IO models และ configuration ตาม template ของโครงการเพิ่มเติม ไม่สามารถใช้ตัวอย่าง core-level configuration นี้แทนได้โดยตรง

กำหนด baseline ตัวอย่าง:

| Parameter | ค่า |
|---|---|
| PDK | `ihp-sg13g2` |
| Top | `neorv32_asic_top` |
| Clock port | `clk_i` |
| Target clock period | 20 ns |
| Target frequency | 50 MHz |
| Strategy | `AREA 0` |
| Memory implementation | ตามผล export และ memory plan |

ค่า 20 ns เป็น **เป้าหมายทดลอง** ไม่ใช่ผลยืนยันว่า NEORV32 configuration นี้ทำงานได้ที่ 50 MHz

---

### 5.5 ขั้นตอนที่ 1 — ตรวจ input และ environment

เข้า project root แล้วกำหนดตัวแปร:

```bash
export NEORV32_PROJECT_ROOT="$PWD"
export PDK_ROOT="${PDK_ROOT:-$HOME/.ciel}"

test -d "$PDK_ROOT/ihp-sg13g2"
test -s build/export/neorv32_asic_top.v

mkdir -p reports/lab5
```

หาก project อยู่คนละ directory ต้อง `cd` เข้า project ก่อนกำหนด `NEORV32_PROJECT_ROOT`

ตรวจเครื่องมือ:

```bash
command -v librelane
command -v yosys
command -v python3

librelane --version
yosys -V
python3 --version
```

สำหรับ environment ที่ใช้ Python module entry point ให้ใช้:

```bash
python3 -m librelane --version
```

หาก `librelane` หรือ `yosys` ไม่พบ ให้เข้า Nix environment ที่จัดเตรียมใน Lab 1 ก่อน ไม่ควรแก้โดยนำ executable จาก environment อื่นมาผสมโดยไม่มีการบันทึก version

บันทึกข้อมูล:

```bash
{
  date -u
  librelane --version
  yosys -V
  python3 --version
} > reports/lab5/tool_versions.txt

sha256sum \
  build/export/neorv32_asic_top.v \
  > reports/lab5/input_sha256.txt
```

ตรวจ top แบบเบื้องต้น:

```bash
rg -n '^[[:space:]]*module[[:space:]]+' \
  build/export/neorv32_asic_top.v
```

**ผลที่คาดหวัง**

พบ `neorv32_asic_top` และชื่อ module ที่เกี่ยวข้องตามรูปแบบ export

> การค้นด้วย `rg` ใช้สำรวจ source เท่านั้น การยืนยันว่า top ถูก elaborate และมี dependencies ครบต้องอาศัย synthesis log

---

### 5.6 ขั้นตอนที่ 2 — ตรวจ PDK สำหรับ LibreLane

สำรวจ PDK configuration และ Liberty:

```bash
rg --files -L "$PDK_ROOT/ihp-sg13g2/libs.tech" \
  | rg '/librelane/.*config\.tcl$'

rg --files -L "$PDK_ROOT/ihp-sg13g2/libs.ref" \
  | rg 'sg13g2_stdcell/.*\.lib(\.gz)?$'
```

หาก `rg` รุ่นที่ใช้ไม่รองรับ options ตามตัวอย่าง ให้ใช้:

```bash
find -L "$PDK_ROOT/ihp-sg13g2/libs.tech" \
  -path '*/librelane/*' -name config.tcl

find -L "$PDK_ROOT/ihp-sg13g2/libs.ref" \
  -path '*/sg13g2_stdcell/*' \
  \( -name '*.lib' -o -name '*.lib.gz' \)
```

ตรวจสามประเด็น:

1. มี configuration สำหรับ LibreLane
2. มี standard-cell timing libraries
3. ไฟล์ที่พบอ่านได้ และ symlinks ไม่เสีย

**การพบ Liberty หลาย corner ยังไม่ยืนยันว่า synthesis ใช้ corner ใด** ต้องตรวจ resolved configuration และ log ของ run จริงด้วย

ไม่ควรเลือก Liberty เองด้วยการหยิบไฟล์แรกที่พบแล้ว override `LIB` เพราะอาจขัดกับ corner configuration และ cell exclusions ของ PDK

---

### 5.7 ขั้นตอนที่ 3 — เตรียม synthesis configuration

สร้าง `librelane/config_lab5.json`:

```json
{
  "DESIGN_NAME": "neorv32_asic_top",
  "VERILOG_FILES": [
    "dir::../build/export/neorv32_asic_top.v"
  ],
  "CLOCK_PORT": "clk_i",
  "CLOCK_PERIOD": 20.0,
  "SYNTH_STRATEGY": "AREA 0",
  "ERROR_ON_SYNTH_CHECKS": true
}
```

`dir::` อ้างอิงตำแหน่งจาก directory ของ configuration file ตัวอย่างจึงชี้จาก `librelane/` ไปยัง `build/export/`

LibreLane รองรับการกำหนด design ผ่าน configuration file และมีตัวแปรสำหรับ synthesis strategy กับ synthesis checks โดยตรง [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/reference/configuration.html?utm_source=chatgpt.com)

#### ความหมายของค่าหลัก

| ตัวแปร | หน้าที่ | จุดที่ต้องตรวจ |
|---|---|---|
| `DESIGN_NAME` | กำหนด top module | ต้องตรงกับ exported Verilog |
| `VERILOG_FILES` | รายการ synthesis inputs | ห้ามมี testbench ปะปน |
| `CLOCK_PORT` | ระบุ clock ของ design | ต้องเป็น port ของ top |
| `CLOCK_PERIOD` | กำหนด clock target | หน่วย ns |
| `SYNTH_STRATEGY` | เลือก mapping strategy | บันทึกค่าทุก run |
| `ERROR_ON_SYNTH_CHECKS` | ให้ flow หยุดเมื่อพบ structural errors | ต้องเปิดสำหรับ baseline |

ตรวจ JSON syntax:

```bash
python3 -m json.tool \
  librelane/config_lab5.json \
  > /dev/null
```

คำสั่งนี้ตรวจได้เฉพาะ JSON syntax การตรวจชื่อ config variables, top และ PDK ต้องอาศัย LibreLane

หาก export จาก Lab 3 แยกหลายไฟล์ ให้เพิ่ม **ทุกไฟล์ที่จำเป็น** ใน `VERILOG_FILES` ตาม manifest ของ Lab 3 ไม่ควรเติมไฟล์ด้วย wildcard ที่อาจรวม testbench หรือ generated output จาก run เก่า

---

### 5.8 ขั้นตอนที่ 4 — ตรวจ flow และลำดับ steps ของ version ที่ติดตั้ง

LibreLane ประกอบ flow จาก steps และลำดับอาจเปลี่ยนตาม version หรือชนิด flow ให้ตรวจ version ที่ใช้งานจริงก่อนกำหนดจุดหยุด [LibreLane Documentation](https://librelane.readthedocs.io/en/stable/reference/architecture.html?utm_source=chatgpt.com)

ตรวจ CLI:

```bash
librelane --help \
  > reports/lab5/librelane_help.txt
```

สำหรับ built-in `Classic` flow สามารถตรวจรายชื่อ steps ผ่าน Python API:

```bash
python3 - <<'PY' \
  > reports/lab5/classic_steps.txt
from librelane.flows import Flow

flow = Flow.factory.get("Classic")
if flow is None:
    raise SystemExit("Classic flow is unavailable in this installation")

for step in flow.Steps:
    print(step.id)
PY
```

ดู steps ที่เกี่ยวข้อง:

```bash
rg -n 'Yosys|Synth|Unmapped|STA' \
  reports/lab5/classic_steps.txt
```

ตรวจว่ามี:

- `Yosys.Synthesis`
- `Checker.YosysSynthChecks`
- Step ตรวจ unmapped cells หาก flow รุ่นนั้นมี
- Timing steps ที่อยู่หลัง synthesis

หากโครงการใช้ custom flow หรือ `Chip` flow ให้ตรวจ steps ของ flow นั้นแทน `Classic`

**หลักการเลือกจุดหยุด:** ต้องได้ mapped netlist และรัน structural checks ที่เกี่ยวข้องครบ จุดหยุด `Yosys.Synthesis` เพียงอย่างเดียวอาจยังไม่รวม checker steps ถัดไป

---

### 5.9 ขั้นตอนที่ 5 — รัน baseline synthesis

ตัวอย่างต่อไปนี้ใช้ `Classic` และหยุดที่ `Checker.YosysSynthChecks` หลังตรวจแล้วว่า step นี้มีใน installation:

```bash
set -o pipefail

LAB5_RUN_TAG="LAB5_AREA0_$(date -u +%Y%m%d_%H%M%S)"

librelane \
  --flow Classic \
  --pdk ihp-sg13g2 \
  --pdk-root "$PDK_ROOT" \
  --run-tag "$LAB5_RUN_TAG" \
  --to Checker.YosysSynthChecks \
  librelane/config_lab5.json \
  2>&1 | tee "reports/lab5/${LAB5_RUN_TAG}.log"

LAB5_STATUS=${PIPESTATUS[0]}

printf '%s\n' "$LAB5_RUN_TAG" \
  > reports/lab5/run_tag.txt

printf '%s\n' "$LAB5_STATUS" \
  > reports/lab5/run_exit_code.txt

test "$LAB5_STATUS" -eq 0
```

หาก CLI version ที่ใช้มี syntax ต่างออกไป ให้ปรับจาก `librelane --help` ที่บันทึกไว้

#### เหตุผลที่ใช้ `pipefail` และ `PIPESTATUS`

เมื่อใช้:

```bash
librelane ... | tee run.log
```

สถานะของ pipeline อาจสะท้อนความสำเร็จของ `tee` โดยที่ LibreLane ล้มเหลว การเก็บ `PIPESTATUS[0]` ทันทีหลัง pipeline จึงช่วยบันทึกผลของ LibreLane โดยตรง

#### สิ่งที่ควรเห็นใน log

- โหลด source และเลือก top สำเร็จ
- Elaborate hierarchy สำเร็จ
- โหลด technology libraries
- ทำ generic optimization
- ทำ technology mapping
- เขียน mapped netlist
- รัน synthesis checks
- จบ run โดยไม่มี fatal error

**ห้ามแก้ baseline ด้วยการ skip synthesis checker เพื่อให้ run จบ** ต้องวิเคราะห์และแก้ต้นเหตุ หรือบันทึก waiver ที่มีเหตุผลและได้รับอนุมัติตาม methodology

---

### 5.10 ขั้นตอนที่ 6 — ระบุ artifacts ของ run จริง

โดยทั่วไป run ของตัวอย่างอยู่ใต้:

```bash
export LAB5_RUN_DIR="librelane/runs/$LAB5_RUN_TAG"

test -d "$LAB5_RUN_DIR"
```

หาก log ระบุตำแหน่งอื่น ให้ใช้ตำแหน่งจาก log

สำรวจ artifacts:

```bash
rg --files "$LAB5_RUN_DIR" \
  | rg 'state_out\.json$|metrics.*\.json$|config.*\.json$|\.v$|\.rpt$|\.log$'
```

ไม่ควรผูก scripts เข้ากับเลข stage เช่น `06-yosys-synthesis` เพราะเลขอาจเปลี่ยนเมื่อ flow หรือ version เปลี่ยน

ค้น synthesis directory:

```bash
find "$LAB5_RUN_DIR" \
  -type d -iname '*yosys-synthesis*'
```

เปิด `state_out.json` ของ synthesis step เพื่อดู netlist ที่เป็น output ของ step นั้น:

```bash
python3 -m json.tool \
  /path/to/synthesis-step/state_out.json
```

ใช้ path ที่พบจริงแทน `/path/to/synthesis-step/`

**อย่าเลือก `.v` ไฟล์แรกใน run directory เป็น mapped netlist** เพราะอาจเป็น intermediate file หรือ model ที่ไม่ได้ใช้ส่งต่อ

---

### 5.11 ขั้นตอนที่ 7 — ตรวจ resolved configuration

เปิด configuration ที่ LibreLane บันทึกใน run แล้วตรวจ:

| รายการ | สิ่งที่ต้องยืนยัน |
|---|---|
| Top module | เป็น top เดียวกับ Lab 4 |
| Source paths | ชี้ไปยัง input ที่ตั้งใจใช้ |
| PDK | `ihp-sg13g2` |
| Standard-cell library | เป็น library ที่โครงการอนุญาต |
| Clock port | ตรงกับ top interface |
| Clock target | 20 ns สำหรับ baseline |
| Strategy | `AREA 0` |
| Mapping libraries | ตรงกับ PDK configuration |
| Excluded cells | ไม่มี override ที่ไม่ได้ตั้งใจ |
| Extra models/macros | มีเฉพาะรายการที่อนุมัติ |

ค้นข้อมูลประกอบใน log:

```bash
rg -n -i \
  'liberty|sg13g2|corner|abc|dfflibmap|strategy|clock' \
  "reports/lab5/${LAB5_RUN_TAG}.log"
```

ชื่อ pass และรายละเอียด log อาจต่างกันตาม synthesis implementation ให้ใช้ทั้ง log และ resolved configuration ประกอบกัน

---

### 5.12 ขั้นตอนที่ 8 — ตรวจ mapping completeness

กำหนด netlist จาก path ที่อ่านได้ใน synthesis state:

```bash
export LAB5_NETLIST="/absolute/path/to/mapped_netlist.v"

test -s "$LAB5_NETLIST"
```

สำรวจ standard cells:

```bash
rg -n 'sg13g2_' "$LAB5_NETLIST" | head -n 40
```

ควรพบ instance ของ SG13G2 cells หาก design มี logic ที่ต้อง mapping

สำรวจ generic cells หรือ structures ที่อาจเหลือ:

```bash
rg -n \
  '\$_|\$dff|\$adff|\$dlatch|\$mem|\$add|\$mul' \
  "$LAB5_NETLIST"
```

**การค้นไม่พบไม่เท่ากับพิสูจน์ว่า mapping ครบ** เนื่องจากรูปแบบ output และชื่อ cell อาจแตกต่างกัน ต้องใช้ synthesis statistics และ unmapped-cell checker ประกอบด้วย

จำแนก cell ทุกประเภทเป็น:

1. Standard cells ที่อนุญาต
2. Approved macros เช่น SRAM
3. Approved black boxes ที่มีแผน physical integration
4. Unexpected หรือ unmapped cells

รายการกลุ่มที่ 4 ต้องเป็นศูนย์ก่อนผ่าน Lab

ถ้า unmapped checker อยู่หลังจุดหยุดที่เลือก ให้ขยาย run ถึง checker นั้น โดยใช้ชื่อ step จาก installation จริง

---

### 5.13 ขั้นตอนที่ 9 — ตรวจ structural errors และ warnings

ค้นข้อความที่เกี่ยวข้อง:

```bash
rg -n -i \
  'error:|warning:|no driver|has no driver|multiple.*driver|conflicting drivers|logic loop|combinational loop|unmapped|blackbox|latch' \
  "$LAB5_RUN_DIR" \
  > reports/lab5/diagnostic_hits.txt
```

ตรวจ report ของ Yosys เช่น `pre_synth_chk.rpt` หาก version ที่ใช้สร้างไฟล์นี้:

```bash
rg --files "$LAB5_RUN_DIR" \
  | rg 'pre_synth_chk\.rpt$|.*synth.*chk.*\.rpt$'
```

`Checker.YosysSynthChecks` ตรวจปัญหาเช่น combinational loops และ wires ที่ไม่มี driver โดยสามารถกำหนดให้ flow หยุดเมื่อพบ errors ได้ [librelane.readthedocs.io](https://librelane.readthedocs.io/en/stable/reference/step_config_vars.html?utm_source=chatgpt.com)

#### วิธีวิเคราะห์ปัญหาหลัก

| ปัญหา | สิ่งที่ต้องไล่ตรวจ |
|---|---|
| Wire ไม่มี driver | Source assignment, conditional generation, wrapper connections |
| Multiple drivers | หลาย process ขับ signal เดียวกัน หรือต่อ output ชนกัน |
| Combinational loop | Feedback ที่ไม่มี sequential element หรือ logic export ผิด |
| Unexpected latch | Assignment ไม่ครบทุก branch หรือ construct ที่ export ไม่ตรงเจตนา |
| Missing module | Input files ไม่ครบ ชื่อ module ผิด หรือ macro model หาย |
| Unexpected black box | ไม่ได้ resolve implementation ของ module |
| Logic ถูกลบจำนวนมาก | Constant controls, unused outputs หรือ memory/configuration ผิด |

จัด warning review เป็นตาราง:

| ข้อความ/ตำแหน่ง | สาเหตุ | ผลกระทบ | การดำเนินการ | สถานะ |
|---|---|---|---|---|
| ตัวอย่าง: peripheral output unused | ปิด feature ใน baseline | Logic ของ peripheral ถูกลบ | เทียบกับ configuration | Reviewed |
| ตัวอย่าง: signal ไม่มี driver | Wrapper connection ไม่ครบ | Functional error | แก้และ export ใหม่ | Open |

ไม่จำเป็นต้องกำหนดว่า warning ทุกชนิดต้องเป็นศูนย์ แต่ **warning ทุกประเภทต้องมีคำอธิบายและผลการ review**

---

### 5.14 ขั้นตอนที่ 10 — ตรวจ ROM และ RAM ของ NEORV32

Memory เป็นจุดที่อาจทำให้ synthesis สำเร็จแต่ผลลัพธ์ไม่ตรงเป้าหมายมากที่สุด

#### 5.14.1 แยก memory แต่ละประเภทก่อน

สร้าง memory inventory:

| Memory | ขนาดจริงจาก configuration | รูปแบบหลัง export | รูปแบบหลัง mapping |
|---|---:|---|---|
| Boot ROM | กรอกจาก Lab 3 | Constant logic / memory structure | ตรวจจาก report |
| Instruction memory | กรอกจาก Lab 3 | Array / registers / wrapper | ตรวจจาก report |
| Data memory | กรอกจาก Lab 3 | Array / registers / wrapper | ตรวจจาก report |

อย่าใช้ขนาด default ของ upstream แทนขนาดที่ export จริง

#### 5.14.2 กรณี memory ถูกสร้างด้วย standard cells

คำนวณขนาด logical storage:

\[
N_{\text{bits}}=N_{\text{words}}\times W_{\text{data}}
\]

ตัวอย่าง RAM 1,024 words × 32 bits:

\[
N_{\text{bits}}=32{,}768\ \text{bits}
\]

ถ้า memory ถูก implement ด้วย flip-flops พื้นที่และ cell count อาจเพิ่มขึ้นมาก รวมทั้งมี mux และ write-control logic เพิ่มเติม

จำนวน flops จริงไม่จำเป็นต้องเท่ากับจำนวน bits เสมอ เพราะขึ้นกับ implementation และ optimization แต่ต้องอธิบายความสัมพันธ์ได้

#### 5.14.3 กรณีใช้ SRAM macro

ต้องมีอย่างน้อย:

- Wrapper ที่ interface ตรงกับ memory เดิม
- Simulation model สำหรับ functional validation
- Black-box declaration หรือ synthesis model ที่ถูกต้อง
- Timing และ physical views สำหรับขั้นต่อไป
- แผนต่อ clock, control และ power pins
- การตรวจ read latency, write behavior และ byte enables

**การมี SRAM module name ใน netlist ไม่ได้พิสูจน์ว่า memory integration พร้อมแล้ว**

การใช้ macro ใน LibreLane ต้องมีการจัดเตรียม macro views และ configuration ที่เกี่ยวข้องสำหรับ implementation ด้วย [LibreLane Documentation](https://librelane.readthedocs.io/en/stable/usage/using_macros.html?utm_source=chatgpt.com)

#### 5.14.4 ตรวจ ROM ไม่ถูก optimize ออกผิดเจตนา

ตรวจว่า:

1. ROM image เป็นไฟล์เดียวกับที่ผ่าน smoke test
2. Address path จาก CPU ยังอยู่
3. ROM data เชื่อมเข้าทาง instruction/data path ที่ถูกต้อง
4. Reset หรือ enable ไม่ถูกผูกเป็นค่าที่ทำให้ ROM ไม่ถูกใช้งาน
5. Logic reduction มีเหตุผลสอดคล้องกับ ROM contents

ROM ที่เป็น constants สามารถถูกลด logic ได้มาก จึงไม่ควรตัดสินจาก cell count อย่างเดียว ต้องมี functional verification ประกอบ

---

### 5.15 ขั้นตอนที่ 11 — อ่าน cell count และ area

ค้น synthesis statistics:

```bash
rg -n -i \
  'number of cells|chip area|cell area|statistics|memory|flip.flop' \
  "$LAB5_RUN_DIR"
```

บันทึกข้อมูล:

| Metric | ค่าจริง | แหล่งข้อมูล |
|---|---:|---|
| Total mapped cells | กรอกจาก run | Synthesis report |
| Sequential cells | กรอกจาก run | Cell statistics |
| Combinational cells | กรอกจาก run | Cell statistics |
| Macro instances | กรอกจาก run | Netlist/report |
| Standard-cell area | กรอกจาก run | Library-based report |
| Unexpected latches | กรอกจาก run | Cell/check report |
| Unmapped cells | กรอกจาก run | Checker/statistics |
| Structural errors | กรอกจาก run | Synthesis checker |

พื้นที่ standard cells โดยหลักคำนวณจาก:

\[
A_{\text{cells}}=\sum_i N_iA_i
\]

โดย \(N_i\) คือจำนวน instances และ \(A_i\) คือ area ของ cell type นั้นตาม library

ต้องตรวจหน่วยจาก library/report ก่อนบันทึกเป็น \(\mu m^2\)

**Synthesis cell area ไม่ใช่ die area หรือ core area** เพราะยังไม่รวมพื้นที่สำหรับ placement whitespace, routing, PDN, macros ตามรูปแบบรายงาน และองค์ประกอบ physical อื่น

---

### 5.16 ขั้นตอนที่ 12 — ตรวจว่า design สำคัญยังอยู่

เปรียบเทียบ top interface กับ Lab 4:

- Clock และ reset
- UART หรือ peripheral ที่เปิดใช้
- GPIO
- Interrupt inputs
- External bus interface หากมี
- Debug interface หากเปิดใช้

สำหรับ NEORV32 ให้ review เพิ่มเติม:

1. CPU logic ยังอยู่ตาม configuration
2. Register file มี implementation ที่อธิบายได้
3. ISA features ตรงกับ generics ที่ export
4. Memory access path ยังเชื่อมต่อครบ
5. ไม่มี reset/enable ถูกผูก constant โดยไม่ตั้งใจ
6. Optional peripherals ที่ถูกลบตรงกับ features ที่ปิดไว้

หาก flatten hierarchy แล้ว ชื่อ CPU module อาจไม่เหลือใน netlist การค้นชื่อ instance เดิมจึงไม่เพียงพอ ต้องอาศัย synthesis statistics, connectivity และ functional verification

---

### 5.17 ขั้นตอนที่ 13 — ทดลอง synthesis strategy

หลัง baseline ผ่านแล้ว ทดลองเปลี่ยนเพียง `SYNTH_STRATEGY` เช่น:

```json
"SYNTH_STRATEGY": "DELAY 0"
```

ใช้ run tag ใหม่ และรักษา:

- Exported Verilog เดิม
- Top เดิม
- PDK revision เดิม
- Clock target เดิม
- Memory implementation เดิม

LibreLane มี synthesis strategies และ synthesis exploration สำหรับเปรียบเทียบผลด้าน area กับ timing แต่ strategy ที่ดีที่สุดต้องประเมินจาก design จริง [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/reference/flows.html?utm_source=chatgpt.com)

ตารางทดลอง:

| Run | Strategy | Clock target | Cell area | Cells | Structural checks | Timing evidence |
|---|---|---:|---:|---:|---|---|
| Baseline | AREA 0 | 20 ns | กรอก | กรอก | กรอก | ยังไม่รัน/ระบุ report |
| Experiment | DELAY 0 | 20 ns | กรอก | กรอก | กรอก | ยังไม่รัน/ระบุ report |

ถ้า run หยุดก่อน STA ให้บันทึก timing ว่า **ยังไม่รัน** ห้ามนำ ABC output หรือชื่อ `DELAY` มาแทนผล STA

การเลือก strategy เพื่อใช้งานจริงควรเปรียบเทียบ timing ด้วย constraints และ corner methodology เดียวกันในขั้นถัดไป

---

### 5.18 ขั้นตอนที่ 14 — จัดทำ synthesis handoff

รวบรวม:

```text
lab5_handoff/
  mapped_netlist.v
  config_lab5.json
  resolved_config.json
  synthesis_state.json
  tool_versions.txt
  input_sha256.txt
  synthesis_summary.md
  warning_review.md
  memory_inventory.md
  reports/
```

ใช้ไฟล์จาก run ที่ผ่านเกณฑ์จริง และคง run directory ต้นฉบับไว้ด้วย เนื่องจาก state อาจอ้างอิง paths ภายใน run การ copy state เพียงไฟล์เดียวไม่ได้ทำให้ package ย้ายเครื่องแล้ว resume ได้ทันที

เนื้อหา `synthesis_summary.md` ควรมี:

```markdown
# Lab 5 Synthesis Summary

- Source revision:
- Exported Verilog SHA256:
- Top module:
- NEORV32 configuration:
- LibreLane version:
- Yosys version:
- PDK revision/build identity:
- Standard-cell library:
- Mapping Liberty files:
- Clock target:
- Synthesis strategy:
- Run tag:
- Exit code:

## Results
- Mapped cells:
- Standard-cell area and units:
- Memory implementation:
- Structural errors:
- Unmapped cells:
- Unexpected black boxes:
- Unexpected latches:

## Warning review
- Reviewed warning categories:
- Open issues:
- Approved waivers:

## Verification status
- RTL/ROM smoke test:
- Structural synthesis checks:
- Gate-level functional simulation:
- Formal equivalence:
- Pre-layout STA:

## Decision
- Ready for the next lab:
- Remaining actions:
```

รายการที่ยังไม่ได้ตรวจต้องระบุว่า `NOT RUN` หรือ “ยังไม่รัน”

### 5.19 เกณฑ์ผ่าน Lab

| Gate | Acceptance |
|---|---|
| Input identity | ระบุ source revision และ export hash ได้ |
| Top/interface | ตรงกับ Lab 3–4 |
| PDK/library | ตรวจจาก resolved configuration และ log แล้ว |
| Run result | LibreLane exit code เป็นศูนย์ |
| Mapping | ใช้ approved standard cells/macros |
| Unmapped cells | เป็นศูนย์ |
| Structural errors | เป็นศูนย์ หรือมี approved waiver ที่ระบุขอบเขต |
| Latches | ไม่มี unexpected latch |
| Black boxes | ไม่มี unexpected black box |
| Memory | อธิบาย implementation ของ ROM/RAM ทุกชุดได้ |
| Optimization | ไม่มี functional block ถูกลบโดยไม่ตั้งใจ |
| Warning review | Review ครบทุกประเภท |
| Reports | มี cell count, area และแหล่งข้อมูล |
| Handoff | มี netlist, config, reports และ provenance |

**สถานะที่เหมาะสมเมื่อผ่าน Lab:** “Technology synthesis และ structural checks ผ่าน พร้อมเข้าสู่การตรวจ timing และ physical implementation ตามขั้นตอนถัดไป”

ยังไม่ควรใช้คำว่า “sign-off ผ่าน”

### 5.20 ปัญหาที่พบบ่อย

| อาการ | สาเหตุที่ควรตรวจ | วิธีแก้ |
|---|---|---|
| Top module not found | ชื่อ top หรือ input path ผิด | เทียบ config กับ exported module |
| Missing built PDK configuration | ใช้ source PDK ที่ยังไม่เตรียมสำหรับ flow | กลับไปตรวจ environment/PDK ใน Lab 1 |
| Module ไม่พบ | Export/dependency list ไม่ครบ | แก้ manifest แล้วตรวจ Lab 4 ใหม่ |
| No driver | Wrapper หรือ conditional configuration ผิด | ไล่ signal ถึงต้นทาง |
| Multiple drivers | หลาย sources ขับ net เดียวกัน | แก้โครงสร้าง RTL/wrapper |
| Unexpected latch | Assignment ไม่ครบ | แก้ VHDL แล้ว export ใหม่ |
| Area ใหญ่ผิดคาด | RAM กลายเป็น register array | ตรวจ memory reports และ macro plan |
| CPU ถูกลบจำนวนมาก | Output unused หรือ controls เป็น constants | ตรวจ top connectivity และ configuration |
| ROM behavior เปลี่ยน | ROM image หรือ export ไม่ตรง | ตรวจ hash แล้วรัน smoke test ใหม่ |
| Black-box SRAM แต่ placement ไม่พร้อม | Macro views/configuration ไม่ครบ | จัดเตรียม memory integration ให้ครบ |
| `tee` จบแต่ synthesis fail | ไม่ตรวจ exit status ของ LibreLane | ใช้ `PIPESTATUS` |
| Timing ไม่มีรายงาน | หยุดก่อน STA | บันทึก “ยังไม่รัน” และตรวจในขั้นถัดไป |

### 5.21 คำถามท้าย Lab

1. เหตุใด generic synthesis ผ่านจึงยังไม่ยืนยันว่า technology mapping สำเร็จ?
2. Liberty กับ LEF มีหน้าที่ต่างกันอย่างไร?
3. `CLOCK_PERIOD: 20.0` รับประกันการทำงานที่ 50 MHz หรือไม่?
4. เหตุใด RAM 32 Kbits จึงอาจทำให้ standard-cell area เพิ่มขึ้นมาก?
5. การไม่พบชื่อ CPU module หลัง flatten หมายถึง CPU ถูกลบเสมอหรือไม่?
6. เหตุใด ROM ที่ใช้ constants จึงมี cell count ต่ำกว่าจำนวน ROM bits มากได้?
7. Synthesis exit code เป็นศูนย์เพียงอย่างเดียวเพียงพอต่อการผ่าน Lab หรือไม่?
8. ต้องมีหลักฐานอะไรบ้างก่อนเลือก `DELAY 0` แทน `AREA 0`?
9. การ copy `state_out.json` เพียงไฟล์เดียวเพียงพอต่อการ resume บนเครื่องใหม่หรือไม่?
10. ผลตรวจใดที่ยังต้องทำก่อนเรียก design ว่าพร้อม sign-off?