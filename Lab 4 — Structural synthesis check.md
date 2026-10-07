## Lab 4 — Structural synthesis check

Lab นี้ตรวจว่า generated Verilog จาก Lab 3 มีโครงสร้างที่เหมาะสำหรับนำไป synthesis ต่อ และเมื่อเชื่อมกับ chip wrapper แล้ว ไม่มีข้อผิดพลาดด้าน hierarchy, drivers, latch inference หรือการเชื่อมต่อที่ทำให้ส่วนสำคัญของ SoC ถูกตัดออก

**เป้าหมายของ Lab:** สร้าง structural checkpoint ที่ตรวจสอบได้ก่อนเริ่ม technology mapping โดยแยกผลของ SoC core ออกจากผลของ full-chip integration

| ระดับตรวจ | Top-level | สิ่งที่ตรวจ |
|---|---|---|
| Core | `neorv32_asic_core` | Generated RTL, registers, memories และ internal connectivity |
| Integration | `chip_core` หรือ wrapper ที่เลือก | Clock/reset, UART, GPIO และ bit mapping |
| Full chip | `chip_top` | Pad instances, macro declarations และ top-level connectivity |

ชื่อ top-level ต้องแทนด้วยชื่อจริงของโครงการหากต่างจากตัวอย่าง

### 4.1 วัตถุประสงค์

เมื่อจบ Lab ผู้เรียนต้องสามารถ:

1. สร้าง synthesis source list ที่มี inputs ครบและไม่มี testbench
2. ตรวจ hierarchy และจำแนก intentional blackboxes
3. ตรวจ undriven signals, multiple drivers และ combinational loops
4. ตรวจ latch inference และ register/reset structures
5. ตรวจ memory structures ก่อน mapping
6. วิเคราะห์ logic ที่ถูก optimization ตัดออก
7. ตรวจว่า CPU, boot path และ interfaces ที่เลือกยังมีโครงสร้างสอดคล้องกับ specification
8. จัดทำรายงาน readiness สำหรับ technology mapping

**ขอบเขต:** Structural check ไม่พิสูจน์ functional equivalence, timing closure, CDC/RDC correctness หรือ power-grid connectivity

---

### 4.2 สิ่งที่ต้องเตรียม

ใช้ผลจาก Lab 1–3:

| Input | เงื่อนไข |
|---|---|
| Generated Verilog | ผ่าน parser และ Verilog ROM smoke |
| Export manifest | ระบุ source/configuration/tool versions |
| Interface matrix | ระบุ ports, widths และ bit mapping |
| Memory inventory | ระบุ capacity, ports และ initialization |
| Chip wrappers | ใช้ revision ที่เลือกไว้ |
| Pad/macro declarations | ตรงกับ PDK และ macro versions |
| Source list | แยก core, integration และ full-chip |

หาก generated Verilog เปลี่ยนหลัง Lab 3 ต้องตรวจ hash และรัน verification ที่เกี่ยวข้องใหม่ก่อนใช้เป็น baseline

---

### 4.3 ขั้นตอนที่ 1 — เตรียม workspace

```bash
cd /path/to/neorv32-ihp

export PROJECT_ROOT="$PWD"
export LAB4_BUILD="$PROJECT_ROOT/build/lab04"
export LAB4_REPORT="$PROJECT_ROOT/reports/lab04"

mkdir -p "$LAB4_BUILD"
mkdir -p "$LAB4_REPORT"
mkdir -p "$PROJECT_ROOT/scripts"

set -o pipefail
```

ตรวจ inputs:

```bash
test -s "$PROJECT_ROOT/rtl/generated/neorv32_asic_core.v"

yosys -V > "$LAB4_REPORT/yosys-version.txt"

sha256sum \
  "$PROJECT_ROOT/rtl/generated/neorv32_asic_core.v" \
  > "$LAB4_REPORT/generated-input.sha256"
```

คัดลอก input ที่จะตรวจเข้า build directory:

```bash
cp "$PROJECT_ROOT/rtl/generated/neorv32_asic_core.v" \
   "$LAB4_BUILD/neorv32_asic_core.v"

cmp \
  "$PROJECT_ROOT/rtl/generated/neorv32_asic_core.v" \
  "$LAB4_BUILD/neorv32_asic_core.v"
```

การคัดลอกช่วยให้ commands และ checkpoints อยู่ใน directory เดียวกัน แต่ต้องเก็บ hash เพื่อยืนยันว่าเป็น source ชุดเดียวกับ release candidate

---

### 4.4 ขั้นตอนที่ 2 — ตรวจ capabilities ของ Yosys version ที่ใช้

เก็บ command help:

```bash
yosys -Q -T -p \
  'help hierarchy; help check; help select; help scc; help memory_collect' \
  > "$LAB4_REPORT/yosys-command-help.txt"
```

ตรวจ options ที่จะใช้จริง:

```bash
less "$LAB4_REPORT/yosys-command-help.txt"
```

เหตุผลที่ต้องตรวจ version:

- บาง options มีเฉพาะ Yosys รุ่นใหม่
- Memory cell types และ report formats อาจเปลี่ยน
- Diagnostic behavior อาจต่างกัน
- LibreLane อาจใช้ Yosys/frontend ที่ต่างจากเครื่องมือ standalone

ตัวอย่างเช่น `check -nolatches` มีในเอกสารรุ่นใหม่ แต่ไม่ปรากฏใน help ของบางรุ่นเก่า จึงใช้ latch selection เป็นวิธีตรวจพื้นฐานใน Lab นี้ [YosysHQ Yosys 0.69-dev documentation](https://yosyshq.readthedocs.io/projects/yosys/en/latest/cmd/index_passes_status.html?utm_source=chatgpt.com)

---

### 4.5 ขั้นตอนที่ 3 — กำหนด synthesis source lists

แบ่ง source ตามขอบเขต:

| Source list | ต้องมี |
|---|---|
| Core | Generated Verilog และ external core IP ถ้ามี |
| Integration | Core sources และ `chip_core` |
| Full chip | Integration sources, `chip_top` และ pad/macro declarations |

ตรวจว่าไม่มี:

- Testbench
- UART stimulus/decoder
- Clock generator สำหรับ simulation
- Generated source รุ่นเก่าที่ชื่อ module ซ้ำ
- FPGA PLL หรือ vendor primitives ที่ไม่ได้ตั้งใจใช้
- Behavioral macro models ที่ไม่เหมาะกับ synthesis

สร้าง source manifest โดยระบุอย่างน้อย:

```csv
file,role,scope,hash,origin
rtl/generated/neorv32_asic_core.v,generated_core,core,...,Lab3 export
```

สำหรับ full-chip ต้องบันทึกด้วยว่าแต่ละ pad/SRAM ใช้:

- Behavioral simulation model
- Synthesis blackbox declaration
- Liberty
- LEF
- GDS
- Power-pin convention

ไฟล์เหล่านี้มีหน้าที่ต่างกัน ไม่ควรใช้แทนกันโดยอัตโนมัติ

---

### 4.6 ขั้นตอนที่ 4 — ตรวจ core hierarchy แบบไม่อนุญาต blackboxes

สร้าง `build/lab04/core_hierarchy.ys`:

```yosys
read_verilog neorv32_asic_core.v

hierarchy -simcheck -top neorv32_asic_core

stat

write_rtlil core_hierarchy.il
```

รัน:

```bash
cd "$LAB4_BUILD"

yosys \
  -l "$LAB4_REPORT/core-hierarchy.log" \
  -s core_hierarchy.ys
```

`hierarchy -check` ตรวจ unknown modules ส่วน `hierarchy -simcheck` ตรวจเพิ่มว่าไม่มี instantiated blackboxes และมี top ที่กำหนดชัดเจน [YosysHQ Yosys 0.51 documentation](https://yosyshq.readthedocs.io/projects/yosys/en/v0.51/cmd/hierarchy.html?utm_source=chatgpt.com)

สำหรับ baseline core ที่ export มาครบจาก GHDL ควรใช้ strict mode นี้

หาก architecture ใช้ SRAM หรือ external IP แบบ blackbox โดยตั้งใจ:

1. ใช้ `hierarchy -check`
2. เก็บรายการ instantiated blackboxes
3. ตรวจรายการกับ approved macro inventory
4. ไม่รายงานว่าผ่าน no-blackbox gate

**เกณฑ์ผ่าน:** ไม่มี unknown module และ blackboxes ทั้งหมดมีเหตุผลและ implementation views ที่ระบุได้

---

### 4.7 ขั้นตอนที่ 5 — สร้าง checkpoints ก่อนและหลัง optimization

สร้าง `build/lab04/core_structural.ys`:

```yosys
read_verilog neorv32_asic_core.v
hierarchy -simcheck -top neorv32_asic_core

# Lower behavioral processes without running the usual proc cleanup.
proc -noopt

# Preserve an inspectable checkpoint.
write_rtlil core_preopt.il
write_json core_preopt.json
stat

# Check structural problems before general optimization.
check -assert

# Reject unintended inferred latches.
select -assert-none t:$dlatch t:$adlatch t:$dlatchsr

# Check combinational strongly connected components.
scc -expect 0

# General optimization without technology mapping.
opt

check -assert
select -assert-none t:$dlatch t:$adlatch t:$dlatchsr
scc -expect 0

# Collect memories for inspection, without memory_map.
memory_collect

check -assert

write_rtlil core_postopt.il
write_json core_postopt.json
stat
```

รัน:

```bash
yosys \
  -l "$LAB4_REPORT/core-structural.log" \
  -s "$LAB4_BUILD/core_structural.ys"
```

ตรวจว่า `proc -noopt` และ `scc -expect` รองรับใน installed version จากขั้นตอนที่ 2

Checkpoint `preopt` หมายถึงก่อน general `opt` pass ไม่ได้หมายถึง source ที่ไม่ผ่าน transformations เลย เพราะ `proc` ได้แปลง behavioral processes เป็น cells แล้ว

**เหตุผลที่ตรวจสองช่วง:** Optimization อาจลบ logic ที่ไม่ถูกใช้งาน รวมถึงปัญหาบางส่วน การตรวจเฉพาะหลัง optimization อาจไม่แสดงสาเหตุของ integration mistake ที่ทำให้ logic ทั้งก้อนหายไป

---

### 4.8 ขั้นตอนที่ 6 — ตรวจ undriven signals และ multiple drivers

Yosys `check` ตรวจปัญหาพื้นฐาน ได้แก่ combinational loops, conflicting drivers และ used wires ที่ไม่มี driver ส่วน `-assert` ทำให้การพบปัญหาเป็น runtime error ของคำสั่ง [YosysHQ Yosys 0.47 documentation](https://yosyshq.readthedocs.io/projects/yosys/en/0.47/cmd/check.html?utm_source=chatgpt.com)

ค้นหา diagnostics:

```bash
rg -n -C 4 \
  'used but has no driver|multiple conflicting drivers|logic loop|ERROR:|Warning:' \
  "$LAB4_REPORT/core-structural.log"
```

#### กรณี undriven signal

ตรวจทีละลำดับ:

1. Signal อยู่ใน module ใด
2. เป็น input port หรือ internal wire
3. ถูกใช้โดย cell ใด
4. Driver ควรมาจาก source ใด
5. Connection ถูกละไว้ หรือ source หาย
6. Feature เปิดใช้งานจริงหรือไม่
7. Tie-off value ตาม protocol คืออะไร

ตัวอย่าง:

```systemverilog
wire uart_rx;

neorv32_asic_core u_soc (
  .uart0_rxd_i(uart_rx),
  ...
);
```

หากไม่มี driver ของ `uart_rx` ต้องแก้ connection ให้มาจาก input pad หรือจาก idle constant ที่เหมาะสมสำหรับ configuration ที่ไม่ได้ใช้งาน

อย่าแก้ทุก undriven signal เป็น zero โดยอัตโนมัติ

#### กรณี multiple drivers

ตัวอย่างผิด:

```systemverilog
assign core_rstn = reset_from_pad;
assign core_rstn = debug_rstn;
```

หากต้องรวม reset sources ต้องกำหนด logic ตาม polarity และ reset architecture เช่น active-low sources อาจต้องใช้ AND ตาม design specification พร้อมตรวจ assertion/deassertion behavior

อย่าแปลงเป็น AND/OR เพียงเพื่อให้ multiple-driver warning หายโดยยังไม่เข้าใจ reset semantics

**เกณฑ์ผ่าน:** ไม่มี undriven used signals หรือ conflicting drivers ที่ยังไม่ได้แก้

---

### 4.9 ขั้นตอนที่ 7 — ตรวจ latches

ค้นหา inference messages และ cell types:

```bash
rg -n -i \
  'latch|dlatch|adlatch' \
  "$LAB4_REPORT/core-structural.log"
```

สำหรับ baseline synchronous SoC นี้ กำหนด policy ว่าไม่มี unintended latches

สาเหตุที่พบบ่อย:

```systemverilog
always @* begin
  if (enable)
    result = data;
end
```

เมื่อ `enable` เป็น low ค่า `result` ต้องคงค่าเดิม จึงอาจเกิด latch

แก้ตามพฤติกรรมที่ต้องการจริง เช่น combinational default:

```systemverilog
always @* begin
  result = '0;

  if (enable)
    result = data;
end
```

หรือเปลี่ยนเป็น sequential logic หากต้องการเก็บค่า

สำหรับ generated Verilog ให้ trace กลับไปยัง VHDL source/wrapper และแก้ต้นทาง ไม่แก้ output ด้วยมือเป็นวิธีหลัก

หาก design ตั้งใจใช้ latch ต้องมี specification, timing methodology และ mapping support แยกต่างหาก จึงไม่สามารถผ่าน zero-latch gate นี้โดยไม่ปรับ policy

---

### 4.10 ขั้นตอนที่ 8 — ตรวจ combinational loops

ใช้ทั้ง:

```yosys
check -assert
scc -expect 0
```

ตรวจ log:

```bash
rg -n -i \
  'scc|strongly connected|logic loop|combinatorial loop' \
  "$LAB4_REPORT/core-structural.log"
```

สาเหตุที่พบบ่อย:

- Ready/valid feedback ที่ไม่มี register
- Output ถูกต่อกลับ input โดยไม่ตั้งใจ
- Bus mux/decode ผิด
- Wrapper connections ผูก nets เข้าหากัน
- Reset/control logic แบบ self-reference

ตัวอย่าง:

```systemverilog
assign a = b & enable;
assign b = a | request;
```

ต้องตรวจ architecture และเพิ่ม sequential boundary หรือแก้ connection ตามพฤติกรรมที่ต้องการ

**ข้อจำกัด:** SCC result ขึ้นกับ cell connectivity model และ selection การตรวจ core แบบ hierarchical ไม่ควรใช้แทนการตรวจ flattened integration cone ทั้งหมด และ macro blackboxes ไม่เปิดเผย internal paths ให้ตรวจครบ

หากสงสัย loop ข้าม module boundary ให้สร้าง flattened diagnostic checkpoint สำหรับ core/integration แล้วตรวจซ้ำ โดยเก็บ hierarchical checkpoint ไว้สำหรับ trace source

---

### 4.11 ขั้นตอนที่ 9 — ตรวจ registers และ reset structures

จาก JSON/RTLIL ให้สำรวจ storage cell types เช่น:

- `$dff`
- `$dffe`
- `$adff`
- `$adffe`
- `$sdff`
- `$sdffe`
- Cell variants ที่ version นั้นสร้าง

ไม่ควรกำหนดว่าต้องมี cell type เดียว เพราะ synthesis transformations อาจสร้างรูปแบบต่างกัน

ตรวจประเด็นหลัก:

| ประเด็น | สิ่งที่ต้องยืนยัน |
|---|---|
| Clock | Registers ที่เกี่ยวข้องใช้ clock ตาม architecture |
| Clock polarity | ตรงกับ RTL |
| Async reset | Polarity และ reset value ถูกต้อง |
| Sync reset | Priority เทียบกับ enable ถูกต้อง |
| Clock enable | ไม่กลายเป็น clock net โดยไม่ได้ตั้งใจ |
| Unreset storage | มี validity/control sequence ป้องกันใช้งานข้อมูลไม่พร้อม |
| Clock domains | จำนวนและหน้าที่ตรงกับ specification |

ห้ามตัดสินว่า register ไม่มี reset คือ error ทุกกรณี ต้องตรวจว่ามี valid-state control หรือ firmware initialization ตาม architecture หรือไม่

การตรวจโครงสร้างนี้ไม่ได้พิสูจน์ recovery/removal timing หรือ reset-domain crossing

---

### 4.12 ขั้นตอนที่ 10 — สร้าง JSON inventory

สร้าง `scripts/lab04_inventory.py`:

```python
import json
import sys
from collections import Counter

if len(sys.argv) != 2:
    raise SystemExit("usage: lab04_inventory.py design.json")

with open(sys.argv[1]) as f:
    design = json.load(f)

def enabled_attribute(value):
    if isinstance(value, int):
        return value != 0
    if isinstance(value, str):
        try:
            return int(value, 2) != 0
        except ValueError:
            return value.lower() in {"true", "yes"}
    return bool(value)

for module_name, module in sorted(design["modules"].items()):
    attrs = module.get("attributes", {})
    cells = module.get("cells", {})

    blackbox = enabled_attribute(attrs.get("blackbox", 0))
    counts = Counter(cell["type"] for cell in cells.values())

    print(f"\nMODULE {module_name}")
    print(f"  blackbox={blackbox}")
    print(f"  local_cell_count={len(cells)}")

    for name, port in sorted(module.get("ports", {}).items()):
        print(
            f"  PORT {name}: "
            f"{port['direction']} width={len(port['bits'])}"
        )

    for cell_type, count in sorted(counts.items()):
        print(f"  CELL {cell_type}: {count}")

    for instance, cell in sorted(cells.items()):
        if cell["type"] in {"$mem", "$mem_v2"}:
            print(f"  MEMORY {instance}: {cell['type']}")

            for key in (
                "WIDTH", "SIZE", "ABITS",
                "RD_PORTS", "WR_PORTS"
            ):
                raw = cell.get("parameters", {}).get(key)

                if raw is not None:
                    value = int(raw, 2) if isinstance(raw, str) else raw
                    print(f"    {key}={value}")

    init_count = sum(
        "init" in net.get("attributes", {})
        for net in module.get("netnames", {}).values()
    )

    print(f"  nets_with_init_attribute={init_count}")
```

รัน:

```bash
python3 "$PROJECT_ROOT/scripts/lab04_inventory.py" \
  "$LAB4_BUILD/core_preopt.json" \
  > "$LAB4_REPORT/core-preopt-inventory.txt"

python3 "$PROJECT_ROOT/scripts/lab04_inventory.py" \
  "$LAB4_BUILD/core_postopt.json" \
  > "$LAB4_REPORT/core-postopt-inventory.txt"
```

Inventory นี้แสดง **local cell counts ต่อ module** ไม่ใช่จำนวน instances ทั้งชิป หาก module ถูก instantiate หลายครั้งต้องคำนวณตาม hierarchy หรือใช้ hierarchical statistics เพิ่ม

ใช้ inventory เพื่อ review ไม่ใช้การมีชื่อ cell เพียงอย่างเดียวเป็นหลักฐานว่า feature ทำงานถูกต้อง

---

### 4.13 ขั้นตอนที่ 11 — ตรวจ memories ก่อน mapping

เปรียบเทียบกับ memory inventory จาก Lab 1:

| รายการ | Expected | Observed | Status |
|---|---|---|---|
| Boot ROM | Capacity/image ตาม configuration | ROM/memory/logic ที่พบ | MATCH/REVIEW |
| IMEM | Depth × width | Memory parameters หรือ storage logic | MATCH/REVIEW |
| DMEM | Depth × width | Memory parameters หรือ storage logic | MATCH/REVIEW |
| Read ports | ตาม RTL | `RD_PORTS` และ connections | MATCH/REVIEW |
| Write ports | ตาม RTL | `WR_PORTS` และ connections | MATCH/REVIEW |
| Byte writes | ตาม architecture | Enable bus width/logic | MATCH/REVIEW |

ROM อาจถูกแปลงเป็น constants หรือ mux logic จึงไม่จำเป็นต้องเหลือ `$mem_v2` เสมอไป ต้อง trace representation ที่ได้จริง

Lab นี้ใช้ `memory_collect` แต่ยังไม่เรียก `memory_map` เพื่อหลีกเลี่ยงการแปลง memories เป็น register/mux structures ก่อนกำหนด memory implementation policy

สำหรับ SRAM macro ให้ตรวจเพิ่ม:

- Model/declaration name
- Data/address widths
- Enable/write-enable polarity
- Read latency
- Byte-mask convention
- Power pins
- Physical views และ version

การที่ macro เป็น blackbox และ hierarchy ผ่าน ไม่ยืนยันว่า protocol หรือ timing ของ macro ตรงกับ RTL

---

### 4.14 ขั้นตอนที่ 12 — วิเคราะห์ logic ที่ optimization ตัดออก

เปรียบเทียบ:

```bash
diff -u \
  "$LAB4_REPORT/core-preopt-inventory.txt" \
  "$LAB4_REPORT/core-postopt-inventory.txt" \
  > "$LAB4_REPORT/inventory-diff.txt"
```

ค้นหา optimization messages:

```bash
rg -n -i \
  'removed|unused|constant|clean|optimized' \
  "$LAB4_REPORT/core-structural.log" \
  > "$LAB4_REPORT/optimization-search.txt"
```

เหตุผลที่ logic หายอาจเป็น:

| กรณี | การตีความ |
|---|---|
| Peripheral ปิดผ่าน generics | คาดหมายได้ |
| GPIO bits ที่ไม่ expose/ไม่ใช้ | ต้องเทียบกับ configuration |
| Constant branches | อาจเป็นผลปกติของ elaboration |
| Duplicate logic | อาจถูก merge |
| UART TX ไม่ต่อถึง top | Integration mistake ที่ต้องแก้ |
| Reset ผูก active ตลอด | อาจทำให้ sequential logic ถูกลดรูป |
| CPU output ไม่ observable | CPU/SoC บางส่วนอาจถูกตัด |
| Memory ไม่มี read path ที่ใช้งาน | ต้องตรวจ boot/data path |

**จุดตรวจสำคัญ:** หาก CPU หรือ UART ที่ควรใช้งานหายไป ไม่ควรแก้ด้วย `keep` ทั้ง design ทันที ต้องตรวจ source list, top selection, reset constants และ output connectivity ก่อน

จำนวน cells ลดลงมากไม่ใช่หลักฐานว่า design ดีขึ้น และไม่ใช่ error โดยอัตโนมัติ ต้องอธิบายด้วย configuration และ connectivity

---

### 4.15 ขั้นตอนที่ 13 — ตรวจ chip integration

ใช้ source ของโครงการจริง เช่น:

```yosys
read_verilog neorv32_asic_core.v
read_verilog -sv chip_core.sv

hierarchy -simcheck -top chip_core
proc -noopt

write_rtlil integration_preopt.il

check -assert
select -assert-none t:$dlatch t:$adlatch t:$dlatchsr

flatten
check -assert
scc -expect 0

opt
check -assert

write_json integration_postopt.json
stat
```

ตัวอย่างนี้ใช้ได้เมื่อ `chip_core` มี dependencies ครบและไม่มี intentional blackboxes หากมี ต้องปรับ policy และ inventory ให้ตรง architecture

ตรวจทีละเส้นทาง:

1. Clock เข้า SoC ถูก net
2. Reset polarity ถูกต้อง
3. UART RX มาจาก input interface ที่กำหนด
4. UART TX ต่อออก wrapper
5. GPIO bits ไม่สลับ
6. Unused inputs มี inactive values ที่ audit แล้ว
7. ไม่มี output net ถูกใช้เป็น input โดยไม่ตั้งใจ
8. ไม่มี connection ทำให้ SoC ถูกตัดออกหลัง `opt`

การ flatten ในขั้นตอนนี้ใช้ช่วยตรวจ cross-module connectivity ส่วน checkpoint ก่อน flatten ใช้ trace hierarchy และ source

---

### 4.16 ขั้นตอนที่ 14 — ตรวจ full-chip hierarchy และ pad blackboxes

Full-chip synthesis อาจใช้ pad/SRAM declarations แบบ blackbox จึงใช้:

```yosys
hierarchy -check -top chip_top
```

แทน no-blackbox gate

อ่าน source และ library declarations ตาม synthesis flow ของ template/PDK ที่ใช้งานจริง โดยรักษา defines และ power-pin conventions ให้ตรงกัน

ตรวจรายการ instantiated blackboxes กับ approved list:

| Instance | Module type | หน้าที่ | Required views | Status |
|---|---|---|---|---|
| Clock pad | ตาม RTL จริง | Clock input | Declaration, Liberty/LEF/GDS ตาม flow | CHECKED |
| Reset pad | ตาม RTL จริง | Reset input | เช่นเดียวกัน | CHECKED |
| UART pads | ตาม RTL จริง | RX/TX | เช่นเดียวกัน | CHECKED |
| GPIO pads | ตาม RTL จริง | Output | เช่นเดียวกัน | CHECKED |
| Power pads | ตาม RTL จริง | Supplies | Physical/power models ตาม methodology | CHECKED |
| SRAM | ถ้ามี | Memory | Logical/timing/physical views | CHECKED |

ห้ามสร้าง empty modules ให้ unknown modules ทั้งหมดเพื่อทำให้ hierarchy ผ่าน เพราะจะซ่อน source omissions

สำหรับ tri-state และ supply nets ให้ใช้ full-chip synthesis methodology ของ template ไม่ใช้ core-only assumptions ตัดสิน physical IO ทั้งหมด

**ข้อจำกัดสำคัญ:** RTL structural connectivity ไม่ยืนยัน PDN/global power connections ใน OpenROAD ต้องตรวจ physical connectivity ใน Lab ที่เกี่ยวข้องต่อไป

---

### 4.17 ขั้นตอนที่ 15 — Audit initialization โดยแยกจาก driver checks

`check -noinit` ใช้ตรวจ wires ที่มี `init` attribute ส่วน `check -initdrv` ตรวจความสัมพันธ์ของ init attributes กับ FF drivers ตาม semantics ของคำสั่ง [YosysHQ Yosys 0.47 documentation](https://yosyshq.readthedocs.io/projects/yosys/en/0.47/cmd/check.html?utm_source=chatgpt.com)

คำสั่งเหล่านี้ไม่ใช่ universal ASIC readiness gate และไม่ครอบคลุม memory initialization ทุก representation

ให้ทำ initialization review แยก:

| Storage | ต้องการค่าเริ่มต้นหรือไม่ | ใครทำให้ค่า valid |
|---|---|---|
| Control/state registers | ตาม reset specification | Reset logic |
| Boot ROM | ต้องมี code | Constant contents/ROM implementation |
| Writable RAM | ไม่สมมติ power-up contents | Bootloader/firmware/test sequence |
| Data buffers | ตาม valid protocol | Producer และ valid bits |
| SRAM macro | ตาม macro specification | Loading/initialization architecture |

หาก methodology กำหนดว่าห้าม FF init attributes ให้ใช้ `check -noinit -assert` เป็น gate ของ checkpoint ที่เหมาะสม แต่ต้องไม่เหมารวมว่า ROM initialization คือ FF init

---

### 4.18 ขั้นตอนที่ 16 — ทำ checker negative test

ตรวจว่า flow ล้มเหลวเมื่อมี structural defect จริง

สร้าง `build/lab04/undriven_example.v`:

```verilog
module undriven_example (
  input  wire clk,
  output reg  q
);

  wire missing_driver;

  always @(posedge clk)
    q <= missing_driver;

endmodule
```

รัน:

```bash
yosys \
  -l "$LAB4_REPORT/negative-undriven.log" \
  -p 'read_verilog undriven_example.v; hierarchy -simcheck -top undriven_example; proc; check -assert'
```

ผลที่ต้องได้:

- พบ `missing_driver` ไม่มี driver
- `check -assert` ล้มเหลว
- Exit code ไม่เป็นศูนย์

เก็บ test fixture นี้แยกจาก project source ห้ามรวมเข้า LibreLane source list

---

### 4.19 การใช้ `check -mapped`

**ยังไม่ใช้ `check -mapped` เป็น gate หลักใน Lab นี้** เพราะ checkpoint ยังมี generic cells เช่น arithmetic, muxes, registers และ memories ที่ยังไม่ได้ map ไปยัง target library

ใช้ option นี้หลัง technology mapping ตาม methodology ของ flow และตรวจ policy สำหรับ macros/IO cells ให้เหมาะสม

การพบ generic cells ก่อน mapping เป็นเรื่องคาดหมายได้ ไม่ควรนับว่าเป็น unmapped-cell failure ของ ASIC release

---

### 4.20 ผลส่งของ Lab

ต้องส่ง:

1. Source manifest และ hashes
2. Yosys version และ command help
3. Core hierarchy/structural logs
4. Pre/post optimization RTLIL และ JSON checkpoints
5. Register/latch/memory inventory
6. Optimization disposition report
7. Integration/full-chip results ตามขอบเขตที่ทำ
8. Approved blackbox inventory
9. Initialization audit
10. Negative-test result

สรุปเป็นตาราง:

| Gate | Core | Integration | Full chip |
|---|---|---|---|
| Hierarchy | PASS/FAIL | PASS/FAIL | PASS/FAIL |
| Blackbox disposition | NONE/APPROVED | NONE/APPROVED | APPROVED/FAIL |
| Driver check | PASS/FAIL | PASS/FAIL | PASS/REVIEW/FAIL |
| Latch policy | PASS/FAIL | PASS/FAIL | PASS/REVIEW |
| Combinational loops | PASS/FAIL | PASS/FAIL | PASS/REVIEW |
| Interface mapping | PASS/FAIL | PASS/FAIL | PASS/FAIL |
| Memory audit | PASS/REVIEW | PASS/REVIEW | PASS/REVIEW |
| Optimization review | PASS/FAIL | PASS/FAIL | PASS/FAIL |

ใช้ `NOT RUN` เมื่อยังไม่ได้ทำ ห้ามใช้ PASS แทนผลของระดับอื่น

### 4.21 เกณฑ์ผ่าน

Lab ผ่านเมื่อ:

- ไม่มี unknown hierarchy
- Intentional blackboxes มี inventory และ views ที่ระบุได้
- ไม่มี unexplained undriven signals หรือ conflicting drivers
- ไม่มี unintended latches
- ไม่มี unexplained combinational loops
- Clock/reset และ interface mapping ตรง specification
- Memories และ initialization assumptions ได้รับการตรวจ
- Logic ที่ถูกตัดออกมีเหตุผลสอดคล้องกับ configuration
- ส่วนที่ต้องใช้งาน เช่น CPU, boot path และ UART ไม่หายเพราะ integration mistake
- Negative test ทำให้ checker ล้มเหลวตามคาด

### 4.22 คำถามท้าย Lab

1. `hierarchy -check` กับ `hierarchy -simcheck` ต่างกันอย่างไร?
2. เหตุใดต้องตรวจโครงสร้างทั้งก่อนและหลัง optimization?
3. Undriven signal ต่างจาก uninitialized register อย่างไร?
4. ทำไมการเพิ่ม `keep` จึงไม่ใช่วิธีแก้แรกเมื่อ CPU ถูกตัดออก?
5. เมื่อใด blackbox จึงถือเป็น intentional macro และเมื่อใดถือเป็น source omission?
6. เหตุใดต้องตรวจ integration หลัง flatten เพิ่ม?
7. Generic cells ก่อน technology mapping ถือเป็น error หรือไม่?
8. ROM ถูกแปลงเป็น mux/constants แล้วไม่เหลือ memory cell ถือว่าผิดหรือไม่?
9. Structural check ผ่านยืนยัน functional correctness ได้มากน้อยเพียงใด?
10. เพราะเหตุใด power-net connectivity ใน RTL จึงยังไม่ยืนยันว่า PDN ผ่าน?