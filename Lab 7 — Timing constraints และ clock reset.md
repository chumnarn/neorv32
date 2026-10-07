## Lab 7 — Timing constraints และ clock/reset

Lab นี้ต่อจาก Lab 5 ซึ่งได้ technology-mapped netlist และ Lab 6 ซึ่งกำหนด pad ring และ pin mapping แล้ว ผู้เรียนจะสร้าง timing constraints สำหรับ NEORV32 full-chip บน IHP SG13G2 ตรวจว่า clock ผ่าน input pad ไปถึง sequential cells และตรวจ reset โดยแยกประเด็น functional behavior, synchronization และ recovery/removal ออกจากกัน

**เป้าหมายสำคัญ:** ทุก timing path ต้องได้รับการวิเคราะห์ หรือมีข้อยกเว้นพร้อมหลักฐานอธิบาย การที่รายงานไม่มี timing violation ยังไม่เพียงพอ หาก clock หรือ paths บางส่วนไม่ได้ถูกวิเคราะห์

ตัวอย่างชื่อไฟล์ พอร์ต และ hierarchy ใน Lab นี้เป็นรูปแบบสำหรับจัดทำคู่มือ ต้องปรับให้ตรงกับ wrapper และ mapped netlist ที่ได้จาก Lab 5–6 ก่อนใช้งาน ส่วนค่าตัวเลขเป็น **สมมติฐานสำหรับการฝึก** ไม่ใช่ข้อกำหนดของ NEORV32 หรือ PDK

### 7.1 วัตถุประสงค์

เมื่อจบ Lab ผู้เรียนต้องสามารถ:

1. ระบุ clock source, clock sinks และ clock domain จาก RTL และ mapped netlist
2. กำหนด clock, uncertainty, input/output delay, input transition และ output load
3. แยก synchronous interface ออกจาก asynchronous input เช่น UART RX
4. ตรวจโครงสร้าง reset และขอบเขตของ timing exceptions
5. เชื่อม SDC เข้ากับ LibreLane อย่างชัดเจน
6. อ่าน setup/hold reports และตรวจ constraint coverage
7. จัดทำ constraint release สำหรับนำไปใช้ใน floorplan, placement และ CTS

### 7.2 อินพุตและผลส่งของ Lab

| รายการ | รายละเอียด |
|---|---|
| RTL | NEORV32 พร้อม ASIC wrapper ที่ใช้จริง |
| Netlist | Technology-mapped netlist จาก Lab 5 |
| Pad mapping | ตาราง package pin → top-level port → pad → core signal จาก Lab 6 |
| Timing models | Standard-cell, IO-cell และ memory Liberty ตาม corner ที่ใช้ |
| Flow configuration | LibreLane configuration ของโครงการ |
| ผลส่ง | SDC, clock/reset inventory, exception register และ STA reports |

แนะนำให้จัดโครงสร้างดังนี้:

```text
constraints/
├── functional_common.sdc
├── pnr.sdc
└── signoff.sdc

docs/
├── timing_interface.csv
├── clock_reset_inventory.md
└── timing_exceptions.csv

reports/lab7/
├── environment.txt
├── clock_report.txt
├── constraint_checks.txt
├── setup_paths.txt
├── hold_paths.txt
└── coverage_review.md
```

---

### 7.3 ขั้นที่ 1 — บันทึก environment และ design revision

ก่อนเขียน constraints ต้องทราบว่าใช้ design และ tools รุ่นใด เพราะชื่อ register หลัง synthesis และรูปแบบรายงานอาจเปลี่ยนระหว่างรุ่น

รันจาก project root:

```bash
mkdir -p reports/lab7

{
  date -Iseconds
  git rev-parse HEAD
  git status --short
  librelane --version
  yosys -V
  openroad -version
} > reports/lab7/environment.txt 2>&1
```

หาก NEORV32 เป็น Git submodule ให้บันทึก revision เพิ่ม:

```bash
git submodule status \
  > reports/lab7/submodules.txt
```

บันทึกเพิ่มเติม:

- PDK revision หรือ identifier ของ installation
- LibreLane configuration ที่ใช้
- ชื่อ top module หลัง export VHDL
- netlist ที่นำมาวิเคราะห์
- Liberty files และ corners ที่โหลด
- สถานะ parasitics: ไม่มี, estimated หรือ extracted

**จุดตรวจ:** การอ่าน STA report ต้องรู้ว่ารายงานนั้นใช้ netlist, corner และ parasitic model ใด

---

### 7.4 ขั้นที่ 2 — สร้าง timing interface inventory

เริ่มจากพอร์ตของ **full-chip top** ไม่ใช่พอร์ตของ NEORV32 core เพียงอย่างเดียว

ตัวอย่าง mapping:

| Top-level port | Pad instance | Core signal | ประเภท |
|---|---|---|---|
| `clk_pad_i` | `u_pad_clk` | `clk_core` | Primary clock |
| `rstn_pad_i` | `u_pad_rstn` | `rstn_core` | External asynchronous reset |
| `uart_rx_pad_i` | `u_pad_uart_rx` | `uart0_rxd_i` | Asynchronous data input |
| `uart_tx_pad_o` | `u_pad_uart_tx` | `uart0_txd_o` | Serial output |
| `gpio_pad_o[7:0]` | GPIO output pads | `gpio_o[7:0]` | Output ตาม interface contract |

ค้นหาจุดที่เกี่ยวข้องใน source:

```bash
rg -n \
  'clk_i|rstn_i|rising_edge|falling_edge|uart.*rxd|jtag.*tck' \
  rtl src
```

ปรับ directory ให้ตรงกับโครงการ

สำหรับทุกพอร์ต ให้บันทึก:

```text
port_name
direction
functional_role
reference_clock
synchronous_or_asynchronous
external_delay_min
external_delay_max
input_transition
output_load
exception_reason
```

**ข้อควรแยกให้ชัด:** GPIO อาจเป็น synchronous interface หรือ asynchronous external input ขึ้นกับการใช้งานจริง ชื่อ GPIO ไม่ได้บอกว่าจะต้องใช้ timing constraint แบบใด

---

### 7.5 ขั้นที่ 3 — ตรวจ clock architecture

#### 7.5.1 ไล่เส้นทาง clock

ตรวจเส้นทางจริง:

```text
Package clock pin
→ top-level input port
→ input pad
→ internal clock net
→ clock distribution
→ sequential-cell clock pins
```

ตรวจใน mapped netlist:

```bash
rg -n \
  'clk_pad_i|clk_core|u_pad_clk|sg13g2_IOPad' \
  build
```

ใช้ netlist inspection เพิ่มเติมเพื่อยืนยันว่า:

- Clock pad ต่อสัญญาณเข้าถูก pin
- Pad configuration ทำให้ input receiver ทำงาน
- ไม่มี inverter หรือ mux ที่ถูกมองข้าม
- ไม่มี register ที่รับ clock จาก net ที่ไม่ตั้งใจ
- ทุก enabled subsystem มี clock definition รองรับ

#### 7.5.2 แยก clock ออกจาก clock enable

ตัวนับ prescaler หรือสัญญาณ tick ไม่จำเป็นต้องเป็น clock ใหม่

```vhdl
if rising_edge(clk_i) then
  if tick = '1' then
    counter <= counter + 1;
  end if;
end if;
```

ตัวอย่างนี้ register ยังอยู่ใน clock domain ของ `clk_i`

แต่หากมี:

```vhdl
if rising_edge(divided_clk) then
  ...
end if;
```

ต้องตรวจว่า `divided_clk` เป็น generated clock และมี constraints ครบ

ห้ามสร้าง generated clock จากชื่อสัญญาณเพียงอย่างเดียว ต้องตรวจว่ามันขับ clock pins จริงหรือไม่

#### 7.5.3 Clock gating และ JTAG

หาก design revision ที่ใช้มี clock gating ให้ตรวจ:

- Gated clock ไปถึง register ใด
- Timing engine propagate master clock ผ่าน gating cell ได้หรือไม่
- Library มี clock-gating timing checks หรือไม่
- CTS รองรับโครงสร้างนั้นอย่างไร

หากเปิด JTAG/debug และมี `jtag_tck_i` ต้องเพิ่ม clock domain และตรวจ CDC ตามโครงสร้างจริง เอกสาร LibreLane ระบุว่า flow ตั้งต้นสมมติ single clock domain ดังนั้น multi-clock design ต้องปรับทั้ง constraints และ flow ที่เกี่ยวข้อง ไม่ใช่เพิ่ม `create_clock` เพียงคำสั่งเดียว [LibreLane Documentation](https://librelane.readthedocs.io/en/stable/usage/timing_closure/index.html?utm_source=chatgpt.com)

**แนวทางสำหรับ Lab พื้นฐาน:** เริ่มจาก functional configuration ที่มี primary clock เดียว แล้วขยาย debug mode เป็นแบบฝึกขั้นสูง

---

### 7.6 ขั้นที่ 4 — กำหนด timing contract

ใช้ตัวอย่างเป้าหมาย 100 MHz:

$$
T=\frac{1}{100\,\mathrm{MHz}}=10\,\mathrm{ns}
$$

| Parameter | ค่าฝึก | ความหมาย |
|---|---:|---|
| Clock period | 10.00 ns | Primary clock 100 MHz |
| Setup uncertainty | 0.25 ns | Budget สำหรับ setup |
| Hold uncertainty | 0.10 ns | Budget สำหรับ hold |
| Input delay max | 2.00 ns | External data arrival ช้าที่สุด |
| Input delay min | 0.20 ns | External data arrival เร็วที่สุด |
| Output delay max | 4.00 ns | External setup budget |
| Output delay min | −0.20 ns | External hold requirement 0.20 ns ในแบบจำลองอย่างง่าย |
| Data input transition | 0.15 ns | Slew ที่ chip input |
| Output load | 0.033442 pF | ตัวอย่างเมื่อ Liberty ใช้หน่วย pF |

OpenSTA ใช้หน่วยจาก Liberty file แรกที่อ่าน จึงต้องตรวจ time และ capacitance units ก่อนตีความตัวเลขใน SDC [GitHub](https://github.com/The-OpenROAD-Project/OpenSTA/blob/master/doc/Examples.md?utm_source=chatgpt.com)

#### 7.6.1 Input delay มาจากไหน

สำหรับ synchronous input แบบง่าย:

$$D_{\mathrm{in,max}}
=
t_{\mathrm{CQ,max,external}}
+
t_{\mathrm{board,max}}$$

$$D_{\mathrm{in,min}}
=
t_{\mathrm{CQ,min,external}}
+
t_{\mathrm{board,min}}$$

หาก external clock และ chip clock มี board skew ต้องรวม relative clock arrival ไว้ด้วย

#### 7.6.2 Output delay มาจากไหน

สำหรับ output ไปยัง external receiver:

$$D_{\mathrm{out,max}}
=
t_{\mathrm{setup,external}}
+
t_{\mathrm{board,max}}$$

$$D_{\mathrm{out,min}}
=
t_{\mathrm{board,min}}
-
t_{\mathrm{hold,external}}$$

สูตรนี้สมมติว่า clock reference ไม่มี relative skew เพิ่มเติม ค่า `-min` จึงอาจเป็นลบ และไม่ควรถูกตั้งเป็นศูนย์โดยอัตโนมัติ

#### 7.6.3 ตำแหน่งอ้างอิงของ delay

หาก netlist มี IO pads อยู่แล้ว และกำหนด delays ที่ package-side top-level ports:

- Input pad delay เป็นส่วนหนึ่งของ internal timing path
- Output pad delay เป็นส่วนหนึ่งของ internal timing path
- External delay ต้องไม่บวก pad delay ซ้ำ

การวิเคราะห์ core-only ต้องเปลี่ยน interface contract ให้สัมพันธ์กับ boundary ใหม่

---

### 7.7 ขั้นที่ 5 — สร้าง primary clock ที่ full-chip boundary

ตัวอย่าง:

```tcl
create_clock \
    -name core_clk \
    -period 10.000 \
    -waveform {0.000 5.000} \
    [get_ports clk_pad_i]
```

กำหนด uncertainty:

```tcl
set_clock_uncertainty \
    -setup 0.250 \
    [get_clocks core_clk]

set_clock_uncertainty \
    -hold 0.100 \
    [get_clocks core_clk]
```

กำหนด slew ของ clock สำหรับ ideal-clock analysis:

```tcl
set_clock_transition \
    0.150 \
    [get_clocks core_clk]
```

ตัวเลข slew เป็นสมมติฐานฝึก ต้องปรับจาก clock source specification ในงานจริง

**หลีกเลี่ยงการสร้าง primary clock ซ้ำที่ output pin ของ clock pad** เพราะอาจทำให้ upstream pad path ถูกตัดออกจากการวิเคราะห์ หรือเกิด clock definitions ที่ซ้อนกันโดยไม่ตั้งใจ

ตรวจ path report ว่า clock source เป็น `clk_pad_i` และ sequential cells ที่ต้องการได้รับ `core_clk`

---

### 7.8 ขั้นที่ 6 — กำหนด synchronous IO constraints

ใช้เฉพาะพอร์ตที่ interface contract ระบุว่าสัมพันธ์กับ `core_clk`

ตัวอย่างสมมติว่า design มี synchronous input:

```tcl
set sync_inputs [get_ports {
    data_pad_i[0]
    data_pad_i[1]
}]
```

ใส่ทั้ง min และ max:

```tcl
set_input_delay \
    -clock core_clk \
    -max 2.000 \
    $sync_inputs

set_input_delay \
    -clock core_clk \
    -min 0.200 \
    $sync_inputs

set_input_transition \
    0.150 \
    $sync_inputs
```

ตัวอย่าง synchronous output:

```tcl
set sync_outputs [get_ports {
    gpio_pad_o[0]
    gpio_pad_o[1]
}]
```

```tcl
set_output_delay \
    -clock core_clk \
    -max 4.000 \
    $sync_outputs

set_output_delay \
    -clock core_clk \
    -min -0.200 \
    $sync_outputs

set_load \
    0.033442 \
    $sync_outputs
```

**สำคัญ:** หาก GPIO outputs ไม่มี synchronous receiver จริง ค่าข้างต้นเป็นเพียง training contract ต้องบันทึกให้ชัด และเปลี่ยนเป็น requirement ที่เหมาะสมก่อนนำไปใช้งานจริง

ห้ามใช้ `all_inputs` ตั้ง input delay โดยไม่กรอง clock, reset และ asynchronous inputs

---

### 7.9 ขั้นที่ 7 — ตรวจและ constrain UART RX

UART RX ไม่ได้มี phase relation คงที่กับ `core_clk` จึงไม่ควรกำหนด input delay เสมือนเป็น synchronous data โดยไม่มีเหตุผลรองรับ

#### 7.9.1 ตรวจ receiver structure

ค้นหา:

```bash
rg -n \
  'uart|rxd|sync|synchron' \
  rtl src
```

จากนั้นตรวจใน mapped netlist ว่าเส้นทางมีลักษณะ:

```text
UART RX port
→ input pad
→ first synchronization register
→ subsequent synchronization register
→ receiver logic
```

ชื่อจริงและจำนวน stages ต้องยืนยันจาก revision ที่ใช้

#### 7.9.2 กำหนด exception เฉพาะ first-stage endpoint

ตัวอย่างรูปแบบ:

```tcl
set uart_rx_port [get_ports uart_rx_pad_i]

set uart_rx_first_d [get_pins {
    u_soc/u_uart0/u_rx_sync_ff0/D
}]

if {[llength $uart_rx_first_d] != 1} {
    error "UART RX first-stage D pin must resolve to exactly one pin"
}

set_input_transition 0.150 $uart_rx_port

set_false_path \
    -from $uart_rx_port \
    -to $uart_rx_first_d
```

ชื่อ pin ในตัวอย่างต้องแทนด้วยชื่อที่พบใน mapped netlist และ Liberty จริง

ต้องรักษาการวิเคราะห์:

```text
First-stage Q → second-stage D
Second-stage Q → receiver logic
```

**ห้ามใช้ wildcard ครอบคลุม synchronizer ทั้งชุดโดยไม่ตรวจ endpoint** และอย่าสรุปว่า CDC ผ่านเพราะ STA ไม่มี violation ต้องตรวจ synchronization structure และ metastability assumptions แยกต่างหาก

หาก synthesis เปลี่ยนชื่อจน query ไม่พบ object ต้องให้ script หยุด ไม่ควรปล่อย exception หายไปเงียบ ๆ

---

### 7.10 ขั้นที่ 8 — ตรวจ reset architecture

NEORV32 ระบุ external reset `rstn_i` เป็น active-low asynchronous reset ในตัวอย่าง integration และเอกสารอธิบายว่า control-path registers ใช้ asynchronous reset อย่างไรก็ตามต้องตรวจวงจร reset ภายในจาก source revision ของโครงการก่อนกำหนดข้อยกเว้น [GitHub](https://github.com/stnolting/neorv32/blob/main/rtl/test_setups/neorv32_test_setup_on_chip_debugger.vhd?utm_source=chatgpt.com)

#### 7.10.1 ไล่เส้นทาง reset

ตรวจ:

```text
External reset port
→ reset input pad
→ reset conditioning / synchronization
→ internal reset domains
→ sequential-cell reset pins
```

ค้นหา:

```bash
rg -n \
  'rstn|reset|watchdog|wdt|debug' \
  rtl src
```

ตอบคำถามให้ครบ:

1. External reset assert แบบ asynchronous หรือไม่
2. Deassert ถูก synchronize ที่ใด
3. Synchronizer ใช้ clock ใด
4. มี watchdog หรือ debug reset รวมเข้ามาหรือไม่
5. Clock ต้องทำงานระหว่าง reset release หรือไม่
6. Reset pulse width ขั้นต่ำเท่าใด
7. หาก clock ถูก gate อยู่ reset จะถูก release อย่างไร

#### 7.10.2 แยก assertion และ deassertion

**Asynchronous assertion:** สามารถทำให้ reset active โดยไม่ต้องรอ clock edge แต่ต้องตรวจ pulse width, reset-tree electrical behavior และข้อกำหนดของ cells

**Synchronized deassertion:** ทำให้การปล่อย resetสัมพันธ์กับ clock domain ที่รับ reset เพื่อลดความเสี่ยงของการ release ใกล้ clock edge

สำหรับ asynchronous-reset cells ต้องพิจารณา:

- **Recovery:** เวลาที่ reset ต้อง inactive ก่อน active clock edge
- **Removal:** เวลาที่ reset ต้องคง active หลัง active clock edge ก่อนปล่อย

#### 7.10.3 หลีกเลี่ยง blanket reset exception

คำสั่งนี้มีขอบเขตกว้าง:

```tcl
set_false_path -from [get_ports rstn_pad_i]
```

หากใช้โดยไม่ตรวจวงจร อาจซ่อนการตรวจ recovery/removal ของ direct reset paths

แนวทางที่เหมาะสมกว่า:

- ระบุว่า raw asynchronous reset ไปถึง pins ใด
- กำหนด exception เฉพาะขอบเขตที่ methodology อนุญาต
- เก็บ downstream synchronized-reset checks ที่วิเคราะห์ได้
- บันทึก checks ที่ถูกตัดออกและวิธีตรวจทดแทน
- ตรวจทั้ง timing models และ reset functional behavior

หากยังไม่ยืนยัน reset architecture ให้ระบุ **RESET REVIEW OPEN** และไม่ใช้ Lab นี้เป็น sign-off release

#### 7.10.4 Functional case analysis

การกำหนด:

```tcl
set_case_analysis 1 [get_ports rstn_pad_i]
```

อาจใช้ใน functional-mode scenario เพื่อกำหนด reset inactive แต่ scenario นี้ไม่ได้พิสูจน์ความปลอดภัยของ reset release

หากใช้ ต้องมี reset-release verification แยกต่างหาก และบันทึกว่า recovery/removal checks ใดถูก disable ใน functional scenario

---

### 7.11 ขั้นที่ 9 — จัดทำ SDC ที่ตรวจชื่อ object ก่อนใช้งาน

ตัวอย่างต่อไปนี้เป็น baseline สำหรับ clock และ GPIO outputs โดย **ยังต้องเพิ่ม asynchronous-input และ reset policy จากขั้นก่อนหน้า**

ไฟล์ `constraints/functional_common.sdc`:

```tcl
# Training assumptions:
# time unit: ns
# capacitance unit: pF
# Replace ports and budgets with the actual interface contract.

proc require_ports {patterns expected_count label} {
    set objects [get_ports $patterns]

    if {[llength $objects] != $expected_count} {
        error "$label: expected $expected_count ports, got [llength $objects]"
    }

    return $objects
}

set clk_port [require_ports \
    {clk_pad_i} 1 "Primary clock"]

set rst_port [require_ports \
    {rstn_pad_i} 1 "External reset"]

set gpio_outputs [require_ports {
    gpio_pad_o[0]
    gpio_pad_o[1]
    gpio_pad_o[2]
    gpio_pad_o[3]
    gpio_pad_o[4]
    gpio_pad_o[5]
    gpio_pad_o[6]
    gpio_pad_o[7]
} 8 "GPIO output bus"]

create_clock \
    -name core_clk \
    -period 10.000 \
    -waveform {0.000 5.000} \
    $clk_port

set_clock_uncertainty \
    -setup 0.250 \
    [get_clocks core_clk]

set_clock_uncertainty \
    -hold 0.100 \
    [get_clocks core_clk]

set_output_delay \
    -clock core_clk \
    -max 4.000 \
    $gpio_outputs

set_output_delay \
    -clock core_clk \
    -min -0.200 \
    $gpio_outputs

set_load 0.033442 $gpio_outputs

# Add the reviewed UART RX exception here.
# Add the reviewed UART TX requirement here.
# Add the reviewed reset policy here.
# Do not release this file while these items remain unresolved.
```

ไฟล์ `constraints/pnr.sdc`:

```tcl
set constraint_dir [file dirname [info script]]

source [file join \
    $constraint_dir functional_common.sdc]

set_clock_transition \
    0.150 \
    [get_clocks core_clk]
```

ไฟล์ `constraints/signoff.sdc`:

```tcl
set constraint_dir [file dirname [info script]]

source [file join \
    $constraint_dir functional_common.sdc]

set_propagated_clock [get_clocks core_clk]
```

`signoff.sdc` ข้างต้นเหมาะกับ post-CTS/post-route analysis ที่มี clock network จริง ต้องตรวจด้วยว่า LibreLane step ที่ใช้จัดการ propagated clocks อย่างไร

อย่าใส่ `set_propagated_clock` ลงใน common file ที่ใช้กับ pre-CTS โดยไม่มีเหตุผล

---

### 7.12 ขั้นที่ 10 — เชื่อม SDC เข้ากับ LibreLane

เพิ่มใน configuration โดยสมมติว่า config อยู่ที่ project root:

```json
{
  "CLOCK_PORT": "clk_pad_i",
  "CLOCK_PERIOD": 10.0,
  "PNR_SDC_FILE": "dir::constraints/pnr.sdc",
  "SIGNOFF_SDC_FILE": "dir::constraints/signoff.sdc"
}
```

หาก config อยู่ directory อื่น ต้องปรับ relative paths ตามตำแหน่ง config

LibreLane มีตัวแปร `PNR_SDC_FILE` สำหรับ implementation และ `SIGNOFF_SDC_FILE` สำหรับ sign-off STA พร้อม step ตรวจว่าระบุไฟล์อย่างชัดเจนหรือไม่ [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/reference/step_config_vars.html?utm_source=chatgpt.com)

ตรวจความสอดคล้อง:

| รายการ | ค่าที่ควรตรงกัน |
|---|---|
| `CLOCK_PORT` | Clock source port ใน SDC |
| `CLOCK_PERIOD` | Period ของ primary clock |
| NEORV32 `CLOCK_FREQUENCY` | ความถี่ใช้งานที่ software/peripherals ต้องรับรู้ |
| Testbench clock | ความถี่และ waveform ตาม scenario |
| Firmware configuration | สัมพันธ์กับความถี่ hardware ที่ใช้งานจริง |

**ประเด็นสำคัญ:** เปลี่ยน SDC เป็น 100 MHz ไม่ได้เปลี่ยน generic ที่ export เป็น hardware แล้ว และไม่ได้เปลี่ยน firmware configuration โดยอัตโนมัติ

---

### 7.13 ขั้นที่ 11 — รัน pre-PnR STA

ตรวจ syntax ของ CLI รุ่นที่ติดตั้ง:

```bash
librelane --help
```

หากรุ่นที่ใช้รองรับการหยุดที่ step:

```bash
librelane \
  --run-tag lab7_constraints \
  --to OpenROAD.STAPrePNR \
  config.json
```

ใช้ PDK arguments และ environment แบบเดียวกับ Lab 5

หาก synthesis ใช้ทรัพยากรมาก สามารถวิเคราะห์ mapped netlist เดิมใน STA session ที่โหลด timing libraries ครบได้ โดยต้องแน่ใจว่าไม่ได้ reuse ผล constraints เก่า

ค้นหา artifacts:

```bash
rg --files runs/lab7_constraints |
  rg -i 'sta|sdc|timing|check|max\.rpt|min\.rpt'
```

ค้นหาข้อผิดพลาด:

```bash
rg -n -i \
  'error|warning|no clock|unconstrained|not found|empty|sdc' \
  runs/lab7_constraints
```

**จุดตรวจ:**

- SDC ถูกอ่านจาก path ที่ตั้งใจ
- ไม่มี empty collection
- Clock period ถูกต้อง
- IO และ memory timing models ถูกโหลดครบ
- ไม่มี unexplained black-box timing boundary
- ทุก sequential clock domain ที่ใช้งานมี clock รองรับ

Pre-PnR มีประโยชน์ต่อการตรวจ constraints และ setup feasibility แต่ยังไม่แทน post-CTS/post-route timing closure

---

### 7.14 ขั้นที่ 12 — สร้างรายงานใน OpenSTA/OpenROAD

คำสั่งต่อไปนี้ใช้หลังโหลด design, libraries และ SDC แล้ว:

```tcl
report_units
report_clock_properties [get_clocks *]

check_setup -verbose

report_checks \
    -path_delay max \
    -format full_clock_expanded \
    -group_path_count 10

report_checks \
    -path_delay min \
    -format full_clock_expanded \
    -group_path_count 10

report_check_types -all_violators
```

ตรวจ options กับ tool รุ่นที่ใช้:

```tcl
help check_setup
help report_checks
help report_check_types
```

OpenSTA มี `check_setup` สำหรับตรวจความครบถ้วนเบื้องต้น เช่น inputs ที่ไม่มี input delay และมี `report_checks` สำหรับรายงาน max/min timing paths [GitHub](https://github.com/The-OpenROAD-Project/OpenSTA/blob/master/search/Search.tcl?utm_source=chatgpt.com)

สำหรับ asynchronous inputs ที่ไม่มี input delay ต้องจับคู่ warning กับ approved exception ไม่ใช่เพิ่ม delay สมมติเพียงเพื่อให้ warning หาย

#### 7.14.1 อ่าน setup report

ตรวจ:

- Startpoint และ endpoint
- Launch/capture clock
- Clock period
- External input/output delay
- Cell และ net delay
- Uncertainty
- Library setup time
- Slack

สำหรับ reg-to-reg path แบบง่าย:

$$Slack_{\mathrm{setup}}
=
T+L_{\mathrm{capture}}-L_{\mathrm{launch}}
-t_{\mathrm{CQ,max}}
-t_{\mathrm{data,max}}
-t_{\mathrm{setup}}
-U_{\mathrm{setup}}$$

ตัวอย่าง ideal clock:

```text
Period                    = 10.00 ns
Clock-to-Q max            =  0.20 ns
Data path max             =  6.80 ns
Setup time                =  0.15 ns
Setup uncertainty         =  0.25 ns
```

$$Slack_{\mathrm{setup}}
=10.00-0.20-6.80-0.15-0.25
=2.60\,\mathrm{ns}$$

#### 7.14.2 อ่าน hold report

สำหรับ reg-to-reg path แบบง่าย:

$$Slack_{\mathrm{hold}}
=
L_{\mathrm{launch}}
+t_{\mathrm{CQ,min}}
+t_{\mathrm{data,min}}
-L_{\mathrm{capture}}
-t_{\mathrm{hold}}
-U_{\mathrm{hold}}$$

ตัวอย่างหลัง CTS:

```text
Launch clock latency      = 0.30 ns
Capture clock latency     = 0.50 ns
Clock-to-Q min            = 0.08 ns
Data path min             = 0.10 ns
Hold time                 = 0.05 ns
Hold uncertainty          = 0.10 ns
```

$$Slack_{\mathrm{hold}}
=0.30+0.08+0.10-0.50-0.05-0.10
=-0.17\,\mathrm{ns}$$

กรณีนี้มี hold violation ซึ่งการลดความถี่ไม่ได้แก้โดยตรง

---

### 7.15 ขั้นที่ 13 — ตรวจ timing coverage

สร้างตาราง review:

| Path class | วิธีวิเคราะห์ | หลักฐาน |
|---|---|---|
| Register → register | Setup/hold | Max/min path reports |
| Synchronous input → register | Input delay min/max | Interface contract และ STA |
| Register → synchronous output | Output delay min/max | Interface contract และ STA |
| Input → output | Requirement ตาม interface | Path report หรือ absence ที่ตรวจแล้ว |
| UART RX → first synchronizer | Scoped asynchronous exception | RTL/netlist tracing |
| Synchronizer stage 1 → stage 2 | Setup/hold ยัง active | Path report |
| External reset release | Reset methodology | Reset architecture และ check review |
| Internal reset distribution | Recovery/removal ที่เกี่ยวข้อง | Timing checks และ Liberty |
| Clock source → clock sinks | Clock propagation | Clock/path reports |
| Gating control | Clock-gating checks ถ้ามี | Library และ STA reports |

ทุก object ต้องอยู่ในสถานะใดสถานะหนึ่ง:

- **TIMED**
- **EXCEPTED WITH EVIDENCE**
- **NOT APPLICABLE**
- **OPEN**

การรายงาน unconstrained เป็นศูนย์ไม่ได้พิสูจน์ว่า exceptions ถูกต้อง เพราะ paths ที่ถูก false-path อาจหายจากการวิเคราะห์แล้ว

---

### 7.16 ขั้นที่ 14 — จัดทำ exception register

ตัวอย่าง `docs/timing_exceptions.csv`:

```csv
id,type,from,to,reason,evidence,status
EX001,false_path,uart_rx_pad_i,exact_first_sync_D,asynchronous_UART_input,rtl_and_netlist_review,review_required
EX002,reset_policy,rstn_pad_i,explicit_reset_endpoints,reviewed_reset_architecture,reset_review,open
```

ทุก exception ต้องตอบได้ว่า:

1. ทำไม path นี้จึงไม่ใช้ synchronous timing relationship ตามปกติ
2. Endpoint ที่ครอบคลุมมีจำนวนเท่าใด
3. มี paths ที่ยังต้อง timing ต่อจาก boundary นั้นหรือไม่
4. หลักฐานอยู่ที่ RTL/netlist ใด
5. ถ้า synthesis เปลี่ยนชื่อ script จะตรวจพบหรือไม่

ห้ามใช้ multicycle path เพราะ CPU instruction ใช้หลาย cycles หรือเพราะ UART baud rate ต่ำกว่า core clock ต้องพิสูจน์พฤติกรรม launch/capture ของ register คู่ที่เกี่ยวข้องก่อน

---

### 7.17 ขั้นที่ 15 — ทดสอบ clock/reset behavior

ใช้ testbench จาก Lab ก่อนหน้าเพิ่ม scenarios:

| Scenario | สิ่งที่ต้องตรวจ |
|---|---|
| Power-on reset | Design เข้า reset state ตาม specification |
| Reset assert ขณะ CPU ทำงาน | ไม่มี side effects ที่ผิดข้อกำหนด |
| Reset deassert ใกล้ clock edge | Release เป็นไปตาม synchronization architecture |
| Reset ขณะ clock หยุด | Assertion และ release behavior ถูกต้อง |
| Reset pulse หลายความกว้าง | เป็นไปตาม minimum pulse requirement |
| Watchdog reset ถ้าเปิดใช้ | Reset sequence และ boot ใหม่ถูกต้อง |
| Debug reset ถ้าเปิดใช้ | ขอบเขต reset ถูกต้อง |

RTL simulation ตรวจ functional sequencing ได้ แต่ไม่จำลอง analog metastability จึงต้องใช้ร่วมกับ structural CDC/RDC review และ timing checks

การปรับ reset synchronizer เป็นการเปลี่ยน RTL ต้องย้อนตรวจ synthesis และ functional verification ไม่ควรแก้โดยเพิ่ม SDC exception เพียงอย่างเดียว

---

### 7.18 ขั้นที่ 16 — ทดลองเปลี่ยน constraint เพื่อเรียนรู้ผล

ทำสำเนา scenario ก่อนทดลอง และเปลี่ยนครั้งละหนึ่ง parameter:

| การทดลอง | สิ่งที่ควรสังเกต |
|---|---|
| Period 10 → 8 ns | Setup budget ลดลงประมาณ 2 ns สำหรับ one-cycle paths |
| Setup uncertainty 0.25 → 0.50 ns | Setup slack ลดลงประมาณ 0.25 ns |
| Output delay max 4 → 5 ns | Reg-to-output setup budget ลดลงประมาณ 1 ns |
| Input delay max 2 → 3 ns | Input-to-reg setup budget ลดลงประมาณ 1 ns |
| เพิ่ม output load | Delay/slew เปลี่ยนตาม timing model |
| ลบ clock definition | Coverage checks ต้องตรวจพบปัญหา |
| ทำชื่อ first-stage pin ให้ผิด | SDC ต้องหยุดด้วย error |

สำหรับการเปรียบเทียบผลของ constraints ให้ใช้ netlist และ parasitics เดิม หาก rerun optimization ด้วย constraints ใหม่ netlist อาจเปลี่ยน ทำให้ slack ไม่ได้เปลี่ยนตาม budget อย่างเดียว

---

### 7.19 ปัญหาที่พบบ่อย

| อาการ | สาเหตุที่ควรตรวจ | วิธีดำเนินการ |
|---|---|---|
| ไม่มี timing paths | Clock หาย, design ไม่ link, exceptions กว้าง | ตรวจ clocks, libraries และ endpoint queries |
| Clock ไม่ถึง core | Pad arc/model หรือ wiring ผิด | ตรวจ pad configuration และ clock tracing |
| SDC หา port ไม่พบ | ใช้ core port กับ full-chip netlist | เปลี่ยนเป็น top-level boundary จริง |
| UART RX มี setup violation | ถูก constrain เป็น synchronous input | ตรวจ synchronizer แล้วใช้ scoped exception |
| Reset ไม่มี recovery/removal report | False path, case analysis หรือ model ไม่มี checks | ตรวจ exceptions และ Liberty |
| Hold ผ่านก่อน CTS แต่ fail หลัง CTS | Clock skew และ physical delay เปลี่ยน | ตรวจ post-CTS min paths |
| Constraint warning หายแต่ paths หายด้วย | Blanket exception | Review exception coverage |
| Output delay เป็นศูนย์ทั้งหมด | External contract ไม่ครบ | กำหนด setup/hold requirement ของ receiver |
| Clock frequency ใน software ไม่ตรง SDC | Generic/firmware ยังใช้ค่าเก่า | ตรวจ configuration ตั้งแต่ export ถึง firmware |
| Full-chip STA ไม่มี pad delay | IO timing model หายหรือเป็น black box | โหลด IO Liberty ที่ถูกต้อง |

---

### 7.20 Acceptance criteria

แยกการผ่าน Lab เป็นสองระดับ

**ระดับ A — Constraints พร้อมใช้ต่อ**

- Primary clock source และ period ถูกต้อง
- Clock domain ที่เปิดใช้งานมี definitions ครบ
- Timing units ตรวจแล้ว
- ทุก functional IO มี timing requirement หรือ reviewed asynchronous policy
- Min/max delays ครบสำหรับ synchronous interfaces
- Clock ผ่าน pad ไปถึง intended sinks
- UART exceptions จำกัด endpoint และตรวจชื่อได้
- Reset architecture และ release methodology ตรวจแล้ว
- ไม่มี unexplained unconstrained paths หรือ missing timing models
- `PNR_SDC_FILE` และ `SIGNOFF_SDC_FILE` ถูกโหลดจริง
- Exception register ไม่มีรายการสำคัญสถานะ `OPEN`

**ระดับ B — Pre-PnR timing feasibility**

- Setup results ประเมินครบตาม corners ที่กำหนด
- Violations ที่เหลือมี path analysis และแผนแก้
- Hold results ถูกบันทึกพร้อมข้อจำกัดของ pre-CTS analysis
- รายงานไม่ถูกเรียกว่า final sign-off

หาก setup ยัง fail แต่ constraints ถูกต้อง ให้ระบุ **CONSTRAINTS READY / TIMING OPEN** แล้วนำผลไปวางแผน timing closure ใน Lab ถัดไป

### 7.21 แบบฟอร์มสรุปผลส่ง

```text
Lab 7 — Timing constraints and clock/reset

Design revision:
Top module:
LibreLane version:
PDK revision:
Netlist:
Liberty corners:
Parasitic model:

Primary clock:
Clock period:
Clock frequency:
Setup uncertainty:
Hold uncertainty:

Synchronous inputs:
Synchronous outputs:
Asynchronous inputs:
Clock domains:

Reset assertion:
Reset deassertion:
Reset-release verification:
Reviewed exceptions:

Clock coverage:
IO coverage:
Unconstrained paths:
Missing timing models:

Worst setup slack:
Worst hold slack:
Recovery/removal findings:

Constraint status:
Timing feasibility status:
Outstanding actions:
```

### 7.22 คำถามท้าย Lab

1. ทำไม full-chip clock ควรถูกนิยามที่ package-side input port?
2. การใส่ input delay โดยรวม pad delay ซ้ำทำให้ผล STA เปลี่ยนอย่างไร?
3. ทำไม `set_output_delay -min` จึงอาจเป็นค่าลบ?
4. Clock enable ต่างจาก generated clock อย่างไร?
5. ทำไม UART RX exception จึงควรจบที่ first-stage synchronizer?
6. Blanket reset false path ซ่อนการตรวจอะไรได้บ้าง?
7. Functional reset-inactive scenario ต่างจาก reset-release verification อย่างไร?
8. ทำไมลดความถี่แล้ว hold violation ยังอาจอยู่?
9. “ไม่มี violation” ต่างจาก “constraints ครบและถูกต้อง” อย่างไร?
10. หลักฐานใดทำให้อนุมัติ multicycle path ได้?

**ผลส่งหลักของ Lab:** ชุด SDC ที่สะท้อน interface contract จริง พร้อมหลักฐานว่า clock, reset, synchronous paths และ asynchronous boundaries ได้รับการตรวจครบก่อนเข้าสู่ physical implementation.