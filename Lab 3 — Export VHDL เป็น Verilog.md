## Lab 3 — Export VHDL เป็น Verilog

Lab นี้นำ NEORV32 configuration ที่ผ่าน VHDL analysis และ ROM smoke จาก Lab 2 มาแปลงเป็น Verilog สำหรับใช้งานต่อใน LibreLane โดยตรวจทั้ง **ความถูกต้องของ interface, โครงสร้างวงจร และพฤติกรรมหลังแปลง**

การ export สำเร็จต้องมีมากกว่าการสร้างไฟล์ `.v` ได้:

1. GHDL synthesis/export จบโดยไม่มี error
2. Verilog parser อ่านไฟล์ได้
3. Top-level ports ตรงกับ specification
4. Hierarchy และ drivers ผ่านการตรวจ
5. Boot ROM smoke บน Verilog ผ่านด้วย configuration เดียวกับ VHDL
6. เก็บ source revision, configuration, tool version และ hashes ครบ

NEORV32 มี conversion setup ใน `rtl/verilog` ซึ่งใช้ VHDL wrapper กำหนด SoC configuration ก่อนเรียก GHDL synthesis การแปลงให้ผลเป็น Verilog ของ configuration ที่ elaborated แล้ว ไม่ใช่การแปล VHDL generics เป็น Verilog parameters แบบตรงตัว [GitHub](https://github.com/stnolting/neorv32/blob/main/docs/datasheet/overview.adoc?utm_source=chatgpt.com)

### 3.1 วัตถุประสงค์

เมื่อจบ Lab ผู้เรียนต้องสามารถ:

- กำหนดขอบเขตของ VHDL top ที่ต้อง export
- สร้าง wrapper ที่มี configuration แน่นอน
- ใช้ GHDL สร้าง Verilog โดยแยก output ออกจาก diagnostics
- ตรวจ ports, bit widths, hierarchy และ memory structures
- รัน ROM smoke บน generated Verilog
- อธิบายข้อจำกัดของ simulation initialization เมื่อใช้กับ ASIC
- เตรียม generated RTL พร้อมหลักฐานสำหรับ Lab synthesis ถัดไป

### 3.2 ขอบเขตของการ export

กำหนดเส้นทางของโครงการดังนี้:

```text
NEORV32 VHDL + configuration wrapper
                 ↓ GHDL synthesis/export
Generated Verilog ของ SoC core
                 ↓
chip_core + chip_top + IHP IO pads
                 ↓
LibreLane synthesis และ physical implementation
```

ใน Lab นี้ export เฉพาะ SoC core โดยยังไม่รวม:

- Testbench
- Clock/reset stimulus
- UART receiver ของ testbench
- IHP padframe
- Physical power connections
- Standard-cell mapping

**Generated Verilog ยังเป็น technology-independent representation** ต้องนำไปผ่าน technology mapping และตรวจ physical implementation ต่อไป

---

### 3.3 ขั้นตอนที่ 1 — ตรวจเงื่อนไขก่อนเริ่ม

ต้องมีผลจาก Lab 2:

| รายการ | เงื่อนไข |
|---|---|
| Source revision | ระบุ commit แน่นอน |
| VHDL analysis/elaboration | ผ่าน |
| ROM smoke | ผ่านบน configuration ที่จะ export |
| Boot ROM image | ระบุที่มาและ hash |
| Clock configuration | ตรงกับ simulation และ hardware specification |
| Interface | ตรวจชื่อ, direction และ width แล้ว |
| Diagnostics | ไม่มี error หรือ blocker ที่ยังไม่อธิบาย |

หาก Lab 2 ผ่านเฉพาะ upstream baseline แต่ยังไม่ได้ทดสอบ ASIC wrapper ให้ทดสอบ wrapper ก่อนใช้เป็น export top

ไม่ควรเปลี่ยน ISA, memory size หรือ boot mode ระหว่าง VHDL smoke กับ Verilog export โดยไม่ทดสอบ configuration ใหม่

---

### 3.4 ขั้นตอนที่ 2 — เตรียม workspace และเครื่องมือ

```bash
cd /path/to/neorv32-ihp

export PROJECT_ROOT="$PWD"
export NEORV32_ROOT="$PROJECT_ROOT/upstream/neorv32"
export LAB3_BUILD="$PROJECT_ROOT/build/lab03"
export LAB3_REPORT="$PROJECT_ROOT/reports/lab03"

mkdir -p "$PROJECT_ROOT/rtl"
mkdir -p "$PROJECT_ROOT/tb"
mkdir -p "$LAB3_BUILD/lib/neorv32"
mkdir -p "$LAB3_REPORT"

set -o pipefail
```

ตรวจเครื่องมือ:

```bash
command -v ghdl
command -v yosys
command -v iverilog
command -v vvp

ghdl --version > "$LAB3_REPORT/ghdl-version.txt"
yosys -V > "$LAB3_REPORT/yosys-version.txt"
iverilog -V > "$LAB3_REPORT/iverilog-version.txt" 2>&1
```

ตรวจ GHDL synthesis command:

```bash
ghdl --synth --help
```

หาก GHDL version นั้นไม่รองรับ help ในรูปแบบนี้ ให้ใช้:

```bash
ghdl help synth
```

ต้องยืนยันว่า build ที่ใช้อยู่รองรับ synthesis และ Verilog output GHDL ระบุว่า synthesis kernel ยังเป็น experimental จึงต้องตรวจ generated output ด้วยเครื่องมือ downstream และ simulation เสมอ [7.0.0-dev](https://ghdl.github.io/ghdl/using/Synthesis.html?utm_source=chatgpt.com)

---

### 3.5 ขั้นตอนที่ 3 — ตรวจ conversion flow ของ upstream

อ่านไฟล์ conversion ที่มากับ revision:

```bash
rg --files "$NEORV32_ROOT/rtl/verilog"

less "$NEORV32_ROOT/rtl/verilog/Makefile"
less "$NEORV32_ROOT/rtl/verilog/neorv32_verilog_wrapper.vhd"
```

ค้นหาคำสั่งสำคัญ:

```bash
rg -n \
  'ghdl|GHDL|synth|out=verilog|std=|work|file_list|convert' \
  "$NEORV32_ROOT/rtl/verilog"
```

ตรวจ:

1. Wrapper top ชื่ออะไร
2. ใช้ source list ใด
3. วิเคราะห์เข้า library ใด
4. ใช้ VHDL standard ใด
5. มีการแทนบาง VHDL modules ด้วย Verilog IP หรือไม่
6. มี options ที่เปลี่ยน treatment ของ assertions หรือ blackboxes หรือไม่
7. Output ถูกสร้างที่ใด

เอกสาร upstream แสดงคำสั่ง `make convert` สำหรับ predefined wrapper แต่ configuration และ interface ของ wrapper นั้นอาจต่างจาก ASIC project จึงต้องอ่านก่อนเรียกใช้งาน [GitHub](https://github.com/stnolting/neorv32/blob/main/docs/datasheet/overview.adoc?utm_source=chatgpt.com)

สำหรับคู่มือนี้ใช้คำสั่ง export ที่ระบุชัดเจน เพื่อให้ trace configuration และ output ได้ง่าย

---

### 3.6 ขั้นตอนที่ 4 — กำหนด export wrapper

หากมี ASIC wrapper ที่ผ่าน Lab 2 แล้ว ให้ใช้ wrapper นั้นโดยตรง

สำหรับ baseline ที่ต่อเนื่องจาก Lab 2 สามารถสร้าง fixed-configuration wrapper ดังนี้:

ไฟล์ `rtl/neorv32_asic_core.vhd`:

```vhdl
library ieee;
use ieee.std_logic_1164.all;

library neorv32;

entity neorv32_asic_core is
  port (
    clk_i       : in  std_ulogic;
    rstn_i      : in  std_ulogic;
    uart0_rxd_i : in  std_ulogic;
    uart0_txd_o : out std_ulogic;
    gpio_o      : out std_ulogic_vector(7 downto 0)
  );
end entity;

architecture rtl of neorv32_asic_core is
begin

  soc : entity neorv32.neorv32_test_setup_bootloader
    generic map (
      CLOCK_FREQUENCY => 50_000_000,
      IMEM_SIZE       => 16 * 1024,
      DMEM_SIZE       => 8 * 1024
    )
    port map (
      clk_i       => clk_i,
      rstn_i      => rstn_i,
      gpio_o      => gpio_o,
      uart0_txd_o => uart0_txd_o,
      uart0_rxd_i => uart0_rxd_i
    );

end architecture;
```

Wrapper นี้ reuse configuration ภายใน upstream bootloader setup จึงต้องเก็บ setup file และ source commit เป็นส่วนหนึ่งของ baseline

สำหรับ implementation configuration ที่ต้องการควบคุม generics ทุกตัวโดยตรง อาจ instantiate `neorv32_top` ใน wrapper แทน แต่ต้องใช้ generic map และ tie-offs ที่ audit แล้วจาก Lab 1

**ข้อกำหนดของ wrapper:**

- มี ports เฉพาะที่โครงการใช้งาน
- Configuration แน่นอนและตรวจสอบได้
- ไม่มี simulation-only process
- ไม่สร้าง clock ภายในด้วย `after`
- ไม่มี testbench UART decoder
- Reset polarity ตรงกับ specification
- ไม่อาศัย source defaults ที่เปลี่ยนได้โดยไม่มีการบันทึก

ค่าขนาด memory ในตัวอย่างเป็น baseline สำหรับ conversion ไม่ใช่ข้อสรุปว่าเหมาะกับพื้นที่ die หรือ SRAM architecture ของ ASIC แล้ว

---

### 3.7 ขั้นตอนที่ 5 — รัน VHDL smoke บน wrapper เดียวกัน

ก่อน export ให้แก้ DUT instance ใน testbench ของ Lab 2:

```vhdl
dut : entity neorv32.neorv32_asic_core
  port map (
    clk_i       => clk,
    rstn_i      => rstn,
    gpio_o      => gpio,
    uart0_txd_o => tx,
    uart0_rxd_i => rx
  );
```

Wrapper ตัวอย่างไม่มี generics ที่ top level จึงต้องนำ `generic map` เดิมของ DUT ออก

กำหนด testbench clock เป็น 50 MHz และ UART decoder ตาม boot ROM image จริง

**เกณฑ์ผ่านก่อน export:** Testbench ต้องได้รับ expected startup marker จาก wrapper นี้ ไม่ใช้ผลของ DUT คนละตัวแทนกัน

หากต้องเปลี่ยน clock ในภายหลัง ต้องเปลี่ยนค่าภายใน wrapper แล้ว export ใหม่ พร้อมทดสอบใหม่

---

### 3.8 ขั้นตอนที่ 6 — Import และ elaborate export top

ใช้ build directory ใหม่ แยกจาก Lab 2 เพื่อป้องกันการใช้ library database ที่ค้างจาก configuration อื่น:

```bash
cd "$LAB3_BUILD"

ghdl -i \
  --std=08 \
  --work=neorv32 \
  --workdir="$LAB3_BUILD/lib/neorv32" \
  "$NEORV32_ROOT"/rtl/core/*.vhd \
  "$NEORV32_ROOT/rtl/test_setups/neorv32_test_setup_bootloader.vhd" \
  "$PROJECT_ROOT/rtl/neorv32_asic_core.vhd" \
  2>&1 | tee "$LAB3_REPORT/import.log"
```

จากนั้น:

```bash
ghdl -m \
  --std=08 \
  --work=neorv32 \
  --workdir="$LAB3_BUILD/lib/neorv32" \
  neorv32_asic_core \
  2>&1 | tee "$LAB3_REPORT/elaboration.log"
```

หาก compile plan ของ revision มี dependencies เพิ่ม ต้องเพิ่ม source ตามแผนนั้น

ขั้นตอนนี้ตรวจว่า top ที่จะ export สามารถ elaborate ได้ แต่ยังไม่ยืนยันว่าทุก construct อยู่ใน synthesis-supported subset

---

### 3.9 ขั้นตอนที่ 7 — Export เป็น Verilog อย่างปลอดภัย

GHDL ส่ง generated HDL ทาง stdout จึงต้องแยก stdout ออกจาก stderr

```bash
cd "$LAB3_BUILD"

if ghdl --synth \
  --std=08 \
  --work=neorv32 \
  --workdir="$LAB3_BUILD/lib/neorv32" \
  --out=verilog \
  neorv32_asic_core \
  > "$LAB3_BUILD/neorv32_asic_core.v.tmp" \
  2> "$LAB3_REPORT/export.log"
then
  if test -s "$LAB3_BUILD/neorv32_asic_core.v.tmp"
  then
    mv "$LAB3_BUILD/neorv32_asic_core.v.tmp" \
       "$LAB3_BUILD/neorv32_asic_core.v"
  else
    printf 'ERROR: export produced an empty file\n' >&2
    exit 1
  fi
else
  printf 'ERROR: GHDL export failed; inspect export.log\n' >&2
  exit 1
fi
```

เหตุผลที่ใช้ temporary file:

- ไม่แทนที่ output เดิมด้วยไฟล์ที่ export ไม่ครบ
- ไม่ถือว่าไฟล์มีอยู่เท่ากับ export ผ่าน
- ตรวจ exit status และขนาดไฟล์ก่อนยอมรับผล

**ห้ามใช้รูปแบบนี้:**

```bash
ghdl --synth ... > output.v 2>&1
```

เพราะ diagnostics อาจถูกเขียนรวมลงใน Verilog และทำให้ parser ล้มเหลว หรือสร้างไฟล์ที่ปะปนกับข้อความ log

ตรวจผล:

```bash
wc -l "$LAB3_BUILD/neorv32_asic_core.v"

rg -n \
  '^[[:space:]]*module|^[[:space:]]*endmodule' \
  "$LAB3_BUILD/neorv32_asic_core.v"

cat "$LAB3_REPORT/export.log"
```

หนึ่ง output file อาจมีหลาย module definitions ภายใน ไม่ควรสมมติว่า single file หมายถึง hierarchy ถูก flatten เป็น module เดียวทั้งหมด

---

### 3.10 ขั้นตอนที่ 8 — ตรวจ diagnostics จาก synthesis/export

ค้นหาข้อความสำคัญ:

```bash
rg -n -i \
  'error|warning|unsupported|latch|memory|rom|ram|blackbox|unbound' \
  "$LAB3_REPORT/export.log"
```

จัดทำตาราง review:

| Diagnostic | ต้องตรวจอะไร |
|---|---|
| Unsupported construct | เป็น RTL จริงหรือ simulation-only code |
| Inferred latch | มี assignment ไม่ครบหรือเป็นโครงสร้างที่ตั้งใจ |
| ROM/RAM inference | Capacity, width และ latency ตรงกับ specification |
| Unbound component | Source หรือ binding หาย |
| Blackbox | ตั้งใจใช้ external IP หรือเกิดจาก source ไม่ครบ |
| Assertion-related message | Configuration assertion หรือ hardware assertion |
| Ignored attribute | Attribute นั้นจำเป็นต่อ architecture หรือไม่ |

อย่าเพิ่ม options เพื่อปิดข้อความทั้งหมดก่อนเข้าใจสาเหตุ โดยเฉพาะการปิด assertions หรือทำ modules เป็น blackboxes อาจเปลี่ยนสิ่งที่กำลังตรวจ

**เกณฑ์ผ่าน:** ไม่มี error และไม่มี diagnostic สำคัญที่ยังไม่ได้ระบุ disposition

---

### 3.11 ขั้นตอนที่ 9 — ตรวจ Verilog syntax และ hierarchy

เริ่มด้วย Icarus Verilog:

```bash
iverilog \
  -g2012 \
  -s neorv32_asic_core \
  -o "$LAB3_BUILD/core_parse.vvp" \
  "$LAB3_BUILD/neorv32_asic_core.v" \
  2>&1 | tee "$LAB3_REPORT/iverilog-parse.log"
```

ขั้นตอนนี้ตรวจ parser และ elaboration ของ generated modules โดยยังไม่ได้ทดสอบ boot behavior

จากนั้นตรวจด้วย Yosys:

```bash
cd "$LAB3_BUILD"

cat > check_export.ys <<'EOF'
read_verilog neorv32_asic_core.v
hierarchy -check -top neorv32_asic_core
proc
check -assert
stat
write_json export_structure.json
EOF
```

รัน:

```bash
yosys \
  -l "$LAB3_REPORT/yosys-check.log" \
  -s "$LAB3_BUILD/check_export.ys"
```

บทบาทของแต่ละคำสั่ง:

| คำสั่ง | หน้าที่ |
|---|---|
| `read_verilog` | อ่าน generated HDL |
| `hierarchy -check` | ตรวจ hierarchy และ module references |
| `proc` | แปลง procedural logic เป็น internal representation |
| `check -assert` | ให้ structural problems ที่ตรวจพบเป็น error |
| `stat` | แสดงโครงสร้างก่อน technology mapping |
| `write_json` | เก็บ ports/cells สำหรับตรวจต่อ |

หาก GHDL version สร้าง constructs ที่ Verilog frontend ปกติอ่านไม่ได้ ต้องตรวจว่าต้องใช้ frontend/options ใดตาม flow ที่เลือก อย่าเปลี่ยน parser mode เพื่อข้ามปัญหาโดยไม่มีการบันทึก

การผ่าน `check` เป็น structural evidence ยังไม่ใช่ functional equivalence หรือ timing proof

---

### 3.12 ขั้นตอนที่ 10 — ตรวจ ports จาก JSON

สำหรับ wrapper ตัวอย่าง ต้องได้:

| Port | Direction | Width |
|---|---|---:|
| `clk_i` | Input | 1 |
| `rstn_i` | Input | 1 |
| `uart0_rxd_i` | Input | 1 |
| `uart0_txd_o` | Output | 1 |
| `gpio_o` | Output | 8 |

ตรวจอัตโนมัติ:

```bash
python3 - <<'PY'
import json

with open("export_structure.json") as f:
    design = json.load(f)

ports = design["modules"]["neorv32_asic_core"]["ports"]

expected = {
    "clk_i": ("input", 1),
    "rstn_i": ("input", 1),
    "uart0_rxd_i": ("input", 1),
    "uart0_txd_o": ("output", 1),
    "gpio_o": ("output", 8),
}

actual = {
    name: (port["direction"], len(port["bits"]))
    for name, port in ports.items()
}

for name, spec in sorted(actual.items()):
    print(name, spec)

if actual != expected:
    raise SystemExit(
        f"PORT_CHECK_FAIL\nexpected={expected}\nactual={actual}"
    )

print("PORT_CHECK_PASS")
PY
```

สำหรับ wrapper อื่น ให้เปลี่ยน `expected` ตาม interface specification จาก Lab 1

นอกจาก width ต้องตรวจ:

- Vector range และ bit ordering
- GPIO slicing
- Reset inversion
- Signed/unsigned interpretation ของ interfaces ที่เกี่ยวข้อง
- ชื่อ ports ที่ `chip_core` จะ instantiate

อย่าใช้ชื่อ internal signals ที่ GHDL สร้างเป็น interface contract เพราะอาจเปลี่ยนเมื่อเปลี่ยน GHDL version หรือ configuration

---

### 3.13 ขั้นตอนที่ 11 — Audit ROM, RAM และ initialization

ค้นหา memory constructs:

```bash
rg -n \
  'initial|readmem|reg .*\\[|always|case' \
  "$LAB3_BUILD/neorv32_asic_core.v" \
  > "$LAB3_REPORT/memory-construct-search.txt"
```

ใช้ผลค้นหาเป็นจุดเริ่มต้น แล้วอ่าน module ที่เกี่ยวข้องจริง

จัดทำตาราง:

| รายการ | VHDL specification | Generated representation | สิ่งที่ต้องยืนยัน |
|---|---|---|---|
| Boot ROM | Image และ address mapping | Array/constants/case logic | Contents และ read behavior |
| IMEM | Depth, width, writes | Memory หรือ logic | Ports, byte enables, latency |
| DMEM | Depth, width, writes | Memory หรือ logic | Read-during-write และ reset |
| Register file | CPU configuration | Storage/logic | Read/write semantics |
| Initialization | Image/constants/reset | `initial` หรือ logic อื่น | ASIC realizability |

**จุดสำคัญ:** การพบ `initial` ไม่ได้แปลว่าทุกกรณีผิดสำหรับ ASIC ต้องแยกสองกรณี:

1. **Constant ROM contents** ซึ่งอาจถูก synthesis เป็น combinational/sequential ROM logic ที่ realizable ได้
2. **Power-up state ของ writable RAM/registers** ซึ่งต้องมี implementation ที่รองรับจริง หรือมี boot/reset sequence เตรียมข้อมูล

หาก Verilog smoke ผ่านเพราะ RAM ถูก initialize ใน simulation แต่ SRAM macro จริงไม่มี initial contents ต้องแก้ architecture หรือ loading sequence ก่อนใช้ผลนั้นเป็น ASIC evidence

อย่าใช้ `sed` ลบ `initial` ทุก block เพราะอาจลบ boot ROM contents และทำให้ design เปลี่ยนพฤติกรรม

ยังไม่ต้อง map RAM ทั้งหมดเป็น flip-flops ใน Lab นี้ หากโครงการมีแผนใช้ SRAM macro ต้องรักษาและ audit memory abstraction ตามแผนนั้น

---

### 3.14 ขั้นตอนที่ 12 — สร้าง Verilog ROM smoke testbench

สร้าง `tb/tb_rom_smoke_verilog.sv`:

```systemverilog
`timescale 1ns/1ps

module tb_rom_smoke_verilog;

  // Must match the fixed VHDL wrapper configuration.
  localparam real CLK_PERIOD_NS = 20.0;
  localparam real UART_BIT_NS   = 1.0e9 / 19200.0;

  // Seven-byte marker. Change together with its width if needed.
  parameter logic [55:0] EXPECTED_MARKER = "NEORV32";

  logic clk  = 1'b0;
  logic rstn = 1'b0;
  logic rx   = 1'b1;

  wire tx;
  wire [7:0] gpio;

  logic [55:0] window = '0;
  logic [7:0] decoded;

  integer capture_fd;
  integer byte_count = 0;

  neorv32_asic_core dut (
    .clk_i       (clk),
    .rstn_i      (rstn),
    .uart0_rxd_i (rx),
    .uart0_txd_o (tx),
    .gpio_o      (gpio)
  );

  always #(CLK_PERIOD_NS / 2.0) clk = ~clk;

  initial begin
    capture_fd = $fopen("uart_verilog_capture.txt", "w");

    if (capture_fd == 0)
      $fatal(1, "Cannot open UART capture file");

    repeat (20) @(posedge clk);
    @(negedge clk);

    rstn = 1'b1;
    $display("Reset released");
  end

  initial begin : uart_receiver
    wait (rstn === 1'b1);

    forever begin
      @(negedge tx);

      #(UART_BIT_NS / 2.0);

      if (tx !== 1'b0)
        $fatal(1, "UART false start or baud mismatch");

      decoded = '0;

      for (integer bit_index = 0;
           bit_index < 8;
           bit_index = bit_index + 1) begin

        #(UART_BIT_NS);

        if ((tx !== 1'b0) && (tx !== 1'b1))
          $fatal(1, "Unknown UART data bit");

        decoded[bit_index] = tx;
      end

      #(UART_BIT_NS);

      if (tx !== 1'b1)
        $fatal(1, "UART stop-bit error");

      byte_count = byte_count + 1;

      $fwrite(capture_fd, "%c", decoded);
      $display("UART byte %0d = %0d",
               byte_count, decoded);

      window = {window[47:0], decoded};

      if (window === EXPECTED_MARKER) begin
        $display(
          "VERILOG_ROM_SMOKE_PASS: expected marker received"
        );

        $fclose(capture_fd);
        $finish;
      end
    end
  end

  initial begin : watchdog
    #100_000_000; // 100 ms
    $fatal(1, "VERILOG_ROM_SMOKE_FAIL: startup timeout");
  end

endmodule
```

Testbench นี้ใช้ marker 7 bytes หาก expected string ของโครงการต่างออกไป ต้องเปลี่ยน width และ shift window ให้ตรงกัน หรือใช้ checker ที่รองรับความยาวข้อความทั่วไป

ค่าความถี่ของ Verilog testbench ต้องตรงกับ fixed configuration ที่ export แล้ว การเปลี่ยน clock ใน testbenchเพียงอย่างเดียวไม่ได้เปลี่ยน `CLOCK_FREQUENCY` ที่ฝังอยู่ใน generated design

---

### 3.15 ขั้นตอนที่ 13 — Compile และ run generated Verilog

```bash
cd "$LAB3_BUILD"

iverilog \
  -g2012 \
  -s tb_rom_smoke_verilog \
  -o "$LAB3_BUILD/rom_smoke.vvp" \
  "$LAB3_BUILD/neorv32_asic_core.v" \
  "$PROJECT_ROOT/tb/tb_rom_smoke_verilog.sv" \
  2>&1 | tee "$LAB3_REPORT/verilog-tb-build.log"
```

รัน:

```bash
vvp "$LAB3_BUILD/rom_smoke.vvp" \
  2>&1 | tee "$LAB3_REPORT/verilog-rom-smoke.log"
```

ตรวจผล:

```bash
rg -n \
  'VERILOG_ROM_SMOKE_PASS|VERILOG_ROM_SMOKE_FAIL|FATAL' \
  "$LAB3_REPORT/verilog-rom-smoke.log"

cat "$LAB3_BUILD/uart_verilog_capture.txt"
```

ต้องผ่านพร้อมกัน:

- Compile exit code เป็นศูนย์
- Simulation exit code เป็นศูนย์
- พบ `VERILOG_ROM_SMOKE_PASS`
- ไม่มี framing/unknown-bit failure
- UART capture มี expected marker
- Configuration ตรงกับ VHDL reference

ไม่ควรใช้ไฟล์ `.vvp` เดิมหลัง compile ล้มเหลว หากเขียน run script ให้ใช้ `set -euo pipefail` หรือจัดการ exit status ก่อนเริ่ม simulation

---

### 3.16 ขั้นตอนที่ 14 — เปรียบเทียบกับ VHDL reference

ใช้ stimulus และ pass condition เดียวกัน:

| รายการ | VHDL | Verilog |
|---|---|---|
| Wrapper configuration | เดียวกัน | Export จาก wrapper เดียวกัน |
| Clock | 50 MHz | 50 MHz |
| Reset | Low 20 cycles แล้ว release ที่ falling edge | เหมือนกัน |
| UART RX | Idle high | Idle high |
| UART format | 19,200 8-N-1 ตาม image | เหมือนกัน |
| Expected marker | จาก bootloader baseline | ข้อความเดียวกัน |
| Timeout | 100 ms | 100 ms |

หาก capture ของทั้งสอง testbench หยุดตรง marker เดียวกัน สามารถเปรียบเทียบ raw bytes:

```bash
cmp \
  "$PROJECT_ROOT/build/lab02/uart_capture.txt" \
  "$LAB3_BUILD/uart_verilog_capture.txt"
```

คำสั่งนี้มีความหมายต่อเมื่อ Lab 2 capture มาจาก **wrapper และ configuration เดียวกัน** และใช้จุดเริ่ม/หยุด capture เดียวกัน

กรณีข้อความมีข้อมูลแปรผัน ให้เปรียบเทียบส่วน deterministic ที่กำหนดไว้ใน test plan

**ข้อจำกัด:** ROM smoke ที่ได้ข้อความตรงกันยืนยัน startup path ที่ทดสอบ ยังไม่ใช่ formal equivalence ของทั้ง SoC

หากต้องเปรียบเทียบ timing ให้ใช้เวลา reset release, first UART start bit และ decoded-byte sequence อย่าเปรียบเทียบ internal node names โดยตรงระหว่าง VHDL และ generated Verilog

---

### 3.17 ขั้นตอนที่ 15 — ตรวจ negative test ของ Verilog checker

เปลี่ยน expected marker ให้เป็นข้อความ 7 bytes ที่ไม่มีใน startup output:

```bash
iverilog \
  -g2012 \
  -s tb_rom_smoke_verilog \
  '-Ptb_rom_smoke_verilog.EXPECTED_MARKER="INVALID"' \
  -o "$LAB3_BUILD/rom_smoke_negative.vvp" \
  "$LAB3_BUILD/neorv32_asic_core.v" \
  "$PROJECT_ROOT/tb/tb_rom_smoke_verilog.sv"
```

รัน:

```bash
vvp "$LAB3_BUILD/rom_smoke_negative.vvp" \
  2>&1 | tee "$LAB3_REPORT/verilog-negative.log"
```

ผลที่ต้องได้:

- ไม่มี PASS
- ถึง watchdog timeout
- Simulator exit code ไม่เป็นศูนย์

ตรวจรูปแบบ parameter override กับ Icarus version ที่ใช้งาน หากไม่รองรับ ให้สร้าง testbench variant ที่เปลี่ยน marker อย่างชัดเจน และเก็บ source ของ variant นั้นเป็นหลักฐาน

---

### 3.18 ขั้นตอนที่ 16 — เก็บ provenance และ release candidate

เก็บ hashes:

```bash
sha256sum \
  "$PROJECT_ROOT/rtl/neorv32_asic_core.vhd" \
  "$LAB3_BUILD/neorv32_asic_core.v" \
  "$PROJECT_ROOT/tb/tb_rom_smoke_verilog.sv" \
  > "$LAB3_REPORT/artifacts.sha256"

git -C "$NEORV32_ROOT" rev-parse HEAD \
  > "$LAB3_REPORT/neorv32-commit.txt"

git -C "$NEORV32_ROOT" diff --binary HEAD \
  > "$LAB3_REPORT/neorv32-local.patch"
```

เก็บ memory image hashes ตามรายการจาก Lab 1 ด้วย โดยใช้ชื่อไฟล์จริงของ revision นั้น

สร้าง `export-manifest.md`:

```markdown
# VHDL-to-Verilog export manifest

## Inputs
- NEORV32 commit:
- Local modifications:
- VHDL wrapper:
- Wrapper hash:
- Configuration:
- Source list:
- Boot ROM image/hash:
- Other memory images/hashes:

## Tools
- GHDL version/backend:
- Yosys version:
- Icarus Verilog version:

## Export
- Working directory:
- Exact command:
- Output file:
- Output hash:

## Verification
- VHDL wrapper smoke:
- Verilog parser:
- Yosys hierarchy/structural check:
- Port check:
- Memory initialization review:
- Verilog ROM smoke:
- Negative test:
- Reference comparison:

## Open issues

## Decision
- ACCEPTED / REJECTED / BLOCKED
```

เมื่อผ่านทุก gate แล้ว จึงคัดลอก output ที่ตรวจแล้วไปยังตำแหน่ง implementation input:

```bash
mkdir -p "$PROJECT_ROOT/rtl/generated"

cp "$LAB3_BUILD/neorv32_asic_core.v" \
   "$PROJECT_ROOT/rtl/generated/neorv32_asic_core.v"

cmp \
  "$LAB3_BUILD/neorv32_asic_core.v" \
  "$PROJECT_ROOT/rtl/generated/neorv32_asic_core.v"
```

อย่าแก้ generated Verilog ด้วยมือโดยไม่มี patch และเหตุผล หากพบปัญหาควรแก้ wrapper/source หรือเลือก tool version ที่ผ่านการตรวจ แล้ว export ใหม่

---

### 3.19 การเตรียมส่งต่อให้ LibreLane

ใน Lab ถัดไปให้ `chip_core` instantiate generated module ด้วย named port connections:

```systemverilog
neorv32_asic_core u_soc (
  .clk_i       (core_clk),
  .rstn_i      (core_rstn),
  .uart0_rxd_i (core_uart_rx),
  .uart0_txd_o (core_uart_tx),
  .gpio_o      (core_gpio)
);
```

ตัวอย่างชื่อ nets ต้องแทนด้วยชื่อจริงใน IHP integration wrapper

ตรวจเพิ่มเติมก่อน handoff:

- `VERILOG_FILES` มี generated source เพียงชุดที่เลือก
- ไม่รวม generated modules รุ่นเก่าที่ชื่อซ้ำ
- ไม่รวม testbench
- `DESIGN_NAME` ของ full-chip flow ยังอ้างถึง `chip_top`
- Clock/reset ผ่าน IO pads และ `chip_core` ถูกต้อง
- Memory mapping policy ถูกกำหนดแล้ว
- Configuration ที่ firmware ใช้ตรงกับ hardware ที่ export

---

### 3.20 ปัญหาที่พบบ่อย

| อาการ | สาเหตุที่ควรตรวจ | วิธีแก้ |
|---|---|---|
| GHDL simulation ผ่านแต่ export fail | Simulation constructs อยู่นอก synthesizable subset | Export RTL top และตรวจ source ที่เกี่ยวข้อง |
| Output `.v` ว่างหรือไม่ครบ | Export ล้มเหลวแต่ script ยอมรับไฟล์ | ตรวจ exit status และใช้ temporary output |
| Parser พบข้อความแปลกใน `.v` | stdout/stderr ถูกผสม | แยก diagnostics ออกจาก HDL |
| Top module ไม่ตรง | Export entity ผิดตัว | ตรวจ export command และ module declarations |
| Port widths ไม่ตรง | Wrapper หรือ slicing เปลี่ยน | เทียบ JSON ports กับ specification |
| VHDL ผ่านแต่ Verilog UART timeout | Conversion, configuration หรือ init semantics | ตรวจ reset, ROM fetch และ UART configuration |
| Verilog มี unresolved modules | External IP/blackbox replacement ไม่ครบ | เพิ่ม implementation ที่ถูกต้องและ audit views |
| ROM smoke ผ่านแต่ ASIC boot plan ไม่ชัด | อาศัย simulation initialization | ตรวจ ROM/RAM realizability และ loading sequence |
| เปลี่ยน testbench clock แล้ว baud ผิด | Hardware clock metadata ยังเป็นค่าเดิม | เปลี่ยน wrapper และ export ใหม่ |
| Generated Verilog ถูกแก้หลายจุด | ไม่มี reproducible generation path | แก้ต้นทางและสร้าง output ใหม่พร้อม provenance |

### 3.21 ผลส่งและเกณฑ์ผ่าน

ผู้เรียนต้องส่ง:

1. VHDL wrapper และ configuration specification
2. Generated Verilog
3. Import/elaboration/export logs
4. Parser และ Yosys structural-check logs
5. Port-check result
6. Memory/initialization audit
7. VHDL และ Verilog ROM smoke results
8. Negative-test result
9. Export manifest และ hashes

Lab ผ่านเมื่อทุก gate ต่อไปนี้ผ่านหรือมี disposition ที่ระบุชัดเจน:

| Gate | Acceptance |
|---|---|
| Export | Exit code สำเร็จและ output ไม่ว่าง |
| Parsing | Downstream parser อ่านได้ |
| Interface | Ports, widths และ bit ordering ตรง specification |
| Structure | ไม่มี unresolved hierarchy หรือ structural errors |
| Memory | ทราบ implementation และ initialization assumptions |
| Behavior | Verilog ROM smoke ผ่านเทียบกับ VHDL configuration เดียวกัน |
| Checker | Negative test ล้มเหลวตามที่คาด |
| Provenance | สร้าง output เดิมซ้ำได้จาก inputs ที่บันทึกไว้ |

### 3.22 คำถามท้าย Lab

1. เหตุใด export สำเร็จจึงยังไม่เท่ากับ design ถูกต้อง?
2. VHDL generics ที่ถูก resolve แล้วต่างจาก Verilog parameters อย่างไร?
3. เพราะเหตุใดต้องใช้ wrapper เดียวกันใน VHDL smoke และ export?
4. การรวม stderr ลงใน generated HDL มีผลอย่างไร?
5. Constant ROM initialization ต่างจาก RAM power-up initialization อย่างไร?
6. ทำไมไม่ควรลบ `initial` blocks ทั้งหมดโดยอัตโนมัติ?
7. Port check และ structural check พิสูจน์อะไร และยังไม่พิสูจน์อะไร?
8. ROM smoke ที่ตรงกันถือเป็น formal equivalence หรือไม่?
9. หากเปลี่ยน memory size ต้องสร้างและตรวจ artifacts ใดใหม่?
10. ต้องเก็บข้อมูลใดจึงตรวจสอบที่มาของ generated Verilog ได้ครบ?