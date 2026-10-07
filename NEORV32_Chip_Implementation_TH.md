# คู่มือ Chip Implementation of NEORV32 ด้วย LibreLane และ IHP SG13G2

คู่มือภาษาไทยและโปรเจกต์ประกอบ • รุ่นเริ่มต้น 2026-10-05

## 1. เป้าหมายและขอบเขตผลลัพธ์

โครงการนี้เตรียมเส้นทาง VHDL → GHDL synthesis → Verilog → LibreLane Chip flow → full-chip GDSII สำหรับ NEORV32 โดยใช้ IO pad และ bondpad ของ IHP template เป็นพื้นฐาน มี RTL จริงของ NEORV32 รวมในแพ็กเกจ ไม่มีขั้นตอนดาวน์โหลด CPU ที่อาศัย branch ซึ่งเปลี่ยนได้

คำว่า “พร้อมรัน” ในคู่มือนี้หมายถึงมี source, wrapper, firmware smoke, testbench, config, SDC และคำสั่งครบสำหรับเครื่องที่มี dependencies และ PDK ไม่ได้หมายความว่าแพ็กเกจนี้มี GDS ที่ผ่าน sign-off แล้ว การติดตั้ง Nix/PDK ต้องเข้าถึงเครือข่ายและมีพื้นที่ดิสก์เพียงพอ

ผลที่ตรวจจริง: VHDL simulation, GHDL export, simulation ของ exported Verilog และ Yosys structural check ผ่าน รายการที่ยังไม่รันคือ technology mapping, full-chip implementation, pad simulation และ physical sign-off อ่านหลักฐานใน VALIDATION.md ห้ามนำ NOT RUN ไปนับเป็น PASS

## 2. Snapshot ที่ใช้

| ส่วนประกอบ | Snapshot |
|---|---|
| NEORV32 | af905649fdbe650245af700690e435b58601ef7e |
| IHP ihp-sg13-librelane-template | 40ca8b9d5e8b3f87b2a36d590517958df21ee241 |
| LibreLane ใน flake.lock ของ template | aaf7a938766e0708ca894776d81e2c2cfb12b7e7 |
| PDK commit ตาม Makefile ของ template | 22f43352dd8219f9007eb659e422e0d5fe28c5fb |
| เครื่องมือที่ใช้ตรวจในสภาพแวดล้อมจัดทำ | GHDL 4.1.0 / Yosys 0.33 / Icarus 12.0 |

ไฟล์ flake.lock เดิมถูกเก็บไว้ เพิ่ม GHDL, Python และ utility ใน flake.nix เท่านั้น ไม่อัปเดต lock โดยอัตโนมัติ การเปลี่ยน snapshot เครื่องมือหรือ PDK ต้องรัน verification ใหม่ทั้งหมด

NEORV32 เป็น VHDL และใช้ library `neorv32`; ให้วิเคราะห์ source ตาม `rtl/file_list_core.f` ไม่เรียงไฟล์ตามชื่อเอง โปรเจกต์ไม่ได้ให้ LibreLane อ่าน VHDL โดยตรง และไม่ได้สมมติว่ามี Yosys GHDL plugin เพราะใช้ standalone `ghdl --synth --out=verilog`

## 3. สถาปัตยกรรมเริ่มต้น

| รายการ | ค่าเริ่มต้น |
|---|---|
| Core | Single-core RV32I, Zicsr/Zifencei และ Zicntr |
| Clock | 50 MHz / 20 ns |
| Boot mode | 2: pre-initialized IMEM ROM |
| IMEM address window | 4 KiB, base 0x00000000 |
| DMEM | 1 KiB, base 0x80000000 |
| Register file | CPU_RF_ARCH_SEL=2: register-based with reset |
| Peripheral | GPIO 8 inputs/8 outputs, UART0, CLINT, SYSINFO |
| CPU cache, XBUS, OCD, TRNG | ไม่เปิดใช้ |
| Memory implementation | ROM logic + inferred RAM mapped to standard cells |
| Pad ring | 24 pads: 20 signal + 4 supply pads |
| Die/Core | 1600×1600 µm / (365,365) ถึง (1235,1235) µm |
| Placement density target | 35% เป็นค่าเริ่มต้น ต้องประเมินหลัง technology synthesis |

ขนาด IMEM window ไม่ใช่จำนวน ROM bits ที่คงอยู่หลัง optimize: smoke image มีเพียง 144 bytes และ ROM อาจลดลงมาก ส่วน DMEM มี 8192 storage bits การเพิ่ม IMEM-RAM หรือ DMEM โดยไม่ใช้ macro เพิ่ม flip-flop/mux และอาจทำให้ area/routing ไม่ผ่าน

Boot mode 2 ฝังโปรแกรมเป็น ROM ใน silicon การเปลี่ยนโปรแกรม C แล้วสังเคราะห์ใหม่จะเปลี่ยนวงจร ROM ไม่ใช่การโหลด firmware ใหม่หลังผลิต การทดสอบเริ่มต้นจึงเน้นความแน่นอนของ flow; หากต้องการ UART bootloader ให้พัฒนา profile RAM และตรวจความจุ/firmware/runtime แยกก่อนนำไปใช้

## 4. โครงสร้างโปรเจกต์

| ตำแหน่ง | หน้าที่ |
|---|---|
| vendor/neorv32/ | RTL, software framework, datasheet source และ license ของ CPU |
| rtl/neorv32_asic.vhd | ASIC SoC configuration และ port ที่ใช้งาน |
| rtl/chip_core.sv | เชื่อม GPIO/UART เข้ากับ exported NEORV32 |
| rtl/chip_top.sv | SG13G2 input/output/power pads |
| ip/ | GDS/LEF bondpad จาก IHP template |
| librelane/config.yaml | Chip flow, pad placement, die/core, PDN และ source list |
| librelane/chip_top.sdc | Clock, interface timing budgets และ reset exception |
| scripts/analyze.sh, export.sh | VHDL analysis และ export แบบไม่ใช้ plugin |
| scripts/smoke_image.py | สร้าง RV32I smoke ROM โดยไม่ต้องมี compiler |
| scripts/doctor.py | ตรวจ executable และ PDK directories |
| scripts/run_chip.sh | ตรวจ prerequisites, smoke, structural check แล้วรัน chip |
| sim/soc_tb.vhd, soc_tb.v | ตรวจ firmware signatures ของ VHDL/Verilog |
| sw/ | smoke binary/assembly และตัวอย่าง C UART/GPIO |
| build/neorv32_asic.v | Verilog ที่ export และทดสอบแล้ว; สามารถสร้างใหม่ได้ |
| docs/signoff.csv | แบบฟอร์มหลักฐาน sign-off |
| PACKAGE_SHA256.json | hash ของไฟล์ที่ส่งมอบ |

เก็บ original IMEM image ไว้ใน `docs/upstream-neorv32-imem-image.vhd` เพราะ vendor image ถูกแทนด้วย smoke firmware ส่วน RTL ที่เหลือเก็บตาม snapshot อ่านการระบุ license ใน NOTICE.md

## 5. เตรียม Ubuntu/WSL2 และ Nix

ใช้ Linux filesystem เช่น `~/labs` สำหรับ build บน WSL2 เพื่อลด overhead ของไฟล์จำนวนมาก ตรวจ Nix ที่ติดตั้งอยู่ก่อน:

```bash
nix --version
nix show-config | rg 'experimental-features'
```

ถ้าใช้ flakes ให้เปิด `nix-command flakes` ในการตั้งค่า Nix ตามวิธีติดตั้งทางการ ไม่ต้องเปลี่ยน Nix channel เพื่อรันแพ็กเกจนี้

แตกแพ็กเกจและเข้า environment:

```bash
mkdir -p ~/labs
cd ~/labs
unzip /path/to/NEORV32_IHP_SG13G2_Project.zip
cd neorv32-ihp-sg13g2
nix-shell
```

หรือใช้ flakes:

```bash
nix develop --extra-experimental-features 'nix-command flakes'
```

Environment อ้างอิง flake.lock ที่ให้มา การดาวน์โหลดครั้งแรกอาจใช้เวลานาน หาก binary cache หรือ GitHub เข้าไม่ได้ ให้แก้ network/cache configuration ก่อน ไม่ควรแก้ hash ให้ข้ามการตรวจสอบ

ตรวจ executable:

```bash
command -v ghdl yosys librelane ciel iverilog
command -v openroad klayout netgen
make help
ghdl --version
yosys -V
librelane --version
```

GHDL ที่เลือกต้องมี synthesis และ `--out=verilog`; การมี executable อย่างเดียวไม่ได้ยืนยันความสามารถ export โดย `make export` เป็นการตรวจจริง

## 6. ติดตั้ง PDK ให้ตรง snapshot

```bash
make pdk
make doctor
```

`make pdk` เรียก Ciel enable ด้วย commit ที่กำหนด ไม่ดาวน์โหลด source PDK เปล่าแล้วสมมติว่าใช้กับ LibreLane ได้ PDK ต้องมี built configuration, Liberty, LEF, GDS, IO views และ DRC/LVS decks

ถ้ามี PDK ที่เตรียมแล้ว:

```bash
export PDK_ROOT="$HOME/.ciel"
make doctor
```

`PDK_ROOT` คือ parent ของ `ihp-sg13g2` ไม่ใช่ตัว directory `ihp-sg13g2` เอง คำสั่งรันใช้ `--manual-pdk` เพื่อบอกว่าเตรียม PDK แล้วและไม่ให้เลือกเวอร์ชันเอง ตรวจผล doctor; สำเร็จหมายถึงเครื่องมือและ directory สำคัญมีอยู่ ไม่ใช่ sign-off compatibility certificate

เก็บ `build/environment.json`, version output และ PDK revision ร่วมกับผลรัน ถ้าหา PDK snapshot ไม่ได้ ให้เลือก snapshot ที่เข้ากับ LibreLane อย่างมีเหตุผล บันทึกการเปลี่ยนใน dependencies.lock.json และรันทดสอบใหม่ อย่าอ้างผลจาก snapshot เดิม

## 7. Quick start: ทดสอบก่อน implementation

```bash
make verify
make firmware-smoke
make sim
make sim-verilog
make check-verilog
make synth
make chip
```

หรือหลังเข้า Nix shell และติดตั้ง PDK แล้ว:

```bash
bash scripts/run_chip.sh
```

script หยุดที่ error แรก ไม่มีการข้าม Checker/DRC/LVS เพื่อให้ flow ดูเหมือนสำเร็จ เป้าหมาย `make chip` สร้าง Verilog ใหม่ก่อนเสมอ แล้วใช้ full Chip flow และ single worker `--jobs 1` เพื่อลดการใช้หน่วยความจำพร้อมกัน ไม่ได้รับประกันว่า OpenROAD ทุก subprocess จะใช้ thread เดียว

กำหนด tag และเก็บ log ได้ดังนี้:

```bash
set -o pipefail
make chip RUN_TAG=NEORV32_FIRST_CHIP 2>&1 | tee chip-run.log
```

`pipefail` สำคัญ เพราะ pipeline กับ tee อาจปิดบัง exit status ที่ผิดพลาดหากไม่ตั้งไว้ ก่อน rerun tag เดิมให้ดูว่ามี run directory อยู่แล้วหรือไม่ ใช้ tag ใหม่เมื่อเป็นการทดลอง configuration ใหม่

## 8. Lab 1 — Audit source และ interface

1. อ่าน dependencies.lock.json และ NOTICE.md เพื่อทราบ snapshot/ไฟล์ vendor ที่ปรับแก้
2. อ่าน generic map ใน rtl/neorv32_asic.vhd และตรวจว่าไม่มี dual-core/cache/TRNG ถูกเปิดโดยไม่ได้ตั้งใจ
3. ตรวจ `IMEM_SIZE=4096`, `DMEM_SIZE=1024`, `CLOCK_FREQUENCY=50000000`
4. ตรวจ chip_core ว่า input_in[8] คือ UART RX และ output_out[8] คือ UART TX
5. ตรวจ pad instance ทุกชื่อว่าปรากฏใน PAD_* ครบครั้งเดียว

ผลส่ง: source inventory, generic table และ pin mapping Acceptance: ทุก active peripheral มี port connection/ค่าที่ตั้งใจ ไม่มีการใช้ stub CPU หรือ black box แทนวงจรหลัก

## 9. Lab 2 — VHDL analysis และ ROM smoke

```bash
make firmware-smoke
make sim 2>&1 | tee build/vhdl-smoke.log
```

smoke program เริ่มที่ PC=0 ไม่ใช้ stack หรือ libc และใช้ RV32I เท่านั้น ลำดับตรวจคือ:

1. เขียน 0x12345678 ที่ DMEM base
2. LBU ที่ offset 1 ต้องอ่าน 0x56
3. LHU ที่ offset 2 ต้องอ่าน 0x1234
4. SB 0xA5 ที่ offset 1 แล้ว LW ต้องอ่าน 0x1234A578
5. SH 0x05A3 ที่ offset 2 แล้ว LW ต้องอ่าน 0x05A3A578
6. เขียน GPIO output เป็น 0xA5, หน่วงเวลา แล้วเขียน 0x5A
7. หากเปรียบเทียบผิด เขียน GPIO=0xFF และค้าง

testbench ต้องเห็น A5 ก่อน 5A มี timeout และ fail signature จริง ผลจัดทำผ่านที่ 13.650 µs เทียบกับ timeout 1 ms เพื่อมี margin โดยไม่ให้ test ค้างตลอดไป

GHDL อาจรายงาน numeric_std metavalue ที่ time zero ก่อน reset settle ต้องแยกจาก warning ที่เกิดต่อหลังเริ่มทำงาน ห้ามปิด assert error ทั้งหมดเพื่อให้ test ผ่าน

ผลส่ง: log และ build/soc.ghw Acceptance: PASS signature, ไม่มี failure/timeout ทดสอบนี้ครอบคลุม directed CPU/memory/GPIO path ไม่ใช่ CPU compliance suite หรือ UART test

## 10. Lab 3 — Export VHDL เป็น Verilog

```bash
make export
make sim-verilog
```

analyze.sh แทน prefix `$NEORV32_HOME` ด้วย path vendor แล้ววิเคราะห์ตาม file_list_core.f ภายใน library neorv32 จากนั้นวิเคราะห์ wrapper ใน work library พร้อม `-Pbuild/ghdl` สำหรับค้น library

export.sh ใช้:

```bash
ghdl --synth --std=08 --workdir=build/ghdl -Pbuild/ghdl \
  --out=verilog neorv32_asic > build/neorv32_asic.v.tmp
```

สคริปต์เปลี่ยนชื่อไฟล์เมื่อสำเร็จและไฟล์ไม่ว่างเท่านั้น เพื่อป้องกัน truncated Verilog กลายเป็น input ของ flow การสังเคราะห์ elaborated hierarchy กำจัด feature ที่ปิดไว้และ flatten record interfaces ไปเป็น ports ที่ Verilog เข้าใจได้

Icarus testbench รันโปรแกรมเดียวกันกับ VHDL; ผลจัดทำผ่านที่ 13.630 µs ความต่างไม่กี่ clock phase มาจาก scheduling ของ testbench จึงตรวจลำดับค่าที่สังเกตได้ ไม่ใช้ timestamp ตรงกันทุก ps เป็น acceptance

เพิ่มเติม: `make sim-chip-wiring` ตรวจการเชื่อมต่อ chip_top/chip_core ด้วย behavioral pad mocks เท่านั้น models เหล่านี้ไม่อยู่ใน VERILOG_FILES และห้ามใช้แทน PDK views ใน synthesis/sign-off

ผลส่ง: exported Verilog และ simulation log Acceptance: module neorv32_asic มีอยู่, hierarchy ครบ, firmware ผ่านทั้งสองภาษา นี่ยังไม่ใช่ formal equivalence

## 11. Lab 4 — Structural synthesis check

```bash
make check-verilog
rg -n 'Warning|ERROR|Latch inferred|Number of cells' build/yosys-check.log
```

ลำดับ Yosys คือ read_verilog → hierarchy -check → proc → memory → opt → check -assert → stat ตรวจ missing module, conflicting drivers และปัญหาโครงสร้างที่เครื่องมือรายงานก่อนเข้า PDK

หลัง memory pass DMEM จะเป็น storage registers/mux ยังไม่ใช่ RM_IHPSG13 SRAM macro ตัวเลข 20,902 generic cells ที่ตรวจในสภาพแวดล้อมจัดทำเป็นจำนวนก่อน technology mapping ไม่ใช่จำนวน SG13G2 standard cells และใช้แทน area report ไม่ได้

ตรวจ latch report จริง: baseline ไม่พบ latch inferred ใน Yosys proc conversion เพราะใช้ register file architecture 2 หากเปลี่ยน CPU_RF_ARCH_SEL=3 จะเป็น latch-based architecture โดยตั้งใจ ต้องประเมิน flow และ timing methodology ใหม่

ผลส่ง: Yosys report Acceptance: check -assert ผ่าน, ไม่มี unresolved CPU module, memory implementation เข้าใจได้

## 12. Lab 5 — Technology synthesis ใน LibreLane

```bash
make synth RUN_TAG=NEORV32_SYNTH_01
```

หยุด flow หลัง Yosys.Synthesis เพื่อประเมินก่อนเสียเวลาวางและเดินสาย ตรวจ reports ภายใน run:

```bash
rg -n 'ERROR|Warning|unmapped|multiple conflicting|no driver|logic loop' \
  librelane/runs/NEORV32_SYNTH_01
```

ต้องอ่าน synthesis area, sequential count, logic count และ memory mapping รวมกับ timing pre-PNR เทียบกับ core area 756,900 µm² และ utilization จริง หาก density เป้าหมายต่ำกว่าความต้องการให้เพิ่ม core/die หรือลด memory อย่าเพิ่ม density จน routing ไม่มีพื้นที่

IO cells ต้องคงเป็น pad instances ไม่ถูกแทนด้วย logic gates Bondpad เพิ่มใน physical steps ผ่าน EXTRA_GDS/EXTRA_LEFS และ PAD_BONDPAD_NAME ไม่ใช่ CPU macro

ผลส่ง: technology netlist, area และ timing overview Acceptance: mapped cells อยู่ใน library ที่ถูกต้อง ไม่มี unmapped/unsupported storage

## 13. Lab 6 — Pad ring และ pin mapping

| Signal | Pad-facing port | Core function |
|---|---|---|
| Clock | clk_PAD | clk_i 50 MHz |
| Reset | rst_n_PAD | rstn_i active-low |
| GPIO inputs | input_PAD[7:0] | gpio_i[7:0] |
| UART RX | input_PAD[8] | uart_rxd_i |
| GPIO outputs | output_PAD[7:0] | gpio_o[7:0] |
| UART TX | output_PAD[8] | uart_txd_o |
| Core supply | vdd_pad / vss_pad | VDD / VSS |
| IO supply | iovdd_pad / iovss_pad | IOVDD / IOVSS |

| ด้าน | Pad instance order ใน config |
|---|---|
| SOUTH | clk_pad, rst_n_pad, inputs[8].input_pad, outputs[8].output_pad, vdd_pads[0].vdd_pad, vss_pads[0].vss_pad |
| WEST | inputs[0] ถึง inputs[5] |
| NORTH | inputs[6], inputs[7], outputs[0], outputs[1], iovdd_pads[0].iovdd_pad, iovss_pads[0].iovss_pad |
| EAST | outputs[2] ถึง outputs[7] |

มีด้านละ 6 pads การตีความทิศทางการเรียงตามแกนของ flow ต้องตรวจ DEF/layout จริงก่อนกำหนด package pin number ห้ามถือว่า index table นี้เป็นหมายเลข package ที่อนุมัติแล้ว

ชื่อ bracket ใน YAML ใช้ escaped form เช่น `inputs\[0\].input_pad` ให้เป็นชื่อที่ match netlist ของ flow ตรวจ pad instances หลัง synthesis และ floorplan อย่าแก้เฉพาะ PAD_* โดยไม่แก้ RTL

Supply เป็นคนละโดเมน ต้องใช้ voltage ของ core และ IO ตาม PDK/package specification ห้ามต่อ IOVDD เข้ากับ VDD โดยอัตโนมัติ และต้องประเมินจำนวน power pads, current, ESD, latch-up และ package requirements ก่อนใช้ผลิตจริง baseline มีเพียงหนึ่งคู่ต่อโดเมนเพื่อทดลอง flow

## 14. Lab 7 — Timing constraints และ clock/reset

SDC สร้าง clock ที่ external clk_PAD เพื่อรวม delay ของ input clock pad ไม่สร้าง clock ภายในอีกตัวซ้อนกัน CLOCK_NET=clk_pad/p2c ใช้ระบุทางเดิน clock ใน flow

กำหนด input max/min = 2/0 ns, output max/min = 4/0 ns, uncertainty=0.25 ns, transition=0.15 ns และ load=0.033442 ตาม interface budget เริ่มต้นของแบบฝึกหัด ต้องแทนด้วยข้อมูล board/package และ corner specification เมื่อทำผลิตภัณฑ์จริง

GPIO/UART asynchronous inputs มี synchronization ภายใน NEORV32 แต่ synchronous IO budget ไม่ใช่ CDC sign-off เมื่อจะปรับ false paths ให้ระบุเฉพาะเส้นทางเข้าขา D ของ first synchronizer จาก netlist จริง ห้าม false-path ทั้ง subsystem เพราะอาจปิดบัง internal timing

reset ใช้ prototype exception `set_false_path -from rst_n_PAD` เพื่อแยกจาก synchronous data budget; exception นี้ไม่ยืนยัน recovery/removal หรือความปลอดภัยของ reset release ต้องตรวจ reset circuitry, asynchronous assertion, synchronized deassertion และ RDC/STA ตาม methodology ก่อน sign-off ไม่ควรเปลี่ยน reset core โดยไม่ตรวจ requirement ของ NEORV32

ผลส่ง: clock/constraint coverage และรายงาน unconstrained endpoints Acceptance: clock มีอยู่จริง, ไม่มี data path ที่ไม่ตั้งใจหลุด constraint และ reset waiver มีเหตุผล/ผู้ทบทวน

## 15. Lab 8 — Floorplan และ PDN

เริ่ม full Chip flow:

```bash
make chip RUN_TAG=NEORV32_CHIP_01
```

config ใช้ PDN ของ flow เป็นฐานและเปิด core ring; ไม่ใช้ custom pdn_cfg.tcl ที่อ้าง sram_0/sram_1 จาก counter demo เพราะ project นี้ไม่มี SRAM macros และ API `set_global_connections` อาจแตกต่างตาม LibreLane version

ตรวจ floorplan หลัง initial placement: pad ring ถูกล้อมรอบ core, bondpad อยู่ตรง pad ที่ตั้งใจ, core rows มีพื้นที่, VDD/VSS ring ไม่ชน IO region และ die/seal ring bounds ตรง methodology

ตรวจ power connectivity อย่างน้อย four domains/nets ที่เกี่ยวข้องกับ IO: VDD, VSS, IOVDD, IOVSS หาก checker ระบุ missing terminal ให้เทียบ Verilog model/blackbox, LEF PIN และ Liberty pin ของ cell เดียวกัน การใช้ model คนละ PDK revision เป็นสาเหตุสำคัญ อย่าเติม port ชื่อ vss แบบเดาสุ่มเพื่อให้ checker ผ่าน

PDN acceptance ต้องมีสองส่วน: connectivity และ geometry ตรวจ via cut, landing/enclosure, spacing และ ring/stripe connection ใน DEF/GDS/DRC การ connected ใน OpenROAD อย่างเดียวไม่เพียงพอสำหรับ physical sign-off

ผลส่ง: floorplan screenshots, PDN connectivity report และ via geometry evidence Acceptance: pad/core power ถูกโดเมน ไม่มี required pin ลอย และ geometry ผ่าน rule ของ PDK

## 16. Lab 9 — Placement, CTS และ timing closure

หลัง global/detailed placement ตรวจ utilization, congestion, buffer count และ setup slack หาก DMEM mux tree เป็น critical path ให้พิจารณา memory architecture ก่อนปรับ clock อย่างเดียว

CTS ต้องเริ่มจาก clock network หลัง input pad และครอบคลุม clock sinks ที่ใช้จริง ตรวจ insertion delay, skew, clock slew/capacitance และจำนวน clock buffers ตรวจ post-CTS hold เพราะ skew อาจเปลี่ยนเส้นทางที่เคยผ่านเป็น fail

แก้ setup: ลด combinational depth, เพิ่ม register ที่รักษาพฤติกรรม/latency, ปรับ memory interface หรือเปลี่ยน floorplan แก้ hold: ใช้ flow repair ที่ยืนยัน extracted timing ไม่แก้ด้วยการเพิ่ม uncertainty แบบสุ่ม

ทดลองหลาย clock periods ต้องแก้สองส่วนสอดคล้องกัน: `CLOCK_PERIOD`/SDC และ `CLOCK_FREQUENCY` ใน VHDL เพราะ UART baud generator และ SYSINFO อาศัยค่าความถี่นี้ หากเปลี่ยนเฉพาะ SDC โปรแกรม UART อาจผิด baud แม้ STA ผ่าน

ผลส่ง: pre/post CTS comparison, worst paths และ decision log Acceptance:ไม่มี setup/hold violations ตาม required corners และ constraints ที่อนุมัติ

## 17. Lab 10 — Routing, extraction และ finishing

หลัง global routing ตรวจ overflow/congestion และ routing layer limits ก่อน detailed routing หลัง detailed routing ตรวจ wire/via violations, antenna repair และ extracted RC methodology อ่านชื่อ corner ใน report ไม่สรุปจาก nominal เพียง corner เดียว

เปิดผลรัน:

```bash
make gui
```

`make gui` ใช้ OpenInKLayout ของ LibreLane และ `--last-run`; ถ้าใช้หลาย configuration ให้บันทึก run ที่ต้องการให้ชัดเจน อย่า review layout ของ run เก่าแล้วอ้างกับ log ใหม่

ตรวจ filler, density fill, bondpad, seal ring และ finishing ที่ flow/PDK กำหนด จากนั้นรัน checks ที่ถูก invalidate โดย finishing อีกครั้ง final GDS ต้องสัมพันธ์กับ final netlist และ extraction ที่อ้างใน sign-off

ผลส่ง: final GDS/LEF/DEF/netlist/SPEF/SDC และ logs ตาม flow ที่รันจริง Acceptance:ผล physical checks และ final STA มีหลักฐานครบ ห้ามนับแค่ข้อความ “flow completed” เป็น submission ready

## 18. Lab 11 — โปรแกรม C UART/GPIO

ตัวอย่าง C ใช้ NEORV32 software framework ที่รวมมา compiler ต้องรองรับ RV32I/ILP32 แม้ชื่อ prefix เป็น riscv64-unknown-elf- ให้ตรวจ multilib ก่อน:

```bash
riscv64-unknown-elf-gcc --version
riscv64-unknown-elf-gcc -print-multi-lib
make firmware-c
```

หาก compiler ใช้ prefix อื่น:

```bash
make firmware-c RISCV_PREFIX=riscv32-unknown-elf-
```

common.mk เป้าหมาย all จะ build และ install VHDL image กลับไป vendor RTL ให้ดู size report ว่า ROM ไม่เกิน 4 KiB และ RAM ไม่เกิน 1 KiB หาก link overflow ให้ประเมินโค้ด/หน่วยความจำก่อนเพิ่มขนาด ไม่ควรฝืน linker แล้วใช้ config เดิม

หลังเปลี่ยนเป็น C demo `make sim` ของ smoke จะไม่เห็น A5→5A ตามลำดับที่กำหนด เพราะ firmware เปลี่ยนแล้ว ให้สร้าง testbench สำหรับ UART banner/GPIO counter หรือกลับ smoke ก่อนทดสอบ pipeline:

```bash
make firmware-smoke
make sim
make sim-verilog
```

อย่ารัน firmware-smoke หลังติดตั้ง C image หากตั้งใจสร้างชิปที่มีโปรแกรม C เพราะจะทับโปรแกรมนั้น Firmware C build/UART banner ยังไม่ได้ตรวจในสภาพแวดล้อมจัดทำแพ็กเกจนี้

## 19. Lab 12 — SRAM macro integration สำหรับต่อยอด

baseline ไม่มี SRAM macro จึงไม่มี fake adapter ที่รับประกัน timing โดยไม่มีการทดสอบ การทำ IMEM/DMEM 4 KiB ด้วย RM_IHPSG13_1P_1024x32_c2_bm_bist ต้องพัฒนา adapter และ profile ใหม่อย่างเป็นขั้นตอน:

1. เริ่มจาก interface ใน neorv32_imem.vhd/neorv32_dmem.vhd ของ snapshot นี้: request address/data/byte-enable/strobe/read-write และ response data/ack/error
2. แปลง byte address เป็น word index ด้วย addr[11:2] สำหรับ 1024 words; ให้ decoder ตรวจ address range ก่อน truncate
3. ดู polarity ของ A_MEN/A_WEN/A_REN/A_BM จาก actual macro model ไม่เดาจากชื่อ pin แปลง byte enable เป็น bit mask 32 bitsตาม semantics ของ macro
4. ยืนยัน latency ของ read และ OUTREG ว่า response ack มาถึง cycle ที่ data valid ตรวจ read-after-write, back-to-back requests, request gating และ reset of handshake
5. ต่อ A_DLY/BIST signals ตาม specification; baseline template tie A_DLY=1 แต่ต้องยืนยันกับ macro revision
6. ทดสอบ adapter แบบ standalone ทั้ง word/halfword/byte, boundary address, bursts ที่ interface อนุญาต และ disabled access
7. เปลี่ยน memory source เป็น wrapper ที่ใช้ SRAM; สำหรับ mixed VHDL/Verilog อาจใช้ component boundary/blackbox ที่ GHDL รองรับ หรือย้าย adapter ฝั่ง Verilog พร้อมตรวจ hierarchy ใหม่
8. ใส่ MACROS views ได้แก่ GDS/LEF/Liberty/VH, placements, orientation และ PDN hooks ตามชื่อ instance จริงหลัง synthesis
9. ต่อ power pins ทั้ง logic/array rails ตาม physical views และตรวจ PDN via enclosure
10. ทดสอบ CPU firmware และ formal/GL checks ตาม scope แล้วจึง PNR ใหม่

ห้ามคัดลอก MACROS จาก template ตรง ๆ เพราะ instance `i_chip_core.sram_0` และ `sram_1` ไม่มีใน baseline นี้ การเพิ่ม SRAM ต้องเปลี่ยน memory architecture จริง ไม่ใช่ใส่ macro ที่ไม่ได้เชื่อมกับ CPU

หากต้องการ reprogrammable firmware ให้ใช้ UART bootloader/profile BOOT_MODE_SELECT=0 พร้อม IMEM writable RAM และ memory sizes ที่เพียงพอ ตรวจ binary upload protocol, boot ROM requirements และ CLINT/UART feature ที่ bootloader ใช้ ทั้ง SRAM adapter และ bootloader profile เป็นงานต่อยอด ไม่ได้อ้างว่ารันแล้วในแพ็กเกจนี้

## 20. วิเคราะห์ปัญหาที่พบบ่อย

| อาการ | ตรวจสาเหตุ | วิธีแก้และหลักฐาน |
|---|---|---|
| library neorv32 not found | file order, workdir, -P | ใช้ analyze.sh และอ่าน error แรก |
| GHDL ไม่มี Verilog output | binary version/backend | ใช้ build ที่รองรับ synthesis และทดสอบ export |
| ROM overflow | image_size_c กับ IMEM_SIZE | ลด firmware หรือเพิ่ม window/physical capacityอย่างสอดคล้อง |
| smoke timeout | boot mode, image, CPU feature, decoder | กลับ firmware-smoke แล้วตรวจ PC/bus/GPIO ใน waveform |
| exported simulation ผ่านแต่ technology fail | synth mapping, unsupported cell | ตรวจ mapped netlist/Liberty และ constraints |
| missing built LibreLane PDK config | PDK_ROOT ผิดหรือ raw source | ตรวจ libs.tech/librelane/config.tcl และ built views |
| PAD-0033 / missing terminal | pad RTL กับ LEF/model ไม่ตรง | เทียบ pins จาก PDK snapshot เดียวกัน |
| IOVDD/global connections error | cell/net name หรือ revision mismatch | อ่าน pin model และ global connection definitions ไม่ short โดเมน |
| set_global_connections wrong args | custom Tcl เก่า | ใช้ default PDN ของ version ที่ lock; port custom Tcl อย่างมีหลักฐาน |
| placement ไม่พอ | RAM mapped เป็น FF, area สูง | วัด technology area; ลด memory/เพิ่ม core/ใช้ macro |
| timing fail หลัง CTS | skew, insertion/hold paths | อ่าน path report และรัน targeted repair |
| DRC หลัง PDN | via landing/enclosure/spacing | ตรวจ geometry ด้วย PDK rules ไม่ใช้ connectivity แทน |
| UART baud ผิด | CLOCK_FREQUENCY ไม่ตรงจริง | แก้ clock generic และ SDC พร้อม rebuild |
| GUI psutil missing | package ของ GUI environment | ใช้ environment ที่ตรง toolchain; เป็นปัญหา GUI ไม่ใช่เหตุผลข้าม sign-off |

อย่าคัดลอกคำสั่ง skip Checker.YosysSynthChecks, DRC, LVS หรือ antenna เป็น release flow หากทดลอง skip ให้แยก run tag และระบุชัดว่า debug-only

## 21. Resume โดยรักษา provenance

ใช้ state_out.json จาก step ที่สำเร็จจริง และเลือก starting step ที่สัมพันธ์กับ state โดยตรวจ `librelane --help` และชื่อ steps ของ snapshot ก่อน ตัวอย่างรูปแบบ:

```bash
librelane --pdk ihp-sg13g2 --pdk-root "$PDK_ROOT" --manual-pdk \
  --jobs 1 --run-tag NEORV32_RESUME \
  --with-initial-state /absolute/path/to/state_out.json \
  --from OpenROAD.STAPrePNR librelane/config.yaml
```

คำสั่งนี้เป็นตัวอย่าง workflow ต้องเลือก path ของ run ตัวเอง ถ้าเปลี่ยน firmware/RTL ให้กลับ synthesis เพราะ netlist ใน state เก่าไม่ได้แทน source ใหม่ ถ้าเปลี่ยน pad/die/PDN ให้กลับ step ก่อนสร้างข้อมูลนั้น การใช้ state เก่าโดยไม่คำนึงถึง dependency ทำให้ reports กับ design ไม่ตรงกัน

## 22. Sign-off matrix และ release

กรอก docs/signoff.csv ด้วย PASS/FAIL/NOT_RUN/WAIVED พร้อม evidence path และ reviewer สำหรับ waiver ต้องระบุเหตุผลและ methodology ที่ยอมรับ อย่าใช้ waiver ของตัวอย่างแทนข้อกำหนดโรงงาน

ต้องตรวจ RTL functional, constraints, reset/CDC, technology synthesis, all required PVT/mode setup/hold, final parasitic extraction, IO/power connectivity, DRC, LVS, antenna, density, XOR และ package/ESD/submission rules ที่เกี่ยวข้อง เฉพาะ checks ที่ methodology กำหนดให้ใช้เท่านั้นจึงเป็น required gates แต่ต้องกำหนด methodology ก่อนสรุปสถานะ

```bash
make release
```

release.py สร้าง SHA-256 manifest ของ source/views และบอกสถานะ NOT_CERTIFIED เสมอ มันไม่ parse log แล้วประกาศ sign-off อัตโนมัติ และไม่ถือว่าการมีไฟล์ GDS แปลว่า checks ผ่าน ให้เก็บ run config, tool versions, PDK revision, netlist, SDC, SPEF, GDS, reports และ waivers ในชุดเดียวกัน

ให้ผู้เรียนอีกคนรัน smoke และ rebuild environment จากแพ็กเกจ ตรวจ hash ของ source ก่อนแก้ไข ผลสังเคราะห์/physical ที่ทำซ้ำอาจแตกต่างตาม tool/seed/resource ต้องเปรียบเทียบ acceptance/report ไม่สรุปจาก GDS byte hash อย่างเดียว

## 23. ลำดับ Lab และเกณฑ์ตรวจงาน

| Lab | ผลส่ง | Acceptance หลัก |
|---|---|---|
| 1 Source audit | architecture/pin table | source snapshot และ ports ชัดเจน |
| 2 VHDL smoke | log/waveform | A5→5A, DMEM checks ผ่าน |
| 3 Export | Verilog/log | exported simulation ผ่าน |
| 4 Structural check | Yosys report | hierarchy/check -assert ผ่าน |
| 5 Technology synthesis | area/netlist | mapped cells ถูกต้อง ไม่มี unresolved cell |
| 6 Pad ring | DEF/layout | pads match netlist/configครบ |
| 7 STA constraints | coverage/path report | ไม่มี unexpected unconstrained path |
| 8 PDN | connectivity/geometry | ทั้ง electrical และ physical gates ผ่าน |
| 9 Placement/CTS | setup/hold/skew | required corners ผ่าน |
| 10 Routing/finishing | final views/reports | DRC/LVS/antenna/density/STA มีหลักฐาน |
| 11 Firmware C | image/UART test | size/runtime/clockตรงกัน |
| 12 SRAM extension | adapter tests/PNR | latency, byte masks, views, PDNถูกต้อง |

คำถามสำคัญ: ทำไม memory mapped to FF จึงทำให้ area สูง? ทำไมแก้ CLOCK_PERIOD อย่างเดียวไม่พอสำหรับ UART? ทำไม connected PDN จึงยัง DRC fail ได้? ทำไม ROM firmware เปลี่ยนแล้วต้องเริ่ม synthesis ใหม่? ทำไม flow completed จึงต่างจาก foundry submission ready?

## 24. แหล่งอ้างอิงทางการ

- NEORV32 repository: https://github.com/stnolting/neorv32
- NEORV32 datasheet: https://stnolting.github.io/neorv32/
- NEORV32 user guide: https://stnolting.github.io/neorv32/ug/
- IHP template ที่ตรวจ source: https://github.com/IHP-GmbH/ihp-sg13-librelane-template
- IHP SG13G2 template path ในเอกสารปัจจุบัน: https://github.com/IHP-GmbH/ihp-sg13g2-librelane-template
- IHP full-chip flow: https://ihp-open-pdk-docs.readthedocs.io/en/latest/digital/librelane_full_chip.html
- IHP LibreLane setup: https://ihp-open-pdk-docs.readthedocs.io/en/latest/digital/librelane_setup.html
- LibreLane PDK requirements: https://librelane.readthedocs.io/en/latest/usage/about_pdks.html
- LibreLane Nix installation: https://librelane.readthedocs.io/en/latest/installation/nix_installation/index.html

อ้างอิง snapshot ที่กำหนดสำหรับ rebuild; หน้า latest ใช้เพื่ออ่านภาพรวม ไม่ใช่หลักฐานว่า API ของ lock เดิมเปลี่ยนไปแล้วได้โดยไม่ตรวจ
