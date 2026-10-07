# Lab 6 — Pad ring และ pin mapping

คู่มือ NEORV32 / LibreLane / IHP SG13G2 • ฉบับขยาย 6 ตุลาคม 2026

## 6.1 เป้าหมายและขอบเขต

Lab นี้นำ technology netlist จาก Lab 5 มาประกอบ pad ring และตรวจความสอดคล้องระหว่าง RTL, netlist, placement configuration, ODB/DEF และ bond plan ผู้เรียนต้องอธิบายได้ว่าขาภายนอกแต่ละขาเข้าหรือออกจาก NEORV32 ผ่าน IO cell ใด และใช้ power domain ใด

ผลลัพธ์คือ pad-ring checkpoint ที่ตรวจสอบได้ พร้อม pin-map CSV และหลักฐานการตรวจ ยังไม่ใช่ package pinout ที่อนุมัติผลิต และยังไม่ใช่ผลผ่าน DRC/LVS/ESD หรือ PDN sign-off

เนื้อหายึดคู่มือ NEORV32_Chip_Implementation_TH.md ที่มีอยู่: 24 functional/supply pad instances, signal pads 20 ตัว และ supply pads 4 ตัว ด้านละ 6 pads จำนวนนี้ไม่รวม corners, fillers, bondpads และ seal ring ที่เพิ่มใน physical flow ดังนั้นจำนวน COMPONENTS ทั้งหมดใน DEF จะมากกว่า 24

Snapshot อ้างอิง: NEORV32 af905649fdbe650245af700690e435b58601ef7e; template 40ca8b9d5e8b3f87b2a36d590517958df21ee241; LibreLane aaf7a938766e0708ca894776d81e2c2cfb12b7e7; PDK 22f43352dd8219f9007eb659e422e0d5fe28c5fb ใช้ API ของเครื่องมือที่ lock ไว้เป็นหลัก ไม่คัดลอกตัวเลือกจากหน้า latest โดยไม่ตรวจ

คำสั่งด้านล่างรันจาก project root หลังเข้า environment ของโครงการ ตัวอย่างชื่อ run ต้องเปลี่ยนให้ตรงกับผลรันจริง ในสภาพแวดล้อมจัดทำเอกสารนี้ไม่ได้รัน pad placement หรือ physical verification จึงไม่มีการอ้าง PASS ของ layout

## 6.2 ความรู้ก่อนเริ่ม

แยกวัตถุเหล่านี้ให้ชัด:

| วัตถุ | ตัวอย่าง | ความหมาย |
|---|---|---|
| Top-level port | input_PAD[8] | สัญญาณที่ chip_top เปิดออกภายนอก |
| IO-cell instance | inputs[8].input_pad | วงจรรับสัญญาณจาก pad เข้าสู่ core |
| IO-cell terminal | pad, p2c, c2p | ขาภายใน cell; ตรวจชื่อจาก PDK snapshot |
| Core net/port | input_in[8], uart_rxd_i | สัญญาณ logic หลังผ่าน IO cell |
| Bondpad | instance ที่ flow เพิ่ม | รูปทรงโลหะสำหรับ bonding ตาม methodology |
| Package pin | ยังไม่กำหนด | ขาบน package ซึ่งเชื่อมผ่าน bond wire/assembly |

เส้นทาง input คือ external port → IO pad terminal → receiver output → chip_core → NEORV32 ส่วน output คือ NEORV32 → chip_core → IO driver input → IO pad terminal → external port ตรวจ terminal name ของ cell จริงก่อนเขียน wrapper

Power pad ทำหน้าที่นำไฟเข้าสู่ supply network ไม่ใช่ data input และไม่ควรใส่ input/output timing delay แบบ GPIO การเชื่อมไฟใน netlist และการมีโลหะเชื่อมต่อทางกายภาพเป็นการตรวจคนละระดับ

## 6.3 ขั้นที่ 1 — เตรียม baseline และหลักฐาน

```bash
mkdir -p build/lab6
make doctor
librelane --version > build/lab6/librelane-version.txt
librelane --help > build/lab6/librelane-help.txt
sha256sum rtl/chip_top.sv rtl/chip_core.sv \
  librelane/config.yaml librelane/chip_top.sdc \
  > build/lab6/input-sha256.txt
```

ตรวจ Lab 5 ก่อนเดินต่อ: mapped netlist มี NEORV32 logic จริง, ไม่มี unresolved CPU module, IO pads ยังอยู่, และไม่มี structural check ที่ถูกข้าม บันทึก run/tag และ state ที่เป็นต้นทาง ถ้าเปลี่ยน RTL หรือ pad instances ต้องกลับ synthesis; เปลี่ยนเพียง physical ordering ต้องสร้าง pad ring ใหม่และรัน downstream steps ใหม่

อ่านไฟล์ที่เกี่ยวข้อง:

```bash
rg -n 'module|PAD|sg13g2|input_pad|output_pad|vdd_pad|vss_pad' rtl/chip_top.sv
rg -n 'input_in|output_out|gpio|uart|clk|rst' rtl/chip_core.sv
rg -n 'PAD_|EXTRA_LEFS|EXTRA_GDS|DIE_AREA|CORE_AREA|CLOCK_NET|VDD|VSS' \
  librelane/config.yaml
```

rg ใช้ค้นตำแหน่งให้เปิดอ่าน ไม่ใช่ HDL parser และไม่ใช่หลักฐานว่า hierarchy/check ผ่าน

## 6.4 ขั้นที่ 2 — สร้าง logical pin map

| Function | External port | Core mapping | ทิศทาง |
|---|---|---|---|
| Clock | clk_PAD | clk_i, 50 MHz | Input |
| Reset | rst_n_PAD | rstn_i, active-low | Input |
| GPIO input 0–7 | input_PAD[7:0] | gpio_i[7:0] | Input |
| UART RX | input_PAD[8] | uart_rxd_i | Input |
| GPIO output 0–7 | output_PAD[7:0] | gpio_o[7:0] | Output |
| UART TX | output_PAD[8] | uart_txd_o | Output |
| Core supply | supply pad instances | VDD / VSS | Power/Ground |
| IO supply | supply pad instances | IOVDD / IOVSS | Power/Ground |

ตรวจความกว้าง bus เป็น 9 bits ต่อ input/output wrapper โดย 8 bits เป็น GPIO และ bit 8 เป็น UART อย่าสลับ UART RX/TX ตามความหมายของขาบนอุปกรณ์อีกฝั่ง: RX ของชิปต้องรับจาก TX ของแหล่งสัญญาณภายนอก

ตรวจ bit ordering ทีละ bit: input_PAD[i] ต้องไป gpio_i[i]; gpio_o[i] ต้องไป output_PAD[i] ถ้าใช้ concatenation ให้เขียนตารางเทียบตำแหน่งซ้าย/ขวา อย่าอาศัยชื่อ bus เหมือนกันเป็นหลักฐานว่า bit order ถูก

## 6.5 ขั้นที่ 3 — กำหนด pad list ครบ 24 ตัว

ตารางนี้เป็นลำดับรายการใน configuration ไม่ใช่การยืนยันทิศเดินรอบ die หรือหมายเลข package

| ด้าน | ลำดับในรายการ | Instance | Function |
|---|---:|---|---|
| SOUTH | 1 | clk_pad | Clock |
| SOUTH | 2 | rst_n_pad | Reset |
| SOUTH | 3 | inputs[8].input_pad | UART RX |
| SOUTH | 4 | outputs[8].output_pad | UART TX |
| SOUTH | 5 | vdd_pads[0].vdd_pad | VDD |
| SOUTH | 6 | vss_pads[0].vss_pad | VSS |
| WEST | 1 | inputs[0].input_pad | GPIO input 0 |
| WEST | 2 | inputs[1].input_pad | GPIO input 1 |
| WEST | 3 | inputs[2].input_pad | GPIO input 2 |
| WEST | 4 | inputs[3].input_pad | GPIO input 3 |
| WEST | 5 | inputs[4].input_pad | GPIO input 4 |
| WEST | 6 | inputs[5].input_pad | GPIO input 5 |
| NORTH | 1 | inputs[6].input_pad | GPIO input 6 |
| NORTH | 2 | inputs[7].input_pad | GPIO input 7 |
| NORTH | 3 | outputs[0].output_pad | GPIO output 0 |
| NORTH | 4 | outputs[1].output_pad | GPIO output 1 |
| NORTH | 5 | iovdd_pads[0].iovdd_pad | IOVDD |
| NORTH | 6 | iovss_pads[0].iovss_pad | IOVSS |
| EAST | 1 | outputs[2].output_pad | GPIO output 2 |
| EAST | 2 | outputs[3].output_pad | GPIO output 3 |
| EAST | 3 | outputs[4].output_pad | GPIO output 4 |
| EAST | 4 | outputs[5].output_pad | GPIO output 5 |
| EAST | 5 | outputs[6].output_pad | GPIO output 6 |
| EAST | 6 | outputs[7].output_pad | GPIO output 7 |

เขียน pin-map CSV โดยมีคอลัมน์ function, external_port, core_port, direction, instance, master, domain, side, list_index, x_um, y_um, orientation, bondpad_instance, package_pin, evidence เริ่ม x/y/orientation/package_pin เป็นช่องว่าง แล้วเติมค่าจาก layout จริง ห้ามเติมหมายเลข package จาก list_index

Acceptance ของตาราง: unique instances 24, input signal pads 11 (clock/reset/9 inputs), output signal pads 9, power pads 4 และทุกสัญญาณที่ active มีเส้นทางครบ

## 6.6 ขั้นที่ 4 — ตรวจ views และ supply terminals ของ PDK

```bash
: "${PDK_ROOT:?Set PDK_ROOT to the parent of ihp-sg13g2}"
pdk_dir="$PDK_ROOT/ihp-sg13g2"
rg --files "$pdk_dir/libs.ref/sg13g2_io" > build/lab6/io-view-files.txt
rg --files ip > build/lab6/project-ip-files.txt
```

ตรวจ IO cell แต่ละ master ให้สัมพันธ์กันใน Verilog/blackbox, Liberty, LEF, GDS และ CDL/SPICE ที่ใช้ LVS โดยชื่อ master/terminal ต้องเข้ากันและมาจาก revision เดียวกัน LEF ให้ geometry และ physical pins, Liberty ให้ timing/electrical model, Verilog ให้ interface/functional behavior, GDS ให้ geometry จริง และ LVS views ให้วงจรที่นำมาเทียบ

Supply connection matrix ต้องสร้างจาก cell ที่ใช้จริง:

| Net | หน้าที่ | สิ่งที่ต้องตรวจ |
|---|---|---|
| VDD | Core positive supply | Core power pad, standard-cell PG, IO core-side terminals ที่มีจริง |
| VSS | Core return | Core ground pad, standard-cell ground, IO ground terminals ที่มีจริง |
| IOVDD | IO positive supply | IO power pad และ IO-domain supply rail |
| IOVSS | IO return | IO ground pad และ IO-domain ground rail |

ชื่อ pin ตัวพิมพ์เล็ก เช่น vdd/iovdd และชื่อ net ตัวพิมพ์ใหญ่ เช่น VDD/IOVDD เป็นคนละสิ่ง ต้องมี mapping ที่ถูกต้อง ขา PG ใน wrapper อาจอยู่ใต้ USE_POWER_PINS ให้ตรวจ define ของแต่ละขั้น ไม่ใส่ทุก terminal ลงทุก cell ด้วยการเดา

IOVDD และ VDD ต้องเก็บเป็น nets แยกตาม baseline ค่าแรงดันและการผูก grounds ต้องอ้าง specification ของ PDK/board/package ไม่รวม domains เพียงเพื่อแก้ checker error จำนวนหนึ่งคู่ต่อ domain เป็น baseline สำหรับฝึก flow ต้องคำนวณจำนวน power pads ตามกระแส DC/transient, simultaneous switching, pad limits, bonding และ IR drop เมื่อออกแบบชิปผลิตจริง

## 6.7 ขั้นที่ 5 — ตรวจชื่อ instance ใน mapped netlist

```bash
synth_run=librelane/runs/NEORV32_SYNTH_01
rg --files "$synth_run" -g '*.v' -g '*state*.json'
rg -n 'clk_pad|rst_n_pad|input_pad|output_pad|iovdd_pad|iovss_pad|vdd_pad|vss_pad' \
  "$synth_run" -g '*.v'
```

เลือก netlist ที่ state_out.json ของ Yosys.Synthesis อ้างจริง ไม่เลือกไฟล์แรกจากผลค้นหา ถ้า synthesis หยุดก่อน checker steps ให้ตรวจ reports จาก Lab 5 และตรวจว่า flow ของ Lab 6 รัน checker เหล่านั้นต่อ

HDL อาจแสดง escaped identifier เช่น `\inputs[0].input_pad ` โดย backslash และ whitespace เป็นไวยากรณ์ Verilog ชื่อ logical instance ใน ODB อาจแสดง `inputs[0].input_pad` จึงต้องเทียบชื่อที่ผ่าน parser แล้ว ไม่ลบ backslash ทั้งไฟล์ด้วย sed

สร้าง set ของ expected pad names จากตาราง และเทียบกับ actual names: expected − actual ต้องว่าง และ actual functional/supply pads − expected ต้องว่าง Corner/filler/bondpad ที่ physical flow เพิ่มให้ตรวจเป็นอีกกลุ่ม

## 6.8 ขั้นที่ 6 — ตรวจ PAD_* configuration และ escaping

LibreLane มี OpenROAD.PadRing สำหรับประกอบ pad ring บน floorplanned ODB และ PAD_SOUTH/PAD_EAST/PAD_NORTH/PAD_WEST สำหรับรายชื่อ instance ต่อด้าน ใช้ config ที่ให้มากับ project เป็นจุดตั้งต้น

ตัวอย่างรูปแบบอ้างอิงสำหรับ snapshot ของโครงการ:

```yaml
PAD_SOUTH:
  - 'clk_pad'
  - 'rst_n_pad'
  - 'inputs\[8\].input_pad'
  - 'outputs\[8\].output_pad'
  - 'vdd_pads\[0\].vdd_pad'
  - 'vss_pads\[0\].vss_pad'

PAD_WEST:
  - 'inputs\[0\].input_pad'
  - 'inputs\[1\].input_pad'
  - 'inputs\[2\].input_pad'
  - 'inputs\[3\].input_pad'
  - 'inputs\[4\].input_pad'
  - 'inputs\[5\].input_pad'

PAD_NORTH:
  - 'inputs\[6\].input_pad'
  - 'inputs\[7\].input_pad'
  - 'outputs\[0\].output_pad'
  - 'outputs\[1\].output_pad'
  - 'iovdd_pads\[0\].iovdd_pad'
  - 'iovss_pads\[0\].iovss_pad'

PAD_EAST:
  - 'outputs\[2\].output_pad'
  - 'outputs\[3\].output_pad'
  - 'outputs\[4\].output_pad'
  - 'outputs\[5\].output_pad'
  - 'outputs\[6\].output_pad'
  - 'outputs\[7\].output_pad'
```

Single quotes เก็บ backslash ตามตัวอักษร ส่วน double quotes มี YAML escape rules ของตัวเอง ตัวอย่างนี้รักษารูปแบบ bracket escaping ของคู่มือเดิม แต่ต้องตรวจว่า pad script ของ locked version ตีความเป็น pattern/string อย่างไร ถ้าเปลี่ยน version แล้วชื่อ match ไม่ได้ ให้ตรวจ parsed config และ Tcl ที่ flow สร้าง ไม่เพิ่ม backslash โดยลองผิดลองถูก จุดใน pattern อาจมีความหมายพิเศษด้วย ถ้า API ใช้ regex ต้องตรวจ exact-match semantics ของ API นั้น

ตรวจ PAD_CFG ถ้ามี custom script เพราะ script อาจเปลี่ยนวิธีใช้ PAD_* และห้ามสร้าง config key ซ้ำใน YAML ขณะ merge ตัวอย่างนี้ รายชื่อ pads ต้องสัมพันธ์กับ RTL และ mapped netlist ทุกครั้ง

Bondpad ให้ใช้ EXTRA_LEFS/EXTRA_GDS และ PAD_BONDPAD_NAME ตาม project/template snapshot ตรวจไฟล์มีอยู่และ master names ตรงกัน ไม่ใส่ bondpad เป็น logic replacement หรือใช้ LEF/GDS คนละ geometry

## 6.9 ขั้นที่ 7 — ตรวจ die/core และงบพื้นที่รอบ die

Baseline DIE_AREA เป็น (0,0)–(1600,1600) µm และ CORE_AREA เป็น (365,365)–(1235,1235) µm จึงมี core 870×870 µm หรือ 756,900 µm² และขอบจาก die ถึง nominal core ด้านละ 365 µm

พื้นที่ขอบ 365 µm ไม่ใช่ pad clearance ที่ใช้ได้ทั้งหมด ต้องหัก IO depth, ring routing, keepout, corner geometry และ seal-ring methodology ส่วนพื้นที่ข้างแต่ละด้านต้องตรวจด้วยขนาด pad master จาก LEF จริง

ประเมินความยาวขั้นต่ำ:

L_required = corner reservation ทั้งสองปลาย + ผลรวม pad widths หลัง orientation + ผลรวม gaps/fillers + spacing ที่ methodology กำหนด

อย่าใช้ 1600/6 เป็น pitch ที่อนุมัติ เพราะ pad order algorithm, corner offsets และ bond spacing ทำให้ตำแหน่งจริงต่างได้ ตรวจ transformed bounding boxes หลัง rotation ไม่ใช้ width ก่อน rotation สำหรับทุกด้าน

หาก pads ไม่พอพื้นที่ ให้เพิ่ม die/core margins หรือทบทวน floorplan อย่างมีหลักฐาน การลด spacing ต่ำกว่าข้อกำหนดหรือย้าย pads เข้า core rows เพื่อให้ placer ผ่านไม่เป็นวิธีแก้ที่ยอมรับได้

## 6.10 ขั้นที่ 8 — รันถึง pad-ring checkpoint

ตรวจจาก help ว่า installed CLI รองรับ --flow, --to, --run-tag, --manual-pdk และ --jobs แล้วตรวจว่า Chip flow มี OpenROAD.PadRing จริง หาก locked flow ใช้ step name ต่าง ให้ใช้ชื่อจาก flow registration/log ของ environment นั้น

ตัวอย่างเมื่อ API ตรงตามเงื่อนไขข้างต้น:

```bash
set -o pipefail
librelane \
  --manual-pdk \
  --pdk ihp-sg13g2 \
  --pdk-root "$PDK_ROOT" \
  --flow Chip \
  --jobs 1 \
  --run-tag NEORV32_PADRING_01 \
  --to OpenROAD.PadRing \
  librelane/config.yaml \
  2>&1 | tee build/lab6/padring-run.log
```

รันจากต้น flow ในการตรวจครั้งแรกเพื่อให้ source/config ของ checkpoint เป็นชุดเดียวกัน ห้ามเพิ่ม --skip Checker.DisconnectedPins หรือข้าม synthesis checks เพื่อให้ checkpoint ดูผ่าน

```bash
pad_run=librelane/runs/NEORV32_PADRING_01
rg -n 'PadRing|PAD-|ERROR|WARNING|global.connection|terminal|disconnected' \
  "$pad_run" -g '*.log'
rg --files "$pad_run" -g '*.odb' -g '*.def' -g '*state*.json'
```

ผลคาดหวัง: pad-ring step จบโดยไม่มี immediate error และ state อ้าง ODB/DEF ที่สร้างได้จริง อ่าน deferred errors/checkers ด้วย การหยุดหลัง PadRing อาจยังไม่ได้สร้าง full PDN หรือรัน downstream checker จึงบันทึกขั้นเหล่านั้นเป็น NOT_RUN

## 6.11 ขั้นที่ 9 — ตรวจ physical inventory จาก ODB

วิธีที่ตรวจชื่อได้ชัดคืออ่าน ODB ของ pad-ring checkpoint แล้ว export รายชื่อ instance, master, placement status, orientation และ bounding box ถ้ามี Python OpenDB binding ใน environment ให้บันทึกสคริปต์นี้เป็น build/lab6/odb_inventory.py:

```python
import csv
import sys
import odb

db = odb.dbDatabase.create()
odb.read_db(db, sys.argv[1])
block = db.getChip().getBlock()
dbu = block.getDbUnitsPerMicron()
with open(sys.argv[2], 'w', newline='') as f:
    writer = csv.writer(f)
    writer.writerow(['instance', 'master', 'status', 'orientation',
                     'xmin_um', 'ymin_um', 'xmax_um', 'ymax_um'])
    for inst in sorted(block.getInsts(), key=lambda i: i.getName()):
        box = inst.getBBox()
        writer.writerow([inst.getName(), inst.getMaster().getName(),
                         str(inst.getPlacementStatus()), str(inst.getOrient()),
                         box.xMin()/dbu, box.yMin()/dbu,
                         box.xMax()/dbu, box.yMax()/dbu])
```

เรียกด้วย Python ที่มี odb และ ODB path จาก state จริง สคริปต์เป็นตัวอย่าง audit สำหรับ OpenDB API; ต้องยืนยัน bindings ของเครื่องก่อนใช้ ไม่ได้ทดสอบกับ PDK/ODB ในการจัดทำบทนี้ หากไม่มี odb Python ใช้ OpenROAD GUI หรือ Tcl API ใน environment เดิม ไม่จำเป็นต้องติดตั้ง odb จาก PyPI ซึ่งอาจเป็นคนละแพ็กเกจ

ตรวจ inventory: expected pad instances ทั้ง 24 มีอยู่ครั้งเดียว; masters ตรง views; placed/fixed ตาม pad-flow methodology; bounding boxes อยู่ใน die; corners/fillers ไม่ overlap; และ signal side terminals หันเข้าหา core ส่วน bonding side หันออกตาม geometry ของ master

สำหรับ DEF ตรวจ UNITS DISTANCE MICRONS ก่อนแปลงพิกัด และตรวจ COMPONENTS กับ NETS/SPECIALNETS แยกกัน BTerm ที่ไม่มีตำแหน่งโลหะใน checkpoint แรกอาจเป็นงานที่ flow สร้างภายหลัง ต้องตรวจ step methodology ก่อนตัดสิน fail แต่ต้องไม่ปล่อยให้ missing terminal ที่จำเป็นหายไปจากรายงาน

## 6.12 ขั้นที่ 10 — ตรวจ layout และลำดับรอบ die

เปิด ODB checkpoint ที่ต้องการใน OpenROAD พร้อม LEF/library context ของ run; ใช้ GUI mechanism ของ installed LibreLane หากใช้ make gui ให้ตรวจว่ามันเลือก --last-run และไม่ได้เปิด run อื่น

ตรวจตามลำดับ:

1. เปิดทั้ง die และ core ตรวจ dimensions และไม่มี pad ทับ rows
2. เปิด instance labels และทำเครื่องหมาย SOUTH/WEST/NORTH/EAST
3. คลิก clock/reset/UART/supply pads ตรวจชื่อ master และ instance
4. ตรวจ GPIO ทุก bit โดยไม่ใช้การนับจากมุมภาพอย่างเดียว
5. ตรวจ orientation กับ terminal geometry ไม่กำหนด R0/R90 แบบเดียวกันทุก master
6. ตรวจ corners/fillers ว่าปิด IO rows และ supply-abutment ตาม design rules
7. ตรวจ bondpad ถ้าสร้างแล้ว; ถ้ายังไม่สร้างให้บันทึก pending stage
8. บันทึก screenshots ทั้ง die และภาพขยายของทั้งสี่ด้าน

จาก inventory ให้ sort SOUTH/NORTH ตาม x และ WEST/EAST ตาม y แล้วเปรียบเทียบกับ PAD_* ว่าลำดับทางกายภาพตรงหรือกลับกัน บันทึกทิศจริง เช่น x เพิ่ม หรือ y ลด ไม่ใช้คำว่า clockwise โดยไม่มีจุดเริ่มต้น

สร้าง bond plan ระบุ top view ของ die, จุด origin, die orientation, ตำแหน่งเริ่มนับ, package orientation และการมองจากด้านบน/ล่าง หมายเลข package ต้องเติมหลังเลือก package และทบทวนการ bonding; package 24 ขาไม่ได้ตามมาจากจำนวน pad cells โดยอัตโนมัติ

## 6.13 ขั้นที่ 11 — ตรวจ logical wiring และ polarity

```bash
set -o pipefail
make sim-chip-wiring 2>&1 | tee build/lab6/chip-wiring.log
```

เป้าหมายเดิมใช้ behavioral pad mocks จึงตรวจ wrapper connectivity เท่านั้น ไม่ยืนยัน PDK electrical behavior, ESD หรือ layout และ mocks ต้องไม่อยู่ใน VERILOG_FILES ที่ใช้ implementation

เพิ่ม directed checks ตามความสามารถของ testbench:

| Test | Stimulus | สิ่งที่ต้องเห็น |
|---|---|---|
| Clock | Toggle clk_PAD | Clock หลัง input pad toggle โดยไม่ invert ผิด |
| Reset | Assert/deassert rst_n_PAD | Core reset polarity ตรง RTL |
| GPIO inputs | Walking-one 01,02,...,80 | bit i เข้าถึง core input bit i |
| GPIO outputs | Firmware A5 แล้ว 5A | External output bits ตรงลำดับ |
| UART RX | Idle high และ pattern | input bit 8 ไป UART RX |
| UART TX | Core-side transition/test firmware | UART TX ไป output bit 8 |

Smoke firmware A5→5A ตรวจ output wiring ได้บางส่วน แต่ไม่พิสูจน์ GPIO input หรือ UART protocol ต้องเพิ่ม test ที่เข้าถึงเส้นทางเหล่านั้นจริง ถ้า force core-side signal เพื่อทดสอบ output ให้อธิบายว่าเป็น wiring test ไม่ใช่ CPU/UART functional validation

Reset assertion/deassertion ผ่าน pad ไม่ถือว่าผ่าน RDC หรือ recovery/removal analysis ให้ส่งต่อ Lab 7 เพื่อตรวจ clock/reset constraints ส่วน clock ควรสร้างที่ external clk_PAD เพื่อรวม receiver delay ตาม methodology ของโครงการ

## 6.14 ขั้นที่ 12 — ตรวจ power associations และ bondpad

อ่าน ITerm-to-net association ใน ODB สำหรับ IO cell แต่ละตัว เปรียบเทียบกับ matrix จาก PDK views: ทุก required PG terminal ต้องผูก net ที่ถูก domain และไม่มี data terminal ต่อ supply ผิด ถ้า rail geometry ยังไม่สร้าง การมี association ไม่ใช่ physical continuity

Bondpad audit ต้องตรวจ instance master, จำนวนเทียบกับ intended bond sites, transformed coordinates, opening/passivation, overlap/offset กับ IO pad metal, layer stack และ spacing ตาม assembly rules ถ้า flow สร้าง bondpads ใน finishing stage หลัง checkpoint นี้ ให้บันทึก NOT_RUN และยกไปเป็น required gate ของ stage นั้น

แยกสถานะสามระดับ: logical wiring checked; pad placement checked; physical power/bond/ESD validation pending หรือ passed พร้อม evidence ไม่ใช้ภาพ ring ต่อเนื่องเป็นหลักฐานว่ามี current path ถูกต้อง

## 6.15 วิเคราะห์ปัญหาที่พบบ่อย

| อาการ | ตรวจอะไร | แนวแก้ |
|---|---|---|
| Pad instance not found | Parsed YAML, escaped identifiers, mapped netlist | ทำชื่อให้ match API ของ locked flow |
| PAD-0033/missing block terminal | Exact error context, top ports, IO .pad connection, BTerms | แก้ port/net/view ที่ขาดตามรายงาน |
| add_global_connections failure | Cell master และ LEF/Verilog/Liberty PG pins | ใช้ views ชุดเดียวกันและ map pin/net ให้ตรง |
| Output cell ไม่มี vss ใน view หนึ่ง | Revision และ conditional power defines | ตรวจ cell จริงก่อนแก้ interface; ห้ามเติมขาแบบเดา |
| Pads หายหลัง synthesis | Hierarchy, optimization และ source inclusion | แก้ wrapper/connectivity และรัน synthesis ใหม่ |
| Pads อยู่ผิดด้าน | PAD_* lists และ PAD_CFG | เทียบชื่อกับ physical inventory |
| GPIO bits กลับลำดับ | Bus concatenation และ walking-one test | แก้ bit mapping ไม่แก้ labels อย่างเดียว |
| UART TX อยู่บน RX bond site | Function map กับ instance/port | แก้ mapping ทั้ง RTL/config/bond plan |
| Bondpad GDS missing | EXTRA_GDS/EXTRA_LEFS/master names | ใช้ views ของ bondpad ที่เลือกจริง |
| Ring ดูสมบูรณ์แต่ pins disconnected | ITerm associations และ metal/vias | แก้ logic/PG connections แล้วตรวจ geometry ใน Lab 8 |
| ผลใหม่กับภาพเก่าปะปน | Run tag/state/file timestamps/hashes | ใช้ evidence จาก checkpoint ชุดเดียวกัน |

เมื่อเปลี่ยน pad master, RTL ports หรือ power interface ให้กลับ synthesis เมื่อเปลี่ยน ordering/die/core ให้กลับก่อน pad/floorplan data ที่เปลี่ยน และเมื่อเปลี่ยน bondpad geometry ให้รัน finishing/physical checks ที่ได้รับผลใหม่ทั้งหมด

## 6.16 ผลส่งและ acceptance

| ผลส่ง | รายละเอียดขั้นต่ำ |
|---|---|
| pin_map.csv | Functional map + instances + masters + physical coordinates/orientation + package status |
| Pad checkpoint | ODB/DEF/state_out.json และ source/config hashes |
| inventory.csv | Actual physical instance list จาก checkpoint |
| Wiring report | Tests ที่รันจริง, PASS/FAIL/NOT_RUN และ limitations |
| Power matrix | Expected PG pins/domains กับ actual associations |
| Layout evidence | Whole-die และ zoom ทุกด้านพร้อม run tag |
| Bond-plan draft | View direction/origin/order/package TBD ชัดเจน |
| Review checklist | ผู้ตรวจ, วันที่, evidence และ unresolved items |

Lab 6 ผ่านเมื่อ: 24 expected functional/supply instances ครบและไม่ซ้ำ; core/port mapping ตรงทุก bit; PAD_* match actual instances; placement/orientation/bounds ถูก; required terminal associations ถูก; directed wiring checks ที่กำหนดผ่าน; และ pending downstream gates ระบุชัด

หาก pad-ring step ยังไม่สำเร็จให้ FAIL หากยังไม่รันทดสอบให้ NOT_RUN การมี DEF/ODB อย่างเดียวไม่เป็น PASS ข้อจำกัด power-pad count, package/ESD, final bondpad geometry และ PDN ต้องมีผู้รับผิดชอบใน next-stage checklist

ส่งต่อ Lab 7: clock/reset pad path และ external-port names ที่ยืนยันแล้ว ส่งต่อ Lab 8: four-net domain matrix, pad-ring checkpoint และพื้นที่สำหรับ PDN รายการที่ยังไม่ตรวจต้องตามไปกับ handoff

## 6.17 แบบฝึกหัดและคำถามทบทวน

1. ย้าย UART pads ไปอีกด้าน: ไฟล์ใดต้องเปลี่ยนและ checkpoints ใดใช้ต่อไม่ได้?
2. ลบ input pad ออกจาก PAD_WEST หนึ่งตัว: flow ตรวจพบเมื่อใด และเหตุใดต้องมี independent inventory?
3. ทำ GPIO bus concatenation กลับลำดับ: ทำไม synthesis จึงอาจผ่านแต่ walking-one test fail?
4. เปรียบเทียบ top view กับ bottom view ของ bond plan: pin numbering เปลี่ยนการมองอย่างไร?
5. เหตุใด VDD/IOVDD แยก nets และ connected rail ยังไม่ยืนยัน ESD/latch-up compliance?
6. ความต่างระหว่าง pad-cell origin, pad terminal center และ bondpad center คืออะไร?
7. เหตุใด mock-pad simulation และ final powered-netlist/PDK simulation จึงครอบคลุมต่างกัน?
8. เมื่อเปลี่ยน output drive strength ต้องทบทวน timing/load, power, package และ geometry อะไรบ้าง?

## แหล่งอ้างอิง

- คู่มือโครงการ NEORV32_Chip_Implementation_TH.md: baseline mapping/snapshots/configuration
- IHP full-chip guide: https://ihp-open-pdk-docs.readthedocs.io/en/latest/digital/librelane_full_chip.html
- IHP SG13G2 template: https://github.com/IHP-GmbH/ihp-sg13g2-librelane-template
- LibreLane Pad Ring configuration: https://librelane.readthedocs.io/en/latest/reference/step_config_vars.html#pad-ring-generation

หน้าเอกสาร latest ใช้ตรวจแนวคิด/API ประกอบ ส่วนการรันให้ยึด source ของ dependency snapshot ที่โครงการ lock ไว้
