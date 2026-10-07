## Lab 12 — SRAM macro integration สำหรับต่อยอด

Lab นี้ต่อยอดจาก NEORV32 ที่ผ่านการทดสอบ UART/GPIO และ flow RTL-to-GDSII โดยเพิ่ม **SRAM hard macro ขนาด 4 KiB** เข้าในชิป และตรวจสอบทั้งพฤติกรรมการอ่านเขียน การเชื่อมต่อ bus การคงอยู่ของ macro หลัง synthesis ตลอดจน placement, PDN และ timing ที่ขอบเขต SRAM

แนวทางหลักของ Lab คือเพิ่ม SRAM ผ่าน **XBUS ภายในชิป** โดยยังคง boot ROM และหน่วยความจำเดิมไว้ก่อน วิธีนี้ช่วยแยกการตรวจสอบ SRAM ออกจากการเปลี่ยนระบบ boot และ linker ส่วนการแทนที่ internal DMEM/IMEM ให้เป็นงานต่อยอดหลัง integration ขั้นแรกผ่านแล้ว

> **ขอบเขตของตัวอย่าง:** ชื่อไฟล์ wrapper, memory map และ protocol ภายในที่แสดงเป็นข้อเสนอสำหรับ Lab นี้ ต้องปรับให้ตรงกับ source revision ของโครงการจริง โดยเฉพาะ pin SRAM, polarity ของ write mask, BIST tie-off และ XBUS handshake ห้ามใช้ค่าที่คาดเดาเพื่อสร้าง implementation

---

### 12.1 วัตถุประสงค์

เมื่อจบ Lab ผู้เรียนต้องสามารถ:

1. ตรวจสอบและเลือก SRAM macro ที่มี views ครบสำหรับ flow
2. แปลง byte address เป็น word address ได้ถูกต้อง
3. รองรับการเขียน byte, halfword และ word
4. ออกแบบ bus adapter ให้สอดคล้องกับ synchronous SRAM
5. แยก behavioral simulation model ออกจาก synthesis black box
6. ตรวจว่า synthesis คง SRAM เป็น hard macro
7. วาง macro และต่อ power pins ได้ครบ
8. วิเคราะห์ setup/hold รอบ SRAM ด้วย timing model จริง
9. ทดสอบ SRAM จากโปรแกรม C
10. จัดทำหลักฐาน integration และระบุข้อจำกัดของ sign-off

**ผลส่ง:** SRAM integration package พร้อมรายงาน functional verification, macro audit, PDN audit และ timing coverage

**Acceptance หลัก:** SRAM อ่านเขียนถูกต้อง ไม่มี address alias ที่ไม่ตั้งใจ ไม่มี bus transaction ซ้ำ macro อยู่ครบใน netlist และ layout และทุก required check มีผลตรวจจริง

---

### 12.2 สถาปัตยกรรมเป้าหมาย

ให้เริ่มจากสถาปัตยกรรมดังนี้:

```text
NEORV32 XBUS → address decoder / bus adapter → SRAM wrapper → SRAM hard macro
```

XBUS เป็น external bus interface ของ NEORV32 แต่ใน Lab นี้นำสัญญาณมาต่อกับ SRAM ที่อยู่ภายใน top-level ชิป จึงไม่ต้องเพิ่ม pad สำหรับ address/data ของ SRAM โดย NEORV32 ระบุว่า XBUS เป็น interface ที่เข้ากันได้กับ Wishbone และมีตัวเลือก register stage ซึ่งเพิ่ม latency อีกสอง cycle เมื่อเปิดใช้งาน [GitHub](https://github.com/stnolting/neorv32?utm_source=chatgpt.com)

กำหนด memory region สำหรับการทดลอง:

| รายการ | ค่าที่ใช้ใน Lab |
|---|---|
| SRAM ตัวอย่าง | `RM_IHPSG13_1P_1024x32_c2_bm_bist` |
| จำนวน word | 1,024 |
| ความกว้างข้อมูล | 32 bit |
| ความจุ | 4,096 byte หรือ 4 KiB |
| Base address ที่เสนอ | `0x9000_0000` |
| Last byte address | `0x9000_0FFF` |
| Word address | 10 bit |
| Clock | ใช้ clock domain เดียวกับ CPU ในขั้นแรก |
| Boot location | ใช้ ROM/boot mechanism เดิม |
| Cache | ปิด cache ที่เกี่ยวข้องในขั้นทดสอบแรก |

IHP ระบุ macro นี้เป็น SRAM 1024 × 32 bit พร้อม interface สำหรับ mask และ BIST รวมถึง supply pins เช่น `VDD`, `VSS` และ `VDDARRAY` โดยต้องตรวจรายละเอียดจาก datasheet ของ revision ที่ใช้งานจริง [IHP OpenPDK 11181f4 documentation](https://ihp-open-pdk-docs.readthedocs.io/en/latest/contents/reference_libraries/sram.html?utm_source=chatgpt.com)

**ก่อนใช้ `0x9000_0000`** ให้ตรวจว่าไม่ทับ memory region เดิม และ NEORV32 configuration ที่เลือกส่ง address นี้ออก XBUS ได้จริง

---

### 12.3 ความรู้พื้นฐานที่ต้องเข้าใจ

#### 12.3.1 Inferred RAM กับ hard macro

โค้ด RTL ที่ประกาศ array:

```systemverilog
logic [31:0] mem [0:1023];
```

เป็นคำอธิบายพฤติกรรมหน่วยความจำ แต่ไม่ได้รับประกันว่าจะได้ SRAM macro ของ IHP หลัง synthesis

หาก flow ไม่มี technology mapping สำหรับ memory รูปแบบนั้น อาจได้ flip-flop และ multiplexer จำนวนมาก

ใน Lab นี้ใช้ **explicit macro instantiation** โดยมี wrapper แปลง interface ของระบบให้ตรงกับ macro ที่เลือก

#### 12.3.2 SRAM มีหลาย views

| View | หน้าที่ | สิ่งที่ต้องตรวจ |
|---|---|---|
| Datasheet | พฤติกรรมและ operating conditions | Latency, polarity, mask, BIST |
| Behavioral Verilog | Functional simulation | ชื่อ module, dependencies, semantics |
| Black-box header | Synthesis interface | Pin names, direction, width |
| Liberty | Timing และข้อมูล characterization | Cell name, corner, timing arcs |
| LEF | Physical abstract | Size, origin, pins, obstruction |
| GDS | Layout จริง | Top cell และ geometry |
| CDL/SPICE | LVS reference | Subcircuit และ power pins |

LibreLane ใช้ LEF สำหรับ PnR และ GDS สำหรับ stream-out ส่วน timing integration ต้องมี timing representation ที่เหมาะสม เช่น Liberty การปล่อย SRAM เป็น timing black box อาจซ่อน violation ที่ขอบเขต macro ได้ [LibreLane Documentation](https://librelane.readthedocs.io/en/stable/usage/using_macros.html?utm_source=chatgpt.com)

#### 12.3.3 SRAM ไม่ใช่ ROM ที่มีข้อมูลพร้อมหลังเปิดเครื่อง

อย่ากำหนด acceptance ว่า SRAM ต้องอ่านได้ศูนย์ทันทีหลัง reset

Reset ของ adapter มีหน้าที่ล้างสถานะ protocol ส่วนข้อมูล SRAM หลัง power-up ให้ถือว่า **ไม่ทราบค่า** จนกว่าจะเขียน หรือมี initialization mechanism ที่ตรวจสอบแล้ว

---

### 12.4 Step 1 — Freeze baseline และ revision

ก่อนแก้ source ให้ยืนยัน baseline จาก Lab 11:

- CPU boot ได้
- UART แสดงข้อความได้
- GPIO ทำงานได้
- Functional simulation เดิมผ่าน
- Synthesis เดิมไม่มี unresolved module
- มี configuration และ reports ของ baseline ให้เปรียบเทียบ

บันทึก revision ใน repository ที่เกี่ยวข้อง:

```bash
git rev-parse HEAD
git status --short
```

ทำซ้ำใน NEORV32 repository, implementation repository และ PDK source repository หาก PDK ที่ติดตั้งไม่มีข้อมูล Git ให้บันทึก version manifest ของ installation แทน

บันทึกข้อมูลต่อไปนี้:

```text
NEORV32 revision:
Implementation revision:
IHP PDK revision:
LibreLane version:
VHDL export tool/version:
SRAM macro:
Baseline run:
Clock period:
```

**จุดตรวจ:** ผู้เรียนอีกคนต้องทราบได้ว่า integration นี้อ้างอิง source และ PDK ชุดใด

---

### 12.5 Step 2 — Audit SRAM views

กำหนด path ให้ตรงกับเครื่อง:

```bash
export SRAM_LIB_ROOT="$PDK_ROOT/ihp-sg13g2/libs.ref/sg13g2_sram"
export SRAM_MACRO="RM_IHPSG13_1P_1024x32_c2_bm_bist"

test -d "$SRAM_LIB_ROOT"
rg --files "$SRAM_LIB_ROOT" | rg "$SRAM_MACRO"
```

อย่าเลือกไฟล์จากชื่ออย่างเดียว ให้เปิดแต่ละ view และตรวจชื่อ cell ภายใน

ตัวอย่างการค้นหา:

```bash
rg -n 'module|input|output|inout' path/to/sram_model.v

rg -n 'MACRO|SIZE|ORIGIN|SYMMETRY|PIN|USE|LAYER' \
  path/to/sram.lef

rg -n 'cell\s*\(|pg_pin|related_pin|timing_type|operating_conditions' \
  path/to/sram.lib

rg -n -i '\.subckt' path/to/sram.cdl
```

หากไฟล์ถูกบีบอัด ให้แตกไฟล์ไปยังพื้นที่ทำงานก่อนตรวจ โดยคงต้นฉบับ PDK ไว้

สร้าง `sram_view_audit.md`:

| รายการ | ผลที่ต้องบันทึก |
|---|---|
| Macro master name | ชื่อในแต่ละ view สอดคล้องกัน |
| Address width | 10 bit สำหรับ 1,024 word |
| Data width | 32 bit |
| Clock edge | ตาม datasheet |
| Read latency | ตาม datasheetและ model |
| Write enable polarity | Active-high หรือ active-low |
| Mask polarity | ระบุชัดเจน |
| Mask granularity | Bit, byte หรือรูปแบบอื่น |
| Disabled behavior | Hold output หรือพฤติกรรมอื่น |
| BIST mode | Functional-mode tie-off ครบ |
| Power pins | ครบทุก supply |
| LEF size/origin | บันทึกค่าจริง |
| Liberty corners | รายชื่อและ operating conditions |
| LVS reference | Path และ subcircuit name |

**Acceptance:** ห้ามเดินหน้าต่อหากยังไม่ทราบ write-mask polarity หรือ read timing

---

### 12.6 Step 3 — กำหนด memory map และ address decode

สำหรับ SRAM ขนาด 4 KiB:

$$1024 \times 32 / 8 = 4096\ \text{bytes}$$

หาก base address มี alignment 4 KiB:

```systemverilog
assign sram_hit =
    (bus_addr[31:12] == 20'h90000);

assign sram_word_addr =
    bus_addr[11:2];
```

ความหมายของ address:

| Address bits | หน้าที่ |
|---|---|
| `[31:12]` | เลือก SRAM region |
| `[11:2]` | เลือก word 0–1023 |
| `[1:0]` | Byte offset ภายใน word |

ตัวอย่าง:

| CPU byte address | SRAM word index |
|---|---:|
| `0x9000_0000` | 0 |
| `0x9000_0004` | 1 |
| `0x9000_0008` | 2 |
| `0x9000_0FFC` | 1,023 |

ข้อผิดพลาดที่ต้องทดลองตรวจ:

```systemverilog
// ผิดสำหรับ CPU ที่ใช้ byte address
assign sram_word_addr = bus_addr[9:0];
```

การใช้ address bits ผิดทำให้ mapping ไม่เป็นไปตามที่ซอฟต์แวร์เข้าใจ และอาจเกิด alias

**จุดตรวจเพิ่มเติม:** ถ้า adapter ได้รับ request ที่ไม่ใช่ SRAM region ต้องส่งต่อไปยัง slave อื่น หรือคืน error จาก default slave ตามระบบ interconnect ห้ามปล่อย transaction ค้างโดยไม่มีกลไกตอบกลับ

---

### 12.7 Step 4 — กำหนด protocol ของ adapter

อย่าใช้ `stb` เป็นเงื่อนไขที่ต้องค้างตลอด transaction โดยไม่ได้ตรวจพฤติกรรม XBUS ของ revision นั้น

ให้เปิด RTL และ documentation ที่ใช้งาน:

```bash
rg -n 'xbus_|XBUS_|stb|cyc|ack|err' \
  rtl/core/neorv32_xbus.vhd \
  rtl/core/neorv32_top.vhd
```

กำหนด contract ก่อนเขียน FSM:

```text
Request acceptance:
  รับ request เมื่อ adapter ว่างและเห็น request ที่ถูกต้อง

Captured fields:
  address, write data, write enable, byte enable

Outstanding transactions:
  1 transaction

Completion:
  ส่ง ACK หรือ ERR เพียงอย่างใดอย่างหนึ่ง

Write issue:
  SRAM write เกิดเพียงครั้งเดียวต่อ transaction

Read completion:
  ACK หลังข้อมูล SRAM ถูกจับและมีเสถียรภาพ

Reset:
  ไม่ส่ง ACK/ERR ค้าง และไม่เปิด write enable

Abort:
  กำหนดพฤติกรรมเมื่อ master ยกเลิก cycle
```

สำหรับการสอน แนะนำ FSM ที่มี registered command และ registered response:

| State | การทำงาน |
|---|---|
| `IDLE` | รับ request และเก็บ fields |
| `ISSUE` | ให้ SRAM sample command ที่ clock edge |
| `CAPTURE` | จับ read data และสร้าง response |
| `RESPOND` | ให้ master รับ response และจัดการ re-arm |

ลำดับ read ตัวอย่าง:

| Edge | เหตุการณ์ |
|---|---|
| E0 | Adapter รับ request |
| E1 | SRAM sample address และ read enable |
| หลัง E1 | SRAM output เปลี่ยนตาม access time |
| E2 | Adapter จับ read data และยก ACK |
| E3 | Master sample ACK/data |

หาก macro model ใช้ nonblocking assignment ทั้ง SRAM และ adapter ที่ E1 adapter จะยังเห็นค่า output เดิม จึงต้องออกแบบ capture edge ให้ถูกต้อง

**ข้อกำหนดสำคัญ:** หลังตอบกลับแล้วต้องไม่รับ request เดิมซ้ำ การ re-arm ต้องอิง protocol ที่เลือก เช่น รอ request strobe ลดลง หรือรองรับ transaction ถัดไปตาม contract ของ master อย่างชัดเจน

---

### 12.8 Step 5 — ออกแบบ generic SRAM interface

แยก adapter ออกจาก pin names ของ macro:

```systemverilog
// Interface ภายในที่เสนอสำหรับ Lab
logic        mem_en;
logic        mem_we;
logic [9:0]  mem_addr;
logic [31:0] mem_wdata;
logic [3:0]  mem_be;
logic [31:0] mem_rdata;
```

กำหนด semantics ภายใน:

```text
mem_en = 1:
  มี memory command

mem_we = 1:
  เป็น write command

mem_be[n] = 1:
  อนุญาตให้เขียน byte lane n
```

จากนั้นให้ technology wrapper แปลง interface นี้เป็น pins จริงของ IHP macro

สำหรับ bit mask แบบ active-high write-enable:

```systemverilog
logic [31:0] write_bits;

assign write_bits = {
  {8{mem_be[3]}},
  {8{mem_be[2]}},
  {8{mem_be[1]}},
  {8{mem_be[0]}}
};
```

หาก macro ใช้ active-low write mask:

```systemverilog
assign macro_mask = ~write_bits;
```

**ใช้สมการนี้เฉพาะเมื่อ datasheet ยืนยัน semantics นั้น** บาง interface ใช้ชื่อ mask ที่หมายถึง “บิตที่ไม่เขียน” จึงต้องตรวจทั้งชื่อและพฤติกรรม

ตาราง byte lane:

| Byte enable | Byte ที่ถูกเขียน |
|---|---|
| `0001` | `[7:0]` |
| `0010` | `[15:8]` |
| `0100` | `[23:16]` |
| `1000` | `[31:24]` |
| `0011` | `[15:0]` |
| `1100` | `[31:16]` |
| `1111` | ทั้ง word |

ตรวจเพิ่มว่า XBUS ของ configuration นี้จัดตำแหน่ง write data และ byte select อย่างไร ห้ามเพิ่มการ shift ซ้ำ หาก upstream จัด lane มาแล้ว

---

### 12.9 Step 6 — สร้าง technology wrapper

เสนอให้ใช้ชื่อ:

```text
rtl/memory/ihp_sram_1024x32_wrapper.sv
```

ภายใน wrapper ต้องมี:

1. การแปลง enable/write polarity
2. การแปลง byte enable เป็น mask
3. Instance ของ macro จริง
4. Functional-mode tie-off ของ test/BIST pins
5. Power interface ตาม methodology ของโครงการ

จัดทำ pin mapping table ก่อนเขียน instance:

| สัญญาณภายใน | Pin macro จริง | การแปลง |
|---|---|---|
| `clk` | ตรวจจาก datasheet | Clock edge ตรงกัน |
| `mem_en` | ตรวจจาก datasheet | ปรับ polarity |
| `mem_we` | ตรวจจาก datasheet | ปรับ polarity |
| `mem_addr` | ตรวจจาก datasheet | 10-bit word address |
| `mem_wdata` | ตรวจจาก datasheet | 32 bit |
| `mem_be` | ตรวจจาก datasheet | Expand/invert ตาม semantics |
| `mem_rdata` | ตรวจจาก datasheet | Output 32 bit |
| Test/BIST | ทุก input ที่เกี่ยวข้อง | ค่า functional mode |
| Power | ทุก supply pin | Net ที่ถูกต้อง |

**อย่าผูก test pins ทุกตัวเป็นศูนย์โดยอัตโนมัติ** ให้ใช้ functional-mode truth table ของ macro

หาก model อ้างอิง module ย่อย ให้รวม dependency ใน simulation file list ด้วย การมีไฟล์ top model เพียงไฟล์เดียวอาจยังทำให้ elaboration ล้มเหลว

---

### 12.10 Step 7 — แยก simulation และ synthesis file lists

กำหนด file list อย่างน้อยสองชุด:

```text
sim/files_sram.f
synth/files_sram.f
```

| Flow | SRAM representation |
|---|---|
| RTL functional simulation | Behavioral model |
| Synthesis | Black-box declaration |
| STA | Liberty ที่ตรง corner |
| PnR | LEF |
| Stream-out | GDS |
| LVS | Reference ตาม methodology |

Black-box header ต้องมีชื่อ module และ ports ตรงกับ macro จริง และไม่มี implementation ของ memory array

ข้อควรตรวจ:

- ไม่ compile behavioral model กับ black-box module ชื่อเดียวกันใน run เดียว
- ไม่ส่ง behavioral SRAM array เข้า synthesis โดยไม่ตั้งใจ
- ไม่ใช้ black box เพื่อทดสอบ read/write functionality
- ไม่ถือว่าการ resolve module สำเร็จเท่ากับ timing integration สำเร็จ

สำหรับโครงการ VHDL-to-Verilog ให้ตรวจว่าเส้นทาง export คง boundary ของ wrapper ได้จริง ทางเลือกหนึ่งคือ export NEORV32 core แล้วประกอบ SRAM wrapper ใน Verilog top-level หลัง export โดยต้องรักษา clock/reset และ port mapping เดิม

---

### 12.11 Step 8 — Unit test SRAM wrapper

ทดสอบ wrapper ก่อนต่อ CPU

ลำดับพื้นฐาน:

1. Reset adapter
2. เขียน full word ที่ address 0
3. อ่านกลับ
4. เขียน full word ที่ address 1
5. ยืนยัน address 0 ไม่เปลี่ยน
6. เขียน byte ทุก lane
7. เขียน halfword ต่ำและสูง
8. ทดสอบ address 1023
9. ปิด enable และตรวจ disabled behavior
10. ตรวจว่า reset adapter ไม่สร้าง write pulse

ตัวอย่างตรวจ byte mask:

```text
Initial word:  0xAABBCCDD

Write byte lane 0 = 0x11
Expected:      0xAABBCC11

Write byte lane 1 = 0x22
Expected:      0xAABB2211

Write byte lane 2 = 0x33
Expected:      0xAA332211

Write byte lane 3 = 0x44
Expected:      0x44332211
```

หาก mask กลับ polarity ผลอาจเป็นการเขียนทุก byte ยกเว้น byte ที่ต้องการ จึงต้องตรวจ bytes ที่ควรคงค่าไว้ด้วย

**Acceptance:** ทุก test ผ่านกับ model ที่ใช้จริง และ waveform แสดง write pulse เพียงหนึ่ง pulse ต่อ write request

---

### 12.12 Step 9 — ทดสอบ adapter และ bus behavior

เพิ่ม bus-level tests:

| Test | Expected result |
|---|---|
| Read after write | ข้อมูลตรง |
| Request ที่มี wait state | จบด้วย response เดียว |
| Strobe pulse | Request ไม่สูญหาย |
| Request ค้างตาม protocol | ไม่เขียนซ้ำ |
| Read/write สลับกัน | ไม่มี stale read data |
| Reset ระหว่าง transaction | กลับสู่สถานะที่กำหนด |
| Out-of-range address | Error/forward ตาม decode |
| Byte/halfword access | Lane ถูกต้อง |
| Zero byte-enable | ไม่มีบิตถูกเขียน |
| ACK/ERR | ไม่ assert พร้อมกัน |

Assertions ที่ควรมี:

```systemverilog
assert property (@(posedge clk)
  disable iff (!rst_n)
  !(bus_ack && bus_err)
);
```

เพิ่ม transaction scoreboard เพื่อนับ:

```text
accepted_requests
issued_memory_commands
completed_responses
```

หลัง drain ทุก transaction จำนวน request และ response ต้องเท่ากัน ส่วนจำนวน command ต้องตรงกับชนิด request ที่ยอมรับจริง

หากกำหนด fixed latency ให้ตรวจ latency นั้นด้วย แต่ไม่ใช้เลขเดียวกับทุก configuration เพราะ register stages ของ XBUS อาจเปลี่ยน latency

---

### 12.13 Step 10 — ต่อกับ NEORV32 และทดสอบ firmware

ขั้นแรกให้ firmware อยู่ใน memory เดิม และใช้ SRAM ใหม่เป็น scratch region

ตัวอย่าง C สำหรับการทดสอบ word และ byte:

```c
#include <stdint.h>

#define SRAM_BASE  0x90000000u
#define SRAM_WORDS 1024u

static int sram_test(void) {
  volatile uint32_t *m =
      (volatile uint32_t *)(uintptr_t)SRAM_BASE;

  for (uint32_t i = 0; i < SRAM_WORDS; ++i) {
    m[i] = 0xA5A50000u ^ i;
  }

  for (uint32_t i = 0; i < SRAM_WORDS; ++i) {
    if (m[i] != (0xA5A50000u ^ i)) {
      return 1;
    }
  }

  m[0] = 0xAABBCCDDu;

  volatile uint8_t *b =
      (volatile uint8_t *)(uintptr_t)SRAM_BASE;

  b[0] = 0x11u;
  b[1] = 0x22u;
  b[2] = 0x33u;
  b[3] = 0x44u;

  if (m[0] != 0x44332211u) {
    return 2;
  }

  return 0;
}
```

ตัวอย่างนี้สมมติ little-endian byte ordering และ region นี้ไม่มีการ cache ที่บดบัง SRAM transaction ให้เรียกจาก firmware Lab 11 และรายงานผลผ่าน UART API ของ revision เดิม

ข้อความที่แนะนำ:

```text
SRAM TEST START
WORD TEST PASS
BYTE TEST PASS
SRAM TEST PASS
```

เพิ่ม tests:

- All-zero/all-one
- Walking-one/walking-zero
- Address-dependent pattern
- Halfword write
- First/last word
- Neighbor preservation
- Random access พร้อม reference model ใน simulation

**ข้อจำกัด:** Firmware pattern test เป็น functional test ของ integration ไม่ใช่หลักฐาน manufacturing fault coverage หรือ MBIST sign-off

---

### 12.14 Step 11 — ตรวจ synthesis

หลัง export และ synthesis ให้ตรวจ:

```bash
rg -n 'RM_IHPSG13_1P_1024x32_c2_bm_bist' \
  path/to/synthesized_netlist.v

rg -n -i 'unresolved|blackbox|undriven|multiple.*driver|error' \
  path/to/synthesis_reports
```

การค้นข้อความเป็นเพียงขั้นแรก ให้ตรวจ cell statistics หรือ netlist database เพื่อยืนยันจำนวน instance จริง

ตารางตรวจ:

| รายการ | Expected |
|---|---|
| SRAM master | ชื่อตรงกับ views |
| SRAM instance count | 1 |
| Address pins | ต่อครบ 10 bit |
| Data pins | ต่อครบ 32 bit |
| Mask | ต่อครบตาม interface |
| BIST inputs | ไม่ลอย |
| Memory array expansion | ไม่มีการแทน SRAM ใหม่ด้วย FF array |
| Unresolved module | ไม่มีที่ไม่ได้อธิบายไว้ |

เปรียบเทียบ standard-cell count กับ baseline หากเพิ่มขึ้นหลายหมื่น storage bits ให้ตรวจว่า behavioral model หลุดเข้า synthesis หรือไม่

อย่าตั้งชื่อ placement instance จาก source hierarchy เพียงอย่างเดียว ให้ใช้ชื่อที่ปรากฏหลัง synthesis เพราะ flattening อาจเปลี่ยนชื่อ

---

### 12.15 Step 12 — ลงทะเบียน macro ใน LibreLane

เพิ่ม SRAM ใน `MACROS` ของ configuration เดิม โดยใช้ชื่อ master เป็น key และชื่อ instance หลัง synthesis ใน `instances`

LibreLane กำหนด `lib` เป็น dictionary ที่เลือกไฟล์ตาม timing-corner pattern และการ flatten ของ Yosys โดยทั่วไปใช้ชื่อ instance แบบจุด เช่น `u_mem.u_sram` [LibreLane Documentation](https://librelane.readthedocs.io/en/stable/usage/using_macros.html?utm_source=chatgpt.com)

ตัวอย่างโครงสร้าง **ต้องแทน placeholder ก่อนรัน**:

```json
{
  "MACROS": {
    "RM_IHPSG13_1P_1024x32_c2_bm_bist": {
      "gds": ["dir::macros/sram/sram.gds"],
      "lef": ["dir::macros/sram/sram.lef"],
      "vh": ["dir::macros/sram/sram.bb.v"],
      "lib": {
        "<actual-slow-corner-pattern>": [
          "dir::macros/sram/sram_slow.lib"
        ],
        "<actual-typical-corner-pattern>": [
          "dir::macros/sram/sram_typical.lib"
        ],
        "<actual-fast-corner-pattern>": [
          "dir::macros/sram/sram_fast.lib"
        ]
      },
      "instances": {
        "<actual-instance-name>": {
          "location": [500.0, 500.0],
          "orientation": "N"
        }
      }
    }
  }
}
```

พิกัด `[500,500]` เป็นตัวอย่างเท่านั้น ต้องคำนวณจาก core area, macro dimensions, origin และ orientation จริง

ตรวจว่า:

1. Paths resolve ถูกต้อง
2. ชื่อ Liberty cell ตรง master
3. ทุก required corner ได้ SRAM Liberty ที่ถูกต้อง
4. ไม่มี wildcard กว้างจนใช้ typical model กับ slow/fast โดยไม่ตั้งใจ
5. ชื่อ placement instance ตรง synthesized design

---

### 12.16 Step 13 — Floorplan และ macro placement

อ่านขนาดและ origin จาก LEF แล้วตรวจ bounding box หลัง placement ใน OpenROAD/KLayout

สำหรับ orientation `N` และ origin ที่สอดคล้องกับ placement coordinate:

$$x_\text{max}=x_\text{min}+W_\text{macro}$$

$$y_\text{max}=y_\text{min}+H_\text{macro}$$

หาก origin ไม่เป็นศูนย์หรือมีการหมุน ให้ตรวจ bounding box จาก database แทนการใช้สมการนี้ตรง ๆ

แนวทางเลือกตำแหน่ง:

1. อยู่ภายใน core area
2. มีพื้นที่สำหรับ routing รอบ pins
3. ไม่ชน PDN ring หรือ reserved region
4. ด้านที่มี signal pins หันเข้าหา adapter
5. มีพื้นที่สำหรับ CTS และ hold buffers
6. ไม่สร้างทางเดินแคบระหว่าง macro กับขอบ core
7. Orientation ถูกต้องตาม macro และ methodology

เริ่มด้วยช่องว่างรอบ macro ที่เพียงพอ แล้วปรับจาก congestion และ PDN geometry จริง แทนการใช้ halo ตัวเลขเดียวกับทุก design

**ผลส่ง:** ภาพ floorplan ที่แสดง SRAM, adapter region, routing access และ PDN

---

### 12.17 Step 14 — ต่อ power pins และตรวจ PDN

ทำ power mapping ให้ครบ:

| Pin ของ macro | Net ระบบ | หลักฐานที่ต้องตรวจ |
|---|---|---|
| `VDD` หากมีใน view | Core supply ตาม methodology | Logical และ physical connection |
| `VSS` หากมีใน view | Core ground | Logical และ physical connection |
| `VDDARRAY` หากมีใน view | Supply ตาม datasheet | ไม่ลอยและแรงดันถูกต้อง |
| Supply อื่น | ตาม revision | ตรวจทุก pin |

อย่าต่อ SRAM เข้ากับ I/O supply โดยอนุมานจากชื่อ net และอย่าต่อ `VDDARRAY` ร่วมกับ `VDD` จนกว่าจะตรวจ operating requirements

ตรวจสามระดับ:

1. **Logical connectivity:** Power pins อยู่บน net ที่ถูกต้อง
2. **Physical connectivity:** มี metal/via path เข้าถึง pins
3. **Geometry legality:** Width, spacing, enclosure และ via geometry ถูกต้อง

ขั้นตอนตรวจ:

1. เปิด ODB หลัง PDN generation
2. ตรวจ power net ของ SRAM ทุก terminal
3. ตรวจการทับซ้อน strap กับ pin geometry
4. Export DEF/GDS
5. ตรวจ vias และ landing metals
6. รัน DRC deck ที่ใช้กับโครงการ
7. ตรวจ power connectivity อีกครั้งหลัง routing

**Acceptance:** Connectivity ผ่านอย่างเดียวไม่เพียงพอ หาก via geometry ยังผิด หรือ supply pin บางกลุ่มไม่ได้รับการเชื่อมต่อจริง

---

### 12.18 Step 15 — วิเคราะห์ timing รอบ SRAM

ต้องตรวจอย่างน้อย:

| Path | สิ่งที่ต้องตรวจ |
|---|---|
| Adapter register → SRAM address | Setup/hold |
| Adapter register → enable/write/mask | Setup/hold |
| Adapter register → write data | Setup/hold |
| SRAM output → response register | Clock-to-output และ setup/hold |
| Response register → XBUS receiver | Response timing |
| CTS → SRAM clock pin | Clock propagation, slew, constraints |

แนวคิด setup ฝั่ง write:

$$t_{cq,\text{adapter}}
+t_{\text{logic}}
+t_{\text{wire}}
+t_{\text{setup,SRAM}}
\le T_\text{clk}+t_\text{skew}-t_\text{uncertainty}$$

แนวคิด setup ฝั่ง read:

$$t_{cq,\text{SRAM}}
+t_{\text{wire}}
+t_\text{setup,response}
\le T_\text{clk}+t_\text{skew}-t_\text{uncertainty}$$

สมการนี้ใช้เพื่ออธิบาย budget ส่วนผลจริงต้องอิง STA reports และ timing arcs ของ Liberty

ตรวจ reports ว่า:

- SRAM cell ถูก link กับ timing model จริง
- Clock ไปถึง SRAM clock pin
- มี paths เข้า address/data/control
- มี paths ออกจาก SRAM data pins
- Setup และ hold ได้รับการตรวจ
- Corner mapping ถูกต้อง
- ไม่มี broad false path ตัด SRAM ทั้งก้อน

**ห้ามใช้ false path เพื่อกลบ timing failure** และการเพิ่ม wait state จะช่วย timing ก็ต่อเมื่อ microarchitecture ทำให้ launch/capture timing เปลี่ยนจริง เช่นเพิ่ม pipeline register การหน่วง ACK อย่างเดียวไม่ทำให้ path ทางกายภาพสั้นลง

---

### 12.19 Step 16 — Routing, extraction และ physical verification

หลัง detailed routing:

1. ตรวจ signal connectivity ทุก SRAM pin
2. ตรวจ opens/shorts
3. ตรวจ routing access บริเวณ macro
4. Extract parasitics ของ interconnect
5. รัน STA พร้อม SRAM timing model
6. ตรวจ antenna ตาม methodology
7. ตรวจ final GDS ว่ามี macro layout จริง
8. รัน DRC/LVS ตาม required decks และ integration strategy

สำหรับ extraction ให้แยก:

- Interconnect ภายนอก macro: ใช้ top-level parasitic extraction
- Timing ภายใน SRAM: ใช้ macro model ที่ characterization แล้ว

อย่าซ้ำซ้อน internal parasitics ที่รวมอยู่ใน timing model เว้นแต่ methodology กำหนดวิธีอื่นไว้

LVS ต้องระบุว่าตรวจ transistor-level SRAM ภายใน หรือใช้ hierarchical/black-box boundary strategy หากตรวจเฉพาะ boundary ให้รายงานขอบเขตนั้นอย่างชัดเจน

---

### 12.20 Step 17 — ทดลองย้ายข้อมูลไป SRAM

หลัง scratch test ผ่าน จึงทดลอง linker section สำหรับข้อมูลเฉพาะกลุ่ม

ตัวอย่าง linker fragment:

```ld
MEMORY
{
  EXT_SRAM (rw) : ORIGIN = 0x90000000, LENGTH = 4K
}

.sram_scratch (NOLOAD) :
{
  . = ALIGN(4);
  *(.sram_scratch)
  . = ALIGN(4);
} > EXT_SRAM
```

ให้นำไปผสานกับ linker script เดิม ไม่ใช้ fragment นี้แทน linker script ทั้งไฟล์

ตัวอย่าง C:

```c
__attribute__((section(".sram_scratch"), aligned(4)))
volatile uint32_t scratch[256];
```

ตรวจ linker map ว่า:

- Section อยู่ใน SRAM region
- ขนาดไม่เกิน 4 KiB
- ไม่มี overlap กับ object อื่น
- ไม่คาดหวัง initial values ใน `NOLOAD`
- Firmware เขียนข้อมูลก่อนอ่าน

ลำดับต่อยอดที่แนะนำ:

1. Scratch buffer
2. `.bss` พร้อม startup zeroing
3. `.data` พร้อม copy จาก load image
4. Stack พร้อมการทดสอบ nested calls/interrupts
5. Instruction execution พร้อม boot/loading และ instruction-fetch verification

อย่าย้าย stack หรือ boot code ในการทดลองครั้งแรก เพราะ failure จะวิเคราะห์ได้ยากขึ้น

---

### 12.21 งานต่อยอด — แทน internal DMEM/IMEM

เมื่อ XBUS SRAM integration ผ่านแล้ว ให้เลือกหนึ่งหัวข้อ:

#### A. แทน internal DMEM

ตรวจ internal bus contract ของ revision จริง แล้วสร้าง wrapper ที่รักษา:

- Request/response semantics
- Byte enables
- Latency และ stall
- Error behavior
- Reset behavior
- Arbitration ที่เกี่ยวข้อง

อย่าเปลี่ยน inferred array เป็น instance โดยพิจารณาเพียง data width

#### B. แทน internal IMEM

ต้องเพิ่มการตรวจ:

- Instruction-fetch latency
- Boot/loading mechanism
- SRAM contents ก่อน CPU fetch
- Instruction cache
- การเขียนโปรแกรมและ instruction synchronization
- Linker load/run addresses

#### C. เพิ่ม SRAM หลาย banks

หากต้องการ 16 KiB จาก macro 4 KiB จำนวนสี่ตัว:

```systemverilog
assign bank_select = bus_addr[13:12];
assign word_index  = bus_addr[11:2];
```

ต้องเก็บ bank selection ของ request ไว้จน response เสร็จ แล้วใช้ค่าที่เก็บไว้เลือก read data ห้ามเลือกจาก live address ที่อาจเปลี่ยนไปแล้ว

#### D. MBIST

แยก functional integration ออกจาก DFT project โดยกำหนด test controller, test access, reset/isolation, coverage และ test constraints เพิ่มเติม ไม่ถือว่าการมี BIST pins หมายถึงมี MBIST flow พร้อมใช้งานแล้ว

---

### 12.22 ปัญหาที่พบบ่อย

| อาการ | สาเหตุที่ควรตรวจ | วิธีแก้ |
|---|---|---|
| Module missing ใน simulation | Model dependency ไม่ครบ | Audit module definitions และ file list |
| SRAM กลายเป็น FF array | Behavioral model เข้า synthesis | แยก model/black box |
| อ่านได้ค่าจาก transaction ก่อน | Capture เร็วเกินไป | ตรวจ edge และเพิ่ม capture stage |
| เขียน byte แล้ว byte อื่นเสีย | Mask polarity/lane mapping ผิด | ทดสอบทุก lane และ untouched bytes |
| Address ห่าง 4 byte แต่ข้อมูลชนกัน | Word address/decode ผิด | ตรวจ `[11:2]` และ region decode |
| CPU ค้าง | ACK/ERR หายหรือ request สูญหาย | ตรวจ XBUS waveform และ FSM |
| Write เกิดซ้ำ | รับ request เดิมซ้ำ | แก้ acceptance/re-arm |
| Placement หา instance ไม่เจอ | ชื่อหลัง flatten เปลี่ยน | ใช้ชื่อจาก synthesized netlist |
| STA ผ่านแต่ไม่มี SRAM paths | Macro timing model ไม่ถูก link | ตรวจ Liberty และ corner mapping |
| PDN ต่อไม่ครบ | Supply pins ตกหล่น | Audit ทุก PG terminal |
| GDS ไม่มี SRAM geometry | Missing/wrong macro GDS | ตรวจ master และ final hierarchy |
| ย้าย `.data` แล้ว firmware พัง | Startup copy ไม่รองรับ | ตรวจ load/run addresses และ startup |

---

### 12.23 ผลส่งและ acceptance matrix

| Gate | หลักฐาน | เกณฑ์ผ่าน |
|---|---|---|
| Revision freeze | Manifest | ระบุ source/tools/PDK |
| Macro views | View audit | ชื่อและ interface สอดคล้อง |
| Wrapper test | Log/waveform | Read/write/mask ถูกต้อง |
| Bus integration | Scoreboard/waveform | ไม่มี lost/duplicate transaction |
| CPU test | UART log | Word/byte/halfword tests ผ่าน |
| Synthesis | Netlist/statistics | SRAM instance อยู่ครบ |
| Placement | ODB/ภาพ | Bounds/orientation ถูกต้อง |
| PDN | Connectivity + geometry reports | Supply ครบและ geometry ผ่าน |
| Timing | Corner reports | SRAM boundary coverage ครบ |
| Routing | Reports | Connectivity และ required checks ผ่าน |
| Stream-out | Final GDS | SRAM geometry อยู่จริง |
| LVS | Report + methodology | Required scope ผ่าน |
| Handoff | Configuration/file lists/hashes | ผู้เรียนอีกคนตรวจซ้ำได้ |

ใช้สถานะ:

- `PASS`
- `FAIL`
- `NOT RUN`
- `WAIVED` พร้อมผู้อนุมัติและเหตุผล

**`NOT RUN` ห้ามนับเป็น `PASS`**

---

### 12.24 คำถามท้าย Lab

1. เหตุใด RTL memory array จึงไม่รับประกันว่าจะได้ IHP SRAM macro?
2. เหตุใด SRAM 1024 × 32 จึงใช้ word address `[11:2]` สำหรับ byte-addressed bus?
3. การขยาย byte enable เป็น bit mask ต้องระวังอะไร?
4. เหตุใด read data อาจยังเป็นค่าเดิมที่ clock edge ซึ่ง SRAM รับ address?
5. Adapter ป้องกัน write ซ้ำจาก request ที่ค้างได้อย่างไร?
6. เหตุใด timing black box จึงไม่เพียงพอสำหรับ sign-off?
7. Logical power connection ต่างจาก physical PDN connection อย่างไร?
8. การเพิ่ม wait state โดยหน่วง ACK อย่างเดียวช่วย timing path หรือไม่?
9. SRAM สามารถใช้แทน boot ROM ได้ทันทีหรือไม่?
10. Firmware memory test ต่างจาก manufacturing memory test อย่างไร?

**เกณฑ์จบ Lab:** ผู้เรียนต้องอธิบายและแสดงหลักฐานได้ว่า SRAM ถูกใช้งานจริงทั้งใน transaction simulation, synthesized netlist และ final layout พร้อมการตรวจ power และ timing ที่ครบตามขอบเขตที่ประกาศไว้