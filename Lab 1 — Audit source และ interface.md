## Lab 1 — Audit source และ interface

Lab นี้เตรียมข้อมูลต้นทางก่อนนำ NEORV32 เข้าสู่กระบวนการ ASIC implementation ด้วย LibreLane และ IHP SG13G2 โดยตรวจให้ทราบว่า **ใช้ source revision ใด เลือก hardware configuration แบบใด และเชื่อมต่อสัญญาณจาก SoC ไปถึงขาชิปอย่างไร**

ผลลัพธ์ของ Lab คือชุดข้อมูลที่ใช้ตรวจสอบและสร้าง design เดิมซ้ำได้ ประกอบด้วย source manifest, configuration specification, interface matrix, memory inventory และรายการประเด็นที่ต้องแก้ก่อน elaboration/synthesis

NEORV32 เป็น SoC ที่เขียนด้วย VHDL และปรับโครงสร้างผ่าน generics ดังนั้น source ชุดเดียวกันอาจสร้างวงจรที่มี CPU extensions, memory และ peripherals ต่างกัน การบันทึกเพียง Git commit จึงยังไม่เพียงพอ ต้องบันทึกค่าของ generics และ memory images ด้วย [GitHub](https://github.com/stnolting/neorv32/blob/main/README.md?utm_source=chatgpt.com)

### 1.1 วัตถุประสงค์

เมื่อจบ Lab ผู้เรียนต้องสามารถ:

1. ระบุ repository, commit และ local modifications ของ source ทุกชุด
2. แยก source สำหรับ implementation ออกจาก testbench และตัวอย่าง FPGA
3. ระบุ top-level entity/module ของแต่ละชั้น
4. ตรวจ configuration generics ให้สอดคล้องกับ firmware และ hardware specification
5. จัดทำ interface matrix ครอบคลุม clock, reset, UART, GPIO และ interface ที่เปิดใช้งาน
6. ระบุ memory ที่ต้องใช้และวิธี initialize หลังผลิตชิป
7. ตรวจความสอดคล้องระหว่าง logical interface, pad instances และ LibreLane configuration
8. สรุปสถานะความพร้อมก่อนเข้าสู่ Lab ถัดไป

**ขอบเขต:** Lab นี้ยังไม่ตัดสินว่า design ผ่าน timing, DRC หรือ LVS และยังไม่ถือว่าการตรวจข้อความด้วย `rg` เป็นหลักฐานว่า RTL elaboration ผ่าน

### 1.2 โครงสร้าง design ที่ใช้ในการตรวจ

กำหนดบทบาทของแต่ละชั้นก่อนเริ่มอ่าน source:

| ชั้น | หน้าที่ | สิ่งที่ต้องตรวจ |
|---|---|---|
| `neorv32_top` | NEORV32 SoC ต้นทาง | Generics, ports, memories และ peripherals |
| ASIC VHDL wrapper | กำหนด configuration ของ SoC | Generic map, port map และ tie-offs |
| Generated Verilog | ผลแปลง VHDL สำหรับ flow | ชื่อ module, ports และความสัมพันธ์กับ source |
| `chip_core` | เชื่อม SoC กับสัญญาณภายใน padframe | Reset polarity, bus slicing และ bit mapping |
| `chip_top` | Top-level ของชิป | Signal pads, power pads และ hierarchy |
| LibreLane configuration | กำหนด implementation | Source list, design name, clock และ pad placement |

ชื่อ ASIC wrapper และ generated module ในตารางเป็นบทบาทที่ต้องกำหนดในโปรเจกต์ ไม่ใช่ชื่อที่รับประกันว่ามีอยู่แล้วใน upstream

NEORV32 มีแนวทางแปลง configuration ที่กำหนดผ่าน VHDL wrapper เป็น Verilog ด้วย GHDL อยู่ในเอกสารต้นทาง จึงควร audit เส้นทางนี้ก่อนกำหนด flow ของโครงการ [GitHub](https://github.com/stnolting/neorv32/blob/main/docs/datasheet/overview.adoc?utm_source=chatgpt.com)

---

### 1.3 ขั้นตอนที่ 1 — กำหนด workspace และตำแหน่ง source

ตัวอย่างต่อไปนี้ใช้โครงสร้าง:

```text
neorv32-ihp/
├── upstream/
│   ├── neorv32/
│   └── ihp-template/
├── rtl/
├── librelane/
├── scripts/
└── reports/
    └── lab01/
```

หากโปรเจกต์มี source อยู่แล้ว ให้ตั้งตัวแปรให้ชี้ตำแหน่งเดิม ไม่ต้อง clone ซ้ำ

```bash
cd /path/to/neorv32-ihp

export PROJECT_ROOT="$PWD"
export NEORV32_ROOT="$PROJECT_ROOT/upstream/neorv32"
export IHP_TEMPLATE_ROOT="$PROJECT_ROOT/upstream/ihp-template"
export AUDIT_DIR="$PROJECT_ROOT/reports/lab01"

mkdir -p "$AUDIT_DIR"
```

ตรวจว่า repository อยู่ครบ:

```bash
test -d "$NEORV32_ROOT"
test -d "$IHP_TEMPLATE_ROOT"

git -C "$NEORV32_ROOT" rev-parse --show-toplevel
git -C "$IHP_TEMPLATE_ROOT" rev-parse --show-toplevel
```

หากเริ่มจาก workspace ใหม่ สามารถเตรียม source ด้วย:

```bash
mkdir -p upstream

git clone https://github.com/stnolting/neorv32.git \
  upstream/neorv32

git clone https://github.com/IHP-GmbH/ihp-sg13-librelane-template.git \
  upstream/ihp-template
```

URL ของ template ที่ผู้ใช้ระบุอาจ redirect ไปยัง repository ที่มีชื่อระบุ SG13G2 ให้บันทึก URL และ revision ที่ checkout จริง เอกสาร IHP ปัจจุบันอ้างถึง template `ihp-sg13g2-librelane-template` [IHP OpenPDK 11181f4 documentation](https://ihp-open-pdk-docs.readthedocs.io/en/latest/digital/librelane_full_chip.html?utm_source=chatgpt.com)

**ผลที่คาดหวัง:** สามารถระบุ absolute path ของ source ทั้งสองชุดได้ และคำสั่ง Git ทำงานภายใน repository ที่ถูกต้อง

---

### 1.4 ขั้นตอนที่ 2 — บันทึก revision และ local modifications

เก็บหลักฐานของแต่ละ repository:

```bash
for spec in \
  "neorv32:$NEORV32_ROOT" \
  "ihp-template:$IHP_TEMPLATE_ROOT"
do
  name="${spec%%:*}"
  repo="${spec#*:}"

  {
    printf 'repository=%s\n' "$name"
    printf 'path=%s\n' "$repo"
    printf 'commit='
    git -C "$repo" rev-parse HEAD
    git -C "$repo" describe --tags --always --dirty
    git -C "$repo" remote -v
    git -C "$repo" status --short
    git -C "$repo" submodule status --recursive
  } > "$AUDIT_DIR/${name}-revision.txt"

  git -C "$repo" diff --binary HEAD \
    > "$AUDIT_DIR/${name}-local.patch"

  git -C "$repo" ls-files --others --exclude-standard \
    > "$AUDIT_DIR/${name}-untracked.txt"
done
```

อ่านผล:

```bash
cat "$AUDIT_DIR/neorv32-revision.txt"
cat "$AUDIT_DIR/ihp-template-revision.txt"
```

ตรวจตามลำดับ:

1. Commit เป็น hash เต็ม 40 ตัวอักษรหรือไม่
2. มี tag ที่สอดคล้องกับ commit หรือไม่
3. Working tree มีการแก้ไขหรือไม่
4. มีไฟล์ untracked ที่ build ต้องใช้งานหรือไม่
5. มี submodule ที่ยังไม่ initialize หรือ revision ไม่ตรงหรือไม่

**ข้อควรเข้าใจ:** `git diff HEAD` ไม่เก็บเนื้อหาไฟล์ untracked หากมี wrapper หรือ memory image ที่ยังไม่ถูก track ต้องเก็บไฟล์นั้นไว้ใน project source และ manifest แยกต่างหาก

หลังเลือกรุ่นสำหรับ Lab แล้ว ให้ใช้ commit นั้นตลอดชุดทดลอง การเปลี่ยน upstream revision ต้อง audit ใหม่ โดยเฉพาะ generic names, ports และ boot configuration

**เกณฑ์ผ่าน:** ทุก repository มี revision ที่ระบุแน่นอน และ local modifications ที่มีผลต่อ build ถูกเก็บครบ

---

### 1.5 ขั้นตอนที่ 3 — สำรวจและจำแนก source

สร้างรายการไฟล์:

```bash
(
  cd "$NEORV32_ROOT"
  rg --files -g '! .git' | sort
) > "$AUDIT_DIR/neorv32-files.txt"

(
  cd "$IHP_TEMPLATE_ROOT"
  rg --files | sort
) > "$AUDIT_DIR/ihp-template-files.txt"
```

สำหรับคำสั่งแรกสามารถใช้ `rg --files | sort` ได้เช่นกัน โดยปกติ `rg` จะละเว้น `.git` อยู่แล้ว

ค้นหา source และ build descriptions:

```bash
rg --files "$NEORV32_ROOT" \
  -g '*.vhd' \
  -g '*.vhdl' \
  -g '*.f' \
  -g '*Makefile*' \
  -g '*.sh' \
  > "$AUDIT_DIR/neorv32-build-input-candidates.txt"

rg --files "$IHP_TEMPLATE_ROOT" \
  -g '*.sv' \
  -g '*.v' \
  -g '*.yaml' \
  -g '*.tcl' \
  -g '*.sdc' \
  > "$AUDIT_DIR/ihp-build-input-candidates.txt"
```

จำแนกไฟล์อย่างน้อยตามตารางนี้:

| ประเภท | ตัวอย่างสิ่งที่พบ | ใช้ใน implementation หรือไม่ |
|---|---|---|
| RTL ของ CPU/SoC | Packages, CPU, interconnect, peripherals | ใช้ตาม configuration และ dependency |
| Memory RTL | RAM, ROM และ image packages | ต้องตรวจวิธี mapping/initialization |
| ASIC wrapper | Generic map และ external ports | ใช้ |
| Padframe RTL | `chip_top`, pad instances | ใช้ใน full-chip flow |
| Testbench | Clock generator, stimulus, assertions | ใช้เฉพาะ verification |
| FPGA integration | PLL, vendor primitives, board wrapper | ต้องประเมินก่อนนำมาใช้ |
| Generated source | Verilog ที่สร้างจาก VHDL | ใช้เมื่อมี provenance ครบ |
| Physical configuration | SDC, PDN, pad placement | ใช้ใน physical flow |

อย่ารวมทุกไฟล์ที่ค้นพบเข้า synthesis ด้วย wildcard เพราะอาจรวมหลาย top-level, testbench หรือ FPGA-specific source โดยไม่ตั้งใจ

---

### 1.6 ขั้นตอนที่ 4 — ตรวจ source list และลำดับ dependency

ค้นหา file lists และคำสั่งที่ build ใช้งานจริง:

```bash
rg --files "$NEORV32_ROOT/rtl" -g '*.f'

rg -n \
  'file_list|ghdl|--std|--work|--synth|--out=verilog' \
  "$NEORV32_ROOT/rtl" \
  "$NEORV32_ROOT/sim" \
  "$NEORV32_ROOT/Makefile" \
  > "$AUDIT_DIR/build-command-references.txt"
```

หาก revision นั้นไม่มีบาง path ให้ค้นหาเฉพาะ path ที่มีอยู่ อย่าสร้างไฟล์ว่างเพื่อให้คำสั่งตรวจผ่าน

อ่าน source list ที่เลือกใช้ แล้วตอบให้ได้ว่า:

- Path ใน file list อ้างอิงจาก directory ใด
- มี package ใดต้อง analyze ก่อน entity
- ใช้ VHDL standard ใด
- ใช้ logical library ชื่อใด
- มี memory image package อยู่ใน dependency หรือไม่
- มี source ที่สร้างระหว่าง build หรือไม่
- มี wrapper และ top entity ที่ตรงกับ configuration หรือไม่

จัดทำ `compile-plan.md`:

```markdown
# Compile plan

- Selected source file list:
- File-list path base:
- VHDL standard:
- Logical library:
- Top entity:
- ASIC wrapper:
- Generated Verilog module:
- Memory image inputs:
- Generation command:
- Tool version:

## Dependency notes

## Excluded verification/FPGA sources

## Unresolved items
```

**หลักสำคัญ:** ลำดับจาก `sort` เป็นเพียงรายการ inventory ไม่ใช่ compile order ให้ใช้ dependency และ build script ของ revision ที่เลือกเป็นหลัก

---

### 1.7 ขั้นตอนที่ 5 — อ่าน top-level entity และ configuration generics

เปิด source จริง:

```bash
less "$NEORV32_ROOT/rtl/core/neorv32_top.vhd"
```

ค้นหาตำแหน่งสำคัญ:

```bash
rg -n -i \
  'entity\s+neorv32_top|generic\s*\(|port\s*\(' \
  "$NEORV32_ROOT/rtl/core/neorv32_top.vhd"

rg -n -i \
  'CLOCK_FREQUENCY|BOOT|RISCV_ISA|IMEM|DMEM|CACHE|IO_|DEBUG|XBUS' \
  "$NEORV32_ROOT/rtl/core/neorv32_top.vhd" \
  > "$AUDIT_DIR/top-configuration-search.txt"
```

ผลค้นหาเป็นดัชนีสำหรับอ่านต่อ ต้องตรวจ declaration, comments และ implementation ที่เกี่ยวข้องด้วย

จัดทำ `soc-configuration.csv`:

```csv
generic,value,unit,reason,evidence,status
CLOCK_FREQUENCY,50000000,Hz,initial lab target,configuration specification,proposed
```

เพิ่มแถวสำหรับ configuration ทุกตัวที่มีผลต่อ design รวมถึงค่าที่ใช้ default

ตัวอย่าง specification สำหรับเริ่มต้น:

| รายการ | เป้าหมายของ Lab | สิ่งที่ต้องยืนยัน |
|---|---|---|
| จำนวน CPU | Single core | Generic ที่ควบคุมตรงกับ revision |
| ISA | กำหนดชุดเดียวสำหรับ hardware และ firmware | CPU extensions กับ `-march` สอดคล้องกัน |
| Clock | 50 MHz เป็นเป้าหมายเริ่มต้น | ไม่ถือว่า timing ผ่านแล้ว |
| Boot | Internal bootloader ผ่าน UART | Boot mode, boot ROM และ memory สำหรับรับโปรแกรม |
| UART | UART0 RX/TX | Peripheral เปิดใช้งานและมี clock metadata ถูกต้อง |
| GPIO | Output 8 bits | Core bus slicing และ pad mapping |
| IMEM/DMEM | กำหนดหลังตรวจ boot/firmware footprint | Capacity, implementation และ initialization |
| Debug | กำหนดว่าจะใช้หรือปิด | Boot/recovery และ unused inputs |
| External bus | กำหนดว่าจะใช้หรือปิด | Protocol และ response inputs |

ตัวอย่าง bootloader setup ของ upstream ใช้ `CLOCK_FREQUENCY`, `BOOT_MODE_SELECT`, `IMEM_EN`, `IMEM_SIZE`, `DMEM_EN`, `DMEM_SIZE`, `IO_GPIO_NUM` และ `IO_UART0_EN` แต่ต้องตรวจชื่อและความหมายกับ commit ที่เลือกก่อนนำไปใช้ [GitHub](https://github.com/stnolting/neorv32/blob/main/rtl/test_setups/neorv32_test_setup_bootloader.vhd?utm_source=chatgpt.com)

ค่าความถี่ต้องสอดคล้องกันทั้งสามส่วน:

$$
T_{\mathrm{clock}}=\frac{10^9}{f_{\mathrm{clock}}}\;\mathrm{ns}
$$

สำหรับ 50 MHz:

$$
T_{\mathrm{clock}}=20\;\mathrm{ns}
$$

จึงต้องตรวจ:

- Hardware configuration ระบุ 50,000,000 Hz
- Testbench ใช้ period 20 ns
- SDC กำหนด period 20 ns ที่ clock port ของ implementation top

ค่า `CLOCK_FREQUENCY` เป็นข้อมูล configuration ของวงจร ไม่ได้สร้าง oscillator หรือบังคับให้ ASIC ทำงานถึงความถี่นั้น

---

### 1.8 ขั้นตอนที่ 6 — จัดทำ interface matrix

สร้างตารางจาก ports ของ wrapper ที่จะ implement แล้ว trace กลับเข้า `neorv32_top`

ตัวอย่าง logical interface สำหรับ UART boot และ GPIO output:

| Logical signal | Direction | Width | ความหมาย | เงื่อนไขเมื่อไม่ใช้งาน |
|---|---|---:|---|---|
| `clk_i` | Input | 1 | Main clock, rising edge | ต้องมี clock ขณะใช้งาน |
| `rstn_i` | Input | 1 | Reset active low | High เมื่อไม่ reset |
| `uart0_rxd_i` | Input | 1 | UART receive | High ในภาวะ idle |
| `uart0_txd_o` | Output | 1 | UART transmit | ตรวจ idle behavior จาก RTL/TB |
| `gpio_o[7:0]` | Output | 8 | Software-controlled outputs | ตรวจ reset/startup behavior |

ชื่อในตารางเป็น logical interface ตัวอย่าง ต้องแทนด้วยชื่อของ wrapper จริงก่อนปิด Lab

สำหรับแต่ละ port ให้บันทึกเพิ่มเติม:

```csv
signal,direction,width,source_endpoint,destination_endpoint,clock_domain,polarity,idle_value,pad_instance,timing_class,evidence,status
```

ต้องตอบได้ว่า:

1. ใครเป็น driver
2. ใครเป็น receiver
3. สัญญาณอยู่ใน clock domain ใด
4. ถ้า asynchronous มี synchronization ที่ใด
5. ถ้าไม่เปิดใช้งาน peripheral สัญญาณถูกจัดการอย่างไร
6. สัญญาณไปถึง pad instance ใด
7. จะใช้ timing assumption แบบใดใน Lab SDC

**กรณี bus slicing:** หาก upstream GPIO output กว้างกว่าขาภายนอก ให้บันทึกชัดเจนว่าใช้ bits ใด ห้ามใช้เพียงคำว่า “ต่อ GPIO 8 ขา” เพราะไม่แสดงลำดับ bit

---

### 1.9 ขั้นตอนที่ 7 — Audit clock และ reset

ค้นหา clock/reset logic:

```bash
rg -n -i \
  'rising_edge|falling_edge|rstn|reset|clock|clkgen' \
  "$NEORV32_ROOT/rtl/core" \
  > "$AUDIT_DIR/clock-reset-search.txt"
```

ตรวจ clock:

- Main clock เข้าสู่ SoC จากจุดใด
- มี generated clocks หรือใช้ clock-enable
- มี clock gating หรือ FPGA clock primitives
- มี debug clock หรือ clock domain อื่น
- มี clock signal ถูกนำไปใช้เป็น data หรือไม่
- ชื่อ clock port ที่ LibreLane เห็นหลังรวม padframe คืออะไร

ตรวจ reset โดย trace จากขาชิปไปถึง SoC:

```text
External reset pin → Input pad → chip_core → ASIC wrapper → NEORV32 reset input
```

บันทึก polarity และ inversion ทุกจุด เช่น หาก `chip_core` ใช้ reset active high แต่ NEORV32 ใช้ active low ต้องมีการแปลง polarity ที่ระบุได้แน่นอน

เอกสาร NEORV32 ปัจจุบันระบุ central reset sequencer ซึ่งรับ external reset แบบ active low โดย internal system reset มี synchronous deassertion ทั้งนี้ต้องตรวจ implementation ใน revision ที่เลือกด้วย [GitHub Pages](https://stnolting.github.io/neorv32/?utm_source=chatgpt.com)

จัดทำ `clock-reset-audit.md`:

```markdown
# Clock and reset audit

## Clock
- Physical clock pin:
- Pad instance:
- Implementation clock port:
- SoC clock port:
- Target frequency:
- Clock domains:
- Clock-enable/generated-clock evidence:

## Reset
- Physical reset pin:
- External polarity:
- Pad instance:
- chip_core polarity:
- SoC polarity:
- Inversion locations:
- Assertion/deassertion behavior:
- Internal reset sources:
- Required verification:

## Open issues
```

การมี reset sequencer ไม่ได้แปลว่า STA/RDC ตรวจผ่านแล้ว ต้องนำข้อมูลนี้ไปกำหนด verification และ timing methodology ต่อไป

---

### 1.10 ขั้นตอนที่ 8 — Audit unused inputs และ tie-offs

ค้นหา ports ของ `neorv32_top` ที่ไม่ได้ expose ออก wrapper แล้วจำแนก:

| ประเภท | วิธีตรวจ |
|---|---|
| Interrupt inputs | Polarity, active condition และ inactive constant |
| UART handshake inputs | เปิดใช้ flow control หรือไม่ |
| External bus response | Interface เปิดหรือปิด และค่าที่ไม่สร้าง transaction ผิด |
| Debug inputs | Debug configuration และ inactive behavior |
| GPIO inputs | มี input GPIO ใช้งานหรือไม่ |
| Bidirectional interfaces | Output-enable polarity และ external pull-up |
| Optional peripheral inputs | Default values และผลหลัง synthesis |

สร้าง `tieoff-matrix.csv`:

```csv
port,width,feature_enabled,connection,inactive_value,reason,evidence,status
```

หลักการตรวจ:

- อย่า tie input ทุกตัวเป็น `'0'` โดยอัตโนมัติ
- อย่าใช้ `open` แทนค่าคงที่สำหรับ input โดยไม่มี declaration/default ที่รองรับ
- Output ที่ไม่ใช้งานอาจต่อ `open` ใน VHDL ได้ตามชนิดและบริบท
- หากอาศัย default input ให้บันทึกค่านั้นและตำแหน่ง source
- หาก interface ถูกเปิดใช้งาน ต้องตรวจ protocol ของ inputs แม้ไม่ expose เป็นขาชิป
- Internal tri-state ต้องพิจารณาแปลงเป็น mux หรือย้ายไปที่ IO pad ตาม architecture

**เกณฑ์ผ่าน:** ไม่มี input ของ design ที่เลือกซึ่งยังไม่ทราบแหล่งขับหรือ inactive behavior

---

### 1.11 ขั้นตอนที่ 9 — Audit memories และ boot path

ค้นหาโครงสร้าง memory และ initialization:

```bash
rg -n -i \
  'imem|dmem|bootrom|bootloader|application_init|image|ram|rom' \
  "$NEORV32_ROOT/rtl" \
  > "$AUDIT_DIR/memory-search.txt"

rg -n -i \
  'initial|readmem|ram_style|rom_style|init|attribute' \
  "$NEORV32_ROOT/rtl" \
  > "$AUDIT_DIR/memory-initialization-search.txt"
```

จัดทำ memory inventory:

| Memory | Capacity | Read/write behavior | Initialization | ASIC implementation plan |
|---|---:|---|---|---|
| Instruction memory | ตาม configuration | ตรวจจาก source | Boot upload หรือ ROM image | ต้องกำหนด |
| Data memory | ตาม configuration | ตรวจ ports/byte writes | Firmware initialization ตาม section | ต้องกำหนด |
| Boot ROM | ตาม revision | Read only | Bootloader image | ตรวจ constant ROM synthesis |
| Register file | ตาม CPU configuration | ตรวจ read/write ports | ตรวจ reset/startup behavior | ตรวจ mapping |
| Cache/storage อื่น | ตาม features | ตรวจ latency/ports | Valid-state initialization | ใช้เมื่อเปิด feature |

สำหรับ RAM ทุกตัวต้องระบุ:

- Depth และ data width
- จำนวน read/write ports
- Read latency
- Byte write enable
- Read-during-write behavior
- Reset behavior
- Initialization assumption
- วิธี mapping ใน flow
- ถ้าใช้ SRAM macro ต้องมี logical และ physical views ใดบ้าง

**จุดสำคัญสำหรับ ASIC:** FPGA RAM initialization ไม่ใช่หลักฐานว่า SRAM บนชิปจะมีข้อมูลหลังเปิดไฟ หาก software ต้องเริ่มจาก RAM ต้องอธิบายว่าใครโหลดโปรแกรมเข้ามาก่อน CPU execute

เขียน boot sequence เป็นลำดับ:

1. Power และ clock อยู่ในสภาวะใช้งานได้
2. Reset ถูก assert
3. Reset ถูก release ตาม implementation
4. CPU เริ่ม fetch ที่ boot entry ที่กำหนด
5. Boot code พร้อมใช้งานใน memory ชนิดใด
6. โปรแกรมถูกโหลดไปที่ใด
7. CPU โอนไป execute application อย่างไร
8. Firmware เตรียม `.data` และ `.bss` ตาม linker/startup code

อย่าสมมติว่า reset ล้าง RAM ทั้งก้อน ต้องตรวจ startup code และ RTL แยกกัน

**เกณฑ์ผ่าน:** มี boot path ที่เป็นไปได้หลังผลิตชิปจริง และไม่มี memory initialization assumption ที่ยังไม่อธิบาย

---

### 1.12 ขั้นตอนที่ 10 — ตรวจ interface ของ IHP template

ค้นหาจุดเชื่อมต่อใน template:

```bash
rg -n \
  'module chip_top|module chip_core|NUM_.*PADS|sg13g2_IOPad|chip_core' \
  "$IHP_TEMPLATE_ROOT/src" \
  > "$AUDIT_DIR/template-interface-search.txt"

rg -n \
  'DESIGN_NAME|VERILOG_FILES|CLOCK_PORT|CLOCK_PERIOD|PAD_SOUTH|PAD_EAST|PAD_NORTH|PAD_WEST|VDD|VSS|PDN' \
  "$IHP_TEMPLATE_ROOT/librelane" \
  > "$AUDIT_DIR/template-config-search.txt"
```

Template แยก `chip_top` ซึ่งประกอบ padframe ออกจาก `chip_core` และกำหนดตำแหน่ง pad ผ่าน `PAD_SOUTH`, `PAD_EAST`, `PAD_NORTH`, `PAD_WEST` ใน configuration จึงต้องตรวจทั้ง RTL และ pad placement ร่วมกัน [GitHub](https://github.com/IHP-GmbH/ihp-sg13g2-librelane-template?utm_source=chatgpt.com)

ตรวจทีละรายการ:

1. `DESIGN_NAME` ตรงกับ full-chip top module
2. `VERILOG_FILES` ครอบคลุม RTL และ generated Verilog ที่ต้องใช้
3. ไม่มี testbench อยู่ใน implementation sources
4. Clock port ใน config ตรงกับ port จริงของ top
5. Signal pad counts ตรงกับจำนวน bit ที่ expose
6. Output-enable ของ bidirectional pad มี polarity ที่ถูกต้อง
7. Power pads แยก core supply และ IO supply ตาม template
8. Pad placement อ้างอิงชื่อ instance ที่มีอยู่จริง
9. Simulation models และ physical views มาจาก PDK ชุดที่สอดคล้องกัน
10. Power-net names และ global connections ใช้ชื่อเดียวกันตลอด hierarchy

สำหรับ interface ตัวอย่างที่ไม่มี GPIO inputs:

$$
N_{\mathrm{input}}=1_{\mathrm{clock}}+1_{\mathrm{reset}}+1_{\mathrm{UART\ RX}}=3
$$

$$
N_{\mathrm{output}}=1_{\mathrm{UART\ TX}}+8_{\mathrm{GPIO}}=9
$$

จำนวนนี้เป็นเพียง signal pads ไม่รวม power/ground pads, corner cells, fillers หรือขาเพิ่มเติมของ design จริง

จัดทำ `pad-interface.csv`:

```csv
logical_signal,bit,chip_port,pad_instance,pad_type,core_net,side,position,polarity,status
```

ถ้าใช้ pad arrays ให้บันทึกชื่อพร้อม index จริง เพื่อใช้เทียบกับ configuration และ netlist ใน Lab ถัดไป

---

### 1.13 ขั้นตอนที่ 11 — ตรวจ license และสร้าง source hashes

ค้นหา license:

```bash
rg --files \
  "$NEORV32_ROOT" \
  "$IHP_TEMPLATE_ROOT" \
  -g '*LICENSE*' \
  -g '*COPYING*' \
  -g '*NOTICE*' \
  > "$AUDIT_DIR/license-files.txt"
```

อ่านเงื่อนไขของแต่ละชุด source และบันทึก:

- License ของ NEORV32 revision ที่เลือก
- License ของ IHP template
- License/usage conditions ของ PDK และ SRAM views ที่ใช้งาน
- Copyright notices ที่ต้องเก็บใน release
- Third-party files ที่มีเงื่อนไขต่างจาก repository หลัก

สร้าง hashes ของ tracked source ที่เป็น build candidates:

```bash
(
  cd "$NEORV32_ROOT"

  git ls-files -z -- \
    rtl sw \
    'LICENSE*' 'COPYING*' 'NOTICE*' |
    xargs -0 -r sha256sum
) > "$AUDIT_DIR/neorv32-source.sha256"

(
  cd "$IHP_TEMPLATE_ROOT"

  git ls-files -z -- \
    src librelane \
    Makefile \
    'LICENSE*' 'COPYING*' 'NOTICE*' |
    xargs -0 -r sha256sum
) > "$AUDIT_DIR/ihp-template-source.sha256"
```

รายการนี้เป็น baseline ของ source candidates ต้องเพิ่ม wrapper, configuration และ generated inputs ของโครงการเองด้วย

เมื่อ generated Verilog ถูกสร้างใน Lab ถัดไป ต้องบันทึกเพิ่ม:

- Source commit
- Wrapper hash
- Generic configuration
- Memory image hashes
- GHDL version
- Generation command
- Output hash

---

### 1.14 ขั้นตอนที่ 12 — สรุป audit และจัดการประเด็นค้าง

สร้าง `audit-summary.md`:

```markdown
# Lab 1 — Source and interface audit

## Source baseline
- NEORV32 commit:
- IHP template commit:
- Local modifications:
- Untracked build inputs:

## Design configuration
- ASIC wrapper:
- Implementation top:
- ISA:
- Clock target:
- Boot mode:
- IMEM:
- DMEM:
- Enabled peripherals:
- Debug configuration:

## Interface
- Clock/reset trace:
- UART mapping:
- GPIO bit mapping:
- Other interfaces:
- Unused input disposition:
- Signal pad count:
- Power-domain mapping:

## Build
- Compile/source list:
- VHDL standard:
- Logical library:
- Conversion route:
- Generated-source provenance:

## Issues
| ID | Issue | Evidence | Impact | Required action | Status |
|---|---|---|---|---|---|

## Gate decision
- PASS / FAIL / BLOCKED
- Reason:
- Reviewer:
- Review date:
```

ตัวอย่างประเด็นที่ต้องบันทึก:

| ID | ประเด็น | ผลกระทบ | การแก้ไข |
|---|---|---|---|
| A01 | Generic ใน wrapper ไม่มีใน revision ที่เลือก | Elaboration ล้มเหลว | แก้ชื่อและตรวจ semantics |
| A02 | UART RX ถูก tie low | Boot communication ใช้งานไม่ได้ | เชื่อม pad หรือกำหนด idle high |
| A03 | Reset polarity กลับด้าน | SoC ค้าง reset หรือเริ่มผิดจังหวะ | แก้ inversion และตรวจ waveform |
| A04 | RAM อาศัย FPGA initialization | ไม่มีโปรแกรมหลังเปิดชิป | กำหนด boot/ROM/loading architecture |
| A05 | File list มี testbench | Synthesis พบ verification constructs | แยก source lists |
| A06 | GPIO bit กับ pad index ไม่ตรงกัน | Firmware ควบคุมผิดขา | แก้ mapping และ specification |
| A07 | Generated Verilog ไม่ทราบ source revision | Rebuild ตรวจสอบไม่ได้ | สร้างใหม่พร้อม provenance |
| A08 | Power-net names ไม่ตรงระหว่างชั้น | PDN/connectivity มีปัญหา | ทำชื่อและ connection ให้สอดคล้อง |

### 1.15 ผลส่งของ Lab

ผู้เรียนต้องส่ง:

1. **Source baseline:** revision reports, patches และรายการ untracked inputs
2. **Build specification:** source inventory และ compile plan
3. **Hardware specification:** configuration generics พร้อมเหตุผล
4. **Interface specification:** interface, tie-off และ pad matrices
5. **Clock/reset audit:** trace, polarity และ timing assumptions
6. **Memory/boot audit:** memory inventory และ boot sequence
7. **Audit summary:** ประเด็นค้าง หลักฐาน และ gate decision

### 1.16 เกณฑ์ผ่าน

Lab 1 ผ่านเมื่อ:

- ระบุ source revisions และ modifications ครบ
- ทราบ top-level ของทุกชั้นที่ใช้ implementation
- มี compile plan ที่ไม่รวม testbench โดยไม่ตั้งใจ
- Configuration generics อ้างอิง revision เดียวกับ source
- ทุก input มี driver, connection หรือ inactive value ที่อธิบายได้
- Clock/reset ถูก trace ครบจนถึง SoC
- Memory และ boot assumptions เหมาะกับ ASIC
- Logical signals และ pad mapping สอดคล้องกัน
- ไม่มีประเด็นระดับ blocker ต่อ elaboration หรือ architecture

**Source audit ผ่านไม่ได้หมายความว่า RTL compile หรือ chip flow ผ่านแล้ว** ขั้นตอนถัดไปต้องตรวจ analysis/elaboration ของ configuration ที่เลือก และยืนยันพฤติกรรมด้วย simulation

### 1.17 คำถามท้าย Lab

1. เหตุใด Git commit เดียวกันจึงสร้าง NEORV32 hardware ต่างกันได้?
2. `neorv32_top`, ASIC wrapper และ `chip_top` มีหน้าที่ต่างกันอย่างไร?
3. หาก `CLOCK_FREQUENCY` ระบุ 50 MHz แต่ SDC ใช้ period 10 ns จะกระทบ hardware/software assumptions อย่างไร?
4. เพราะเหตุใด UART RX ที่ไม่ใช้งานจึงไม่ควร tie low?
5. FPGA memory initialization กับ ASIC power-up memory contents ต่างกันอย่างไร?
6. ถ้า SRAM macro มีขนาดตรงกับ RTL แต่ read latency ต่างกัน สามารถแทนกันโดยตรงได้หรือไม่?
7. เหตุใดต้องตรวจ pad placement configuration ควบคู่กับ padframe RTL?
8. ต้องเก็บข้อมูลใดบ้างจึงพิสูจน์ได้ว่า generated Verilog มาจาก configuration ที่ระบุจริง?