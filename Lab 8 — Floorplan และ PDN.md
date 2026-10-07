## Lab 8 — Floorplan และ PDN

Lab นี้นำ netlist ที่ผ่าน technology synthesis ใน Lab 5, pad ring และ pin mapping จาก Lab 6 และ timing constraints จาก Lab 7 มาสร้างโครงร่างทางกายภาพของ **NEORV32 full-chip บน IHP SG13G2** พร้อมระบบกระจายไฟเลี้ยงหรือ Power Distribution Network: PDN

ผลลัพธ์ที่ต้องได้คือ floorplan ที่มีพื้นที่เพียงพอสำหรับ placement, CTS และ routing รวมถึง PDN ที่เชื่อมต่อถูกต้องและมี geometry ผ่านการตรวจตามขอบเขตของ Lab

> **หลักการตรวจรับ:** การสร้าง PDN สำเร็จหรือการตรวจ connectivity ผ่าน ไม่ได้ยืนยันว่า via enclosure, metal spacing และความสามารถในการจ่ายกระแสผ่านทั้งหมด ต้องเก็บหลักฐานแยกกัน

ค่าพื้นที่และชื่อ net ในตัวอย่างต่อไปนี้เป็น **ตัวอย่างสำหรับการฝึก** ผู้เรียนต้องแทนด้วยค่าของโครงการจริง โดยเฉพาะชื่อ power pin ของ IO และ SRAM

### 8.1 วัตถุประสงค์

เมื่อจบ Lab ผู้เรียนสามารถ:

1. คำนวณพื้นที่ core จากผล synthesis และพื้นที่ที่ต้องสำรอง
2. กำหนด die area, core area และพื้นที่ระหว่าง core กับ pad ring
3. ตรวจ row, site, routing track และตำแหน่ง fixed instances
4. วางแผน power domains และจับคู่ชื่อ power nets กับ physical pins
5. สร้าง PDN โดยใช้ configuration ของ IHP template
6. ตรวจ connectivity, via geometry และความสอดคล้องระหว่าง ODB, DEF และ GDS
7. เปรียบเทียบ floorplan หลายแบบและเลือกแบบที่มีหลักฐานรองรับ

### 8.2 ไฟล์นำเข้าและเงื่อนไขก่อนเริ่ม

| รายการ | สิ่งที่ต้องตรวจ |
|---|---|
| Synthesized netlist | ไม่มี unresolved module หรือ black box ที่ไม่ตั้งใจ |
| Synthesis report | มี mapped standard-cell area และ cell count |
| Top-level wrapper | ชื่อ pad instances ตรงกับ Lab 6 |
| SDC | clock/reset และ I/O constraints ผ่านการตรวจจาก Lab 7 |
| Standard-cell LEF | มี site, pin และ obstruction ถูกต้อง |
| IO LEF | มี supply pins และตำแหน่ง physical terminals |
| SRAM LEF ถ้ามี | ขนาด, orientation, supply pins และ signal pins ครบ |
| GDS views | ตรงกับ revision ของ LEF และ netlist models |
| LibreLane configuration | ใช้ flow ของ IHP template ที่โครงการเลือกไว้ |
| PDK revision | บันทึก revision ที่ใช้จริง |

IHP full-chip template มีขั้นตอนประกอบ pad ring และทำ physical implementation รวมถึงงาน finishing และ verification ดังนั้น Lab นี้ควรปรับจาก configuration ของ template เดิม เพื่อรักษาขั้นตอนเฉพาะของ full-chip flow ไว้ครบถ้วน [IHP OpenPDK 11181f4 documentation](https://ihp-open-pdk-docs.readthedocs.io/en/latest/digital/librelane_full_chip.html?utm_source=chatgpt.com)

**Checkpoint 8-A:** ยังไม่เริ่ม floorplan ถ้า LEF ของ macro หรือ supply-pin mapping ยังไม่ครบ เพราะปัญหาเหล่านี้ทำให้การออกแบบพื้นที่และ PDN ผิดตั้งแต่ต้น

---

### 8.3 ขั้นที่ 1 — บันทึก baseline และเตรียมพื้นที่เก็บผล

ทำงานจาก root ของโครงการภายใน environment ที่ใช้รัน LibreLane

```bash
mkdir -p reports/lab8
mkdir -p results/lab8
mkdir -p scripts/lab8

git rev-parse HEAD > reports/lab8/project_commit.txt

librelane --version > reports/lab8/librelane_version.txt 2>&1
openroad -version > reports/lab8/openroad_version.txt 2>&1
librelane --help > reports/lab8/librelane_help.txt 2>&1

cp librelane/config.yaml reports/lab8/config_baseline.yaml
```

ถ้า PDK อยู่ใน Git checkout ให้บันทึก commit ด้วย เช่น:

```bash
git -C "$PDK_ROOT/ihp-sg13g2" rev-parse HEAD \
  > reports/lab8/pdk_commit.txt
```

ถ้า PDK ถูกติดตั้งผ่าน package manager และไม่มี `.git` ให้บันทึก package revision หรือ manifest ที่ใช้ติดตั้งแทน

สร้างตาราง baseline:

| พารามิเตอร์ | ค่าจริงของโครงการ |
|---|---|
| Top module | … |
| Standard-cell area | … µm² |
| Standard-cell count | … |
| Memory implementation | Flops / hard SRAM |
| Hard-macro area | … µm² |
| จำนวน pads แต่ละด้าน | … |
| Clock period | … ns |
| Core supply net | … |
| Core ground net | … |
| IO supply net | … |
| IO ground net | … |

**ผลที่คาดหวัง:** สามารถระบุได้ว่าผล floorplan แต่ละรอบสร้างจาก netlist, configuration และ environment ชุดใด

### 8.4 ขั้นที่ 2 — ตรวจว่า memory ถูก implement แบบใด

สำหรับ NEORV32 ให้ตรวจ instruction memory, data memory และ boot ROM จาก synthesis report

ค้นหาใน run ที่ผ่าน Lab 5:

```bash
rg -n -i \
  'memory|sram|ram|rom|flip.flop|cell area|chip area|blackbox' \
  librelane/runs/<LAB5_RUN>
```

แบ่งกรณีดังนี้:

| Implementation | ผลต่อ floorplan |
|---|---|
| Memory mapped เป็น flops/muxes | รวมอยู่ใน standard-cell area และอาจสร้าง congestion สูง |
| Hard SRAM macro | ต้องสำรองพื้นที่ macro, halo, pin access และ PDN connection |
| ROM mapped เป็น logic | ใช้พื้นที่ standard cells ตามผล mapping |
| Memory เหลือเป็น black box โดยไม่มี physical view | ยังทำ physical implementation ต่อไม่ได้ |

ห้ามคำนวณ core จากจำนวน logic cells ของ CPU เพียงอย่างเดียว เพราะ memory ที่ถูกแปลงเป็น flops อาจเป็นส่วนที่ใช้พื้นที่มากที่สุด

**Checkpoint 8-B:** อธิบายได้ว่า memory ทุกก้อนใน netlist จะปรากฏใน layout เป็นอะไร

### 8.5 ขั้นที่ 3 — คำนวณพื้นที่ core เบื้องต้น

กำหนด:

- $$A_{\text{std}}$$: พื้นที่ mapped standard cells
- $$U_{\text{target}}$$: utilization เป้าหมายในรูปอัตราส่วน
- $$A_{\text{macro}}$$: พื้นที่ hard macros
- $$A_{\text{reserved}}$$: พื้นที่สำหรับ halo, channel และบริเวณที่ห้ามวาง cells

สำหรับแบบที่ไม่มี hard macro:

$$A_{\text{core,initial}}
\approx
\frac{A_{\text{std}}}{U_{\text{target}}}$$

สำหรับแบบที่มี macros:

$$A_{\text{core,initial}}
\approx
\frac{A_{\text{std}}}{U_{\text{target}}}
+
A_{\text{macro}}
+
A_{\text{reserved}}$$

สมการนี้ใช้ประมาณขนาดเริ่มต้น หลังสร้าง rows จริงต้องตรวจพื้นที่ที่วาง standard cells ได้อีกครั้ง

**ตัวอย่างการคำนวณ**

สมมติ mapped standard-cell area เท่ากับ $$180{,}000\ \mu m^2$$ และเลือก utilization เริ่มต้น 35%:

$$A_{\text{core}}
=
\frac{180{,}000}{0.35}
\approx
514{,}286\ \mu m^2$$

ถ้าใช้ core สี่เหลี่ยมจัตุรัส:

$$W_{\text{core}}=H_{\text{core}}
\approx
\sqrt{514{,}286}
\approx717.1\ \mu m$$

อาจเลือกพื้นที่เริ่มต้นประมาณ $$740\times740\ \mu m$$ แล้วตรวจการปรับเข้ากับ site grid จาก ODB

**ข้อควรพิจารณา**

- ต้องเหลือพื้นที่สำหรับ clock buffers และ timing repair cells
- utilization ต่ำไม่ได้รับประกัน routing สำเร็จ ถ้า pin access หรือช่องระหว่าง macros แคบ
- การเพิ่ม core size อาจทำให้ wire length และ clock-tree length เพิ่มขึ้น
- สำหรับ full-chip ขนาด die อาจถูกกำหนดโดยจำนวน pads มากกว่าขนาด logic

### 8.6 ขั้นที่ 4 — ตรวจข้อจำกัดจาก pad ring

นำ pad mapping จาก Lab 6 มาตรวจแต่ละด้าน

ประมาณความยาวที่ต้องใช้:

$$L_{\text{side,required}}
=
\sum_i W_{\text{pad},i}
+
\sum_j W_{\text{filler},j}
+
L_{\text{corner allowance}}
+
L_{\text{required gaps}}$$

ใช้มิติจาก physical views และ placement rules ของ template ไม่ใช้เพียงขนาด bond opening

ตรวจครบทั้งสี่ด้าน:

1. Pads เรียงตาม pin mapping
2. Power pads อยู่ในตำแหน่งที่กำหนด
3. Corner cells มี orientation ถูกต้อง
4. IO fillers ไม่ทับ pads หรือ corners
5. ช่องว่างรอบ pad ring เป็นไปตาม methodology
6. ด้านที่มี pads มากที่สุดยังมีพื้นที่เพียงพอ

**ผลส่งระหว่างขั้น:** ตารางจำนวน pads, ความยาวที่ต้องใช้ และความยาวที่มีจริงของแต่ละด้าน

### 8.7 ขั้นที่ 5 — กำหนด die area และ core area

ตัวอย่างสำหรับทดลอง:

```yaml
FP_SIZING: absolute

DIE_AREA: [0, 0, 1600, 1600]
CORE_AREA: [430, 430, 1170, 1170]
```

ตัวอย่างนี้ให้ core ขนาด:

$$
W_{\text{core}}=1170-430=740\ \mu m
$$

$$
H_{\text{core}}=1170-430=740\ \mu m
$$

พื้นที่ระหว่าง die boundary กับ core boundary มีระยะ 430 µm ต่อด้าน แต่พื้นที่ดังกล่าวต้องถูกแบ่งให้ pad ring, power routing, keepout และองค์ประกอบ finishing ตาม template

LibreLane รองรับ `CORE_AREA` ที่ใช้คู่กับ `DIE_AREA` และมีทั้ง absolute และ relative floorplan sizing ควรตรวจชื่อ configuration variables กับเวอร์ชันที่ติดตั้งจริง [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/reference/step_config_vars.html?utm_source=chatgpt.com)

หลังรันต้องอ่านค่าจาก ODB อีกครั้ง เพราะตำแหน่ง rows และ core boundary อาจถูกปรับให้เข้ากับ site grid

**เกณฑ์ตรวจ**

- Core อยู่ภายใน die
- Core ไม่ทับ IO instances
- มีช่องสำหรับ core ring และการเชื่อมจาก power pads
- ไม่มี standard-cell rows ในบริเวณ macro หรือ keepout
- ไม่มี unknown หรือ unused configuration ที่ทำให้ค่าที่ตั้งไว้ไม่ถูกนำไปใช้

### 8.8 ขั้นที่ 6 — วาง hard macros ถ้ามี

หาก Lab นี้ยังไม่มี hard SRAM ให้บันทึกว่า “ไม่มี hard memory macro ใน baseline” และข้ามขั้นนี้

หากมี SRAM:

1. อ่าน width/height และ supply-pin layers จาก LEF
2. เลือก orientation ที่รองรับโดย physical views
3. หัน signal pins ไปทาง logic ที่ติดต่อกับ SRAM
4. เว้น halo รอบ macro
5. เว้นช่องสำหรับ signal routing และ power connections
6. ตรวจว่าตำแหน่ง PDN straps สามารถเข้าถึง supply pins
7. ตรวจ row cutting หลัง placement

ช่องระหว่าง macros ต้องมีพื้นที่สำหรับทั้ง signal tracks, power structures และ via landing

**ตัวอย่างปัญหา**

SRAM สองก้อนวางห่างกันเพียงเล็กน้อย แม้ไม่เกิด overlap แต่ช่องดังกล่าวอาจไม่พอสำหรับ bus routing และ via arrays จึงยังไม่ถือว่า floorplan ใช้งานได้

**Checkpoint 8-C:** ไม่มี macro overlap และมีหลักฐานว่า supply pins กับ signal pins เข้าถึงได้

---

### 8.9 ขั้นที่ 7 — จัดทำ power-net mapping

สร้างตาราง mapping ก่อนแก้ PDN:

| Physical object | Power pin จาก LEF | Ground pin จาก LEF | Net ที่ต้องเชื่อม |
|---|---|---|---|
| Standard cells | ตรวจชื่อจริง | ตรวจชื่อจริง | Core supply / core ground |
| Core power pad | ตรวจชื่อจริง | ตามชนิด pad | Core supply |
| Core ground pad | ตามชนิด pad | ตรวจชื่อจริง | Core ground |
| Signal IO pads | ตรวจทุก supply pin | ตรวจทุก ground pin | ตาม power domain ของ IO |
| IO supply pads | ตรวจชื่อจริง | ตามชนิด pad | IO supply |
| SRAM | ตรวจชื่อจริง | ตรวจชื่อจริง | ตาม specification ของ SRAM |

ตรวจสามแหล่งให้ตรงกัน:

1. RTL หรือ powered netlist
2. LEF physical terminals
3. Global connection rules และ PDN configuration

ชื่ออย่าง `VDD`, `VSS`, `IOVDD`, `vdd`, `vss` และ `iovdd` ต้องตรวจตามชื่อจริง อย่าสมมติว่าต่างกันเฉพาะตัวพิมพ์แล้วจะถูกจับคู่โดยอัตโนมัติ

สำหรับ IO cell ที่มีทั้ง core-side supply และ IO-side supply ให้ทำ mapping ของทุก pin แยกกัน และตรวจระดับแรงดันที่ cell รองรับ

**ข้อควรระวังในการแก้ missing pin**

ถ้า netlist model ไม่มี power pin ที่ LEF มี ต้องแก้ model หรือ wrapper ให้สอดคล้องกัน ไม่แก้โดยลบ physical pin หรือปิด checker เพื่อให้ flow เดินต่อ

### 8.10 ขั้นที่ 8 — กำหนดโครงสร้าง PDN

วางแผนเส้นทางตั้งแต่แหล่งจ่ายภายนอกจนถึง cells:

1. Package connection และ power pads
2. Supply routing ภายใน IO ring
3. Core power ring
4. Horizontal/vertical straps
5. Standard-cell rails
6. Macro power connections
7. Vias ระหว่าง routing layers

LibreLane มีขั้นตอน `OpenROAD.GeneratePDN` และรองรับทั้ง straps, rings และการเชื่อม macros โดยวิธีที่เลือกต้องสอดคล้องกับ physical views ของ macro [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/usage/pdn.html?utm_source=chatgpt.com)

จัดทำตารางการออกแบบ:

| ส่วน PDN | ข้อมูลที่ต้องระบุ |
|---|---|
| Core ring | Nets, layers, width, spacing, offsets |
| Vertical straps | Layer, width, pitch, offset |
| Horizontal straps | Layer, width, pitch, offset |
| Cell rails | Layer และการเชื่อมกับ straps |
| Macro connection | Pin layers, grid และ via strategy |
| Pad-to-core connection | จุดเชื่อมและเส้นทางจาก supply pads |
| Via connections | Layer pairs และข้อจำกัด geometry |

ใช้ค่าจาก IHP template และ PDK ที่ตรงกับเวอร์ชันของโครงการเป็น baseline

การเพิ่ม strap width หรือจำนวน vias ต้องตรวจผลต่อ spacing, enclosure และ routing resources ด้วย

### 8.11 ขั้นที่ 9 — ตรวจว่าใช้ default หรือ custom PDN

ค้นหาการตั้งค่าในโครงการ:

```bash
rg -n \
  'PDN_CFG|FP_PDN_CFG|PDN_|FP_PDN_|VDD_NETS|GND_NETS' \
  librelane
```

LibreLane ปัจจุบันระบุ `PDN_CFG` สำหรับ custom PDN script และระบุ `FP_PDN_CFG` เป็นชื่อเดิม แต่ project ที่ใช้ revision เก่าต้องอ้างอิง schema ของ revision นั้น [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/reference/step_config_vars.html?utm_source=chatgpt.com)

หาก template ใช้ custom script ให้ตรวจว่า script อ่านตัวแปรใดจริง:

```bash
rg -n \
  'env\(|define_pdn_grid|set_voltage_domain|add_pdn_ring|add_pdn_stripe|add_pdn_connect|global_connect' \
  librelane/pdn_cfg.tcl
```

อย่าเปลี่ยน configuration variable โดยสมมติว่า custom script จะใช้ค่านั้นเสมอ เช่น YAML อาจตั้ง strap width ใหม่ แต่ Tcl ยังใช้ค่าคงที่เดิม

**Checkpoint 8-D:** ระบุได้ว่า PDN script ที่ถูกใช้คือไฟล์ใด และแต่ละพารามิเตอร์สำคัญถูกอ่านจากที่ใด

### 8.12 ขั้นที่ 10 — รัน floorplan และหยุดหลัง PDN

ตรวจ CLI ของ environment ก่อน:

```bash
librelane --help
```

ถ้าเวอร์ชันที่ติดตั้งรองรับ options ต่อไปนี้ และ flow ใช้ step ID นี้ ให้รัน:

```bash
librelane \
  --pdk ihp-sg13g2 \
  --run-tag lab8_baseline \
  --to OpenROAD.GeneratePDN \
  librelane/config.yaml
```

กรณี template ต้องรันผ่าน launcher เฉพาะ ให้ใช้ launcher เดิมพร้อมวิธีหยุดที่ template รองรับ เพื่อให้ custom steps ถูกโหลดครบ

เก็บ:

- Run directory
- Resolved configuration
- Floorplan ODB และ DEF
- Post-PDN ODB และ DEF
- `state_out.json`
- PDN log และ reports

ค้นหาไฟล์โดยไม่ผูกกับเลข stage:

```bash
rg --files librelane/runs/lab8_baseline \
  | rg 'state_out\.json$|\.odb$|\.def$|pdn|floorplan'
```

ตรวจข้อความผิดปกติ:

```bash
rg -n -i \
  'error|warning|failed|unconnected|disconnected|floating|overlap|via' \
  librelane/runs/lab8_baseline
```

ผลจาก `rg` เป็นจุดเริ่มต้นในการอ่านรายงาน ต้องพิจารณาบริบทของข้อความแต่ละรายการ

### 8.13 ขั้นที่ 11 — ตรวจ floorplan จาก ODB และ GUI

เปิด post-floorplan หรือ post-PDN ODB ด้วยวิธีของ template

ตรวจเป็นลำดับ:

1. Die boundary
2. Core boundary
3. Standard-cell rows
4. Pad positions และ orientations
5. Corner cells และ IO fillers
6. Hard macros และ halos
7. Core ring
8. Horizontal/vertical straps
9. จุดเชื่อมกับ rails
10. จุดเชื่อมกับ power pads และ macro pins

อย่าตรวจเพียงภาพเต็ม die ให้ขยายอย่างน้อยบริเวณ:

- มุม core ring
- จุด strap intersection
- จุด strap-to-rail
- จุด pad-to-core
- ขอบ macro ที่มี supply pins

ตัวอย่าง Tcl สำหรับอ่าน geometry ของ floorplan จาก ODB:

```tcl
# ใช้กับ ODB ของ run ที่ต้องการตรวจ
read_db results/lab8/post_pdn.odb

set block [ord::get_db_block]
set dbu [$block getDbUnitsPerMicron]

set die [$block getDieArea]
set core [$block getCoreArea]

puts "DBU_PER_MICRON=$dbu"

puts [format "DIE_UM %.3f %.3f %.3f %.3f" \
    [expr {[$die xMin] / double($dbu)}] \
    [expr {[$die yMin] / double($dbu)}] \
    [expr {[$die xMax] / double($dbu)}] \
    [expr {[$die yMax] / double($dbu)}]]

puts [format "CORE_UM %.3f %.3f %.3f %.3f" \
    [expr {[$core xMin] / double($dbu)}] \
    [expr {[$core yMin] / double($dbu)}] \
    [expr {[$core xMax] / double($dbu)}] \
    [expr {[$core yMax] / double($dbu)}]]

puts "ROW_COUNT=[llength [$block getRows]]"

write_def results/lab8/post_pdn_export.def
```

ปรับชื่อ ODB ให้ตรงกับไฟล์ที่ได้จาก run ก่อนใช้งาน

**ผลที่คาดหวัง:** พื้นที่จริงจาก ODB สอดคล้องกับ configuration และอธิบายการ snap เข้ากับ site grid ได้

---

### 8.14 ขั้นที่ 12 — ตรวจ PDN connectivity ตาม stage

แยกการตรวจสองช่วง:

| ช่วง | ขอบเขต |
|---|---|
| หลัง floorplan/PDN | ตรวจ fixed objects และโครงสร้าง power grid |
| หลัง placement | ตรวจการเชื่อมของ standard cells ที่วางตำแหน่งแล้ว |

ตัวอย่างคำสั่งภายใน OpenROAD session ที่โหลด post-PDN database:

```tcl
check_power_grid \
    -net VDD \
    -floorplanning \
    -error_file reports/lab8/vdd_floorplan_errors.rpt

check_power_grid \
    -net VSS \
    -floorplanning \
    -error_file reports/lab8/vss_floorplan_errors.rpt
```

แทน `VDD` และ `VSS` ด้วยชื่อ nets จริง และตรวจ IO supply nets เพิ่มตามโครงการ

หลัง placement ให้ตรวจซ้ำโดยไม่ใช้ `-floorplanning`:

```tcl
check_power_grid \
    -net VDD \
    -error_file reports/lab8/vdd_placed_errors.rpt

check_power_grid \
    -net VSS \
    -error_file reports/lab8/vss_placed_errors.rpt
```

OpenROAD ระบุว่า `-floorplanning` จะละเว้น non-fixed instances จึงเหมาะกับการตรวจช่วง floorplan แต่ผลผ่านในช่วงนี้ยังไม่ครอบคลุม cells ที่จะถูกวางในภายหลัง [OpenROAD documentation](https://openroad.readthedocs.io/en/latest/main/src/psm/README.html?utm_source=chatgpt.com)

**ตรวจเพิ่มเติม**

- Supply nets มี terminals ตามที่ methodology ต้องการ
- ไม่มี isolated ring หรือ strap islands
- Supply pins ของ fixed macros เชื่อมครบ
- Power pads เชื่อมกับ network ที่ถูกต้อง
- ไม่มีการ short ระหว่าง supply domains

ไม่ใช้ option ที่ข้าม terminal checks เป็นวิธีแก้ปัญหาโดยไม่มีเหตุผลทาง methodology

### 8.15 ขั้นที่ 13 — ตรวจ PDN via geometry

เลือกจุดตรวจอย่างน้อย:

1. Ring corner
2. Strap intersection
3. Strap-to-rail
4. Strap-to-macro pin
5. Pad-to-core connection

สำหรับแต่ละจุด บันทึก:

| รายการ | สิ่งที่ต้องเก็บ |
|---|---|
| Net | ชื่อ power/ground net |
| Location | พิกัด µm |
| Layers | Lower metal, cut และ upper metal |
| Cut geometry | ขนาดและจำนวน cuts |
| Cut spacing | ระยะระหว่าง cuts |
| Lower landing | ขนาดและ enclosure |
| Upper landing | ขนาดและ enclosure |
| Nearby geometry | ระยะห่างจาก metal/vias อื่น |
| DRC evidence | Rule ID และผลตรวจ |

สำหรับ cut rectangle เดี่ยวและ landing rectangle:

$$
E_L=x_{\text{cut,min}}-x_{\text{metal,min}}
$$

$$
E_R=x_{\text{metal,max}}-x_{\text{cut,max}}
$$

$$
E_B=y_{\text{cut,min}}-y_{\text{metal,min}}
$$

$$
E_T=y_{\text{metal,max}}-y_{\text{cut,max}}
$$

ตรวจทั้ง lower และ upper metal เทียบกับกฎที่ใช้กับ via ชนิดนั้น

**ตัวอย่าง**

ถ้า cut อยู่ที่:

$$
[100.00,\ 200.00,\ 100.40,\ 200.40]
$$

และ landing อยู่ที่:

$$
[99.90,\ 199.90,\ 100.50,\ 200.50]
$$

จะได้ enclosure 0.10 µm ทุกด้าน แต่ยังตัดสินว่าผ่านไม่ได้จนกว่าจะเทียบกับ rule ของ PDK

สำหรับ via array ต้องตรวจ array boundary, individual cuts และข้อกำหนดเฉพาะของ array ไม่ใช้ bounding box เพียงอย่างเดียวแทน DRC

### 8.16 ขั้นที่ 14 — Export GDS และตรวจใน KLayout

ใช้ stream-out step ของ template เพื่อให้:

- LEF-to-GDS mapping ถูกต้อง
- Standard-cell, IO และ macro GDS ถูก merge ครบ
- Top-cell name ถูกต้อง
- Layer/datatype mapping ตรงกับ PDK

ไม่ถือว่า DEF ที่เปิดใน KLayout เป็นหลักฐานเทียบเท่าการตรวจ merged GDS

ลำดับการตรวจ:

1. เปิด GDS ที่สร้างจาก post-PDN state
2. โหลด technology/layer properties ของ SG13G2
3. ตรวจจำนวนและชนิดของ cells ที่ merge
4. เปรียบเทียบจุด via audit กับ ODB
5. รัน DRC ที่เกี่ยวข้องกับ PDN geometry
6. บันทึก rule IDs และ marker locations
7. แยก violations ของ PDN ออกจาก violations ที่คาดหมายใน layout ระหว่างขั้น

ตัวอย่าง violations ที่ยังเกี่ยวกับขั้นตอนอนาคต ได้แก่ density fill หรือองค์ประกอบ finishing ที่ยังไม่สร้าง แต่ต้องบันทึกเป็น pending อย่างชัดเจน

**Checkpoint 8-E:** PDN geometry ที่กำหนดให้ตรวจใน Lab ไม่มี violations ค้าง หรือมี waiver ที่ระบุเหตุผลและขอบเขตครบ

### 8.17 ขั้นที่ 15 — ทดลอง `-split_cuts` เมื่อพบปัญหา via

ทำขั้นนี้เมื่อมีหลักฐานว่ารูปแบบ via array เกี่ยวข้องกับปัญหา geometry

OpenROAD รองรับ `-split_cuts` ใน `add_pdn_connect` โดยรับ mapping ของ layer กับ pitch ค่าที่ใช้ต้องมาจากกฎและ via strategy ของโครงการ [OpenROAD documentation](https://openroad.readthedocs.io/en/latest/main/src/pdn/README.html?utm_source=chatgpt.com)

โครงสร้างคำสั่ง:

```tcl
# เป็นรูปแบบสำหรับปรับใช้ ต้องแทนชื่อ layer และ pitch ก่อนรัน
add_pdn_connect \
    -grid <grid_name> \
    -layers {<lower_layer> <upper_layer>} \
    -split_cuts {<layer_name> <pitch_um>}
```

วิธีทดลอง:

1. เก็บ baseline ที่มีปัญหา
2. ใช้ pre-PDN database เดียวกันสำหรับการทดลอง
3. เปลี่ยนเฉพาะ connection strategy ที่ต้องตรวจ
4. สร้าง PDN ใหม่ใน run แยก
5. ตรวจ connectivity ซ้ำ
6. Export GDS ด้วยวิธีเดิม
7. รัน DRC ด้วย deck และ options เดิม
8. เปรียบเทียบจุดผิดเดิมและผลกระทบจุดใหม่

| ตัวชี้วัด | Baseline | Split-cuts variant |
|---|---:|---:|
| Connectivity errors | … | … |
| Via enclosure violations | … | … |
| Cut spacing violations | … | … |
| จำนวน cuts ที่จุดตัวอย่าง | … | … |
| Landing geometry | … | … |
| ผลต่อ routing access | … | … |

อย่าเลือก variant จากจำนวน DRC errors เพียงอย่างเดียว ต้องตรวจว่าการเชื่อมต่อและความสามารถรองรับกระแสยังเหมาะสม

---

### 8.18 ขั้นที่ 16 — ทดลอง floorplan หลายขนาด

ออกแบบการทดลองโดยเปลี่ยนครั้งละหนึ่งปัจจัย:

| Run | สิ่งที่เปลี่ยน | สิ่งที่คงที่ |
|---|---|---|
| A | Baseline | Netlist, SDC, pad mapping, PDN strategy |
| B | เพิ่ม core area | รายการอื่นเหมือน A |
| C | ลด core area | รายการอื่นเหมือน A |
| D | เปลี่ยน strap pitch | Floorplan เหมือนแบบที่เลือก |

ถ้าใช้ absolute `CORE_AREA` ให้เปลี่ยน coordinates จริง การเปลี่ยน `FP_CORE_UTIL` เพียงอย่างเดียวไม่ใช่หลักฐานว่า core เปลี่ยนขนาด

หลัง PDN ผ่าน ให้รันต่อถึง placement หรือ global routing ใน run สำหรับประเมิน โดยใช้ stage IDs ของ flow จริง

เก็บผล:

| ตัวชี้วัด | A | B | C |
|---|---:|---:|---:|
| Core area จริง | … | … | … |
| Usable row area | … | … | … |
| Placement utilization | … | … | … |
| Estimated wire length | … | … | … |
| Setup WNS หลัง placement | … | … | … |
| Global-route overflow ถ้ารัน | … | … | … |
| PDN connectivity errors | … | … | … |
| PDN geometry violations | … | … | … |

เลือกแบบที่ให้พื้นที่และ routing resources เพียงพอ พร้อมอธิบาย tradeoff จากผลจริง ไม่เลือกจากภาพ layout ที่ดูโล่งเพียงอย่างเดียว

### 8.19 ขั้นที่ 17 — ประเมินขอบเขตของ IR drop และ EM

แยกสถานะให้ชัด:

| การตรวจ | ตอบคำถาม |
|---|---|
| Connectivity | มีเส้นทางจ่ายไฟหรือไม่ |
| Geometry DRC | เส้นทางนั้นมีรูปร่างถูกกฎหรือไม่ |
| IR drop | แรงดันที่ load เหลือเท่าใด |
| EM/current limits | Metal และ vias รองรับกระแสตามข้อกำหนดหรือไม่ |

การวิเคราะห์ IR drop ต้องมีข้อมูลแหล่งจ่าย กระแสโหลดและความต้านทานที่เหมาะสม OpenROAD มีเครื่องมือสำหรับวิเคราะห์ power grid แต่คุณภาพของผลขึ้นกับ input models และเงื่อนไขที่ใช้ [OpenROAD documentation](https://openroad.readthedocs.io/en/latest/main/src/psm/README.html?utm_source=chatgpt.com)

สำหรับ Lab นี้:

- ถ้ามีข้อมูลครบ ให้ทำ preliminary analysis และระบุ assumptions
- ถ้ายังไม่มี switching activity หรือ load models ให้บันทึก `PENDING`
- ไม่ใช้ผล connectivity หรือ DRC แทนผล IR/EM

ผลจากช่วง floorplan ยังต้องตรวจซ้ำเมื่อ placement, CTS และ routing เปลี่ยนโหลดหรือโครงสร้าง PDN

### 8.20 การวิเคราะห์ปัญหาที่พบบ่อย

| อาการ | สาเหตุที่ควรตรวจ | แนวทางแก้ |
|---|---|---|
| Pads ไม่พอดีกับด้าน die | Die เล็ก, จำนวน pads หรือ corner allowance ผิด | คำนวณความยาวแต่ละด้านใหม่ |
| Core ทับ IO | Core boundary หรือ pad orientation ผิด | ตรวจ coordinates จาก ODB |
| Rows อยู่ใต้ SRAM | Macro placement/row cutting ไม่ครบ | ตรวจขั้นตอนตัด rows และ halos |
| Global connection ไม่พบ pin | ชื่อ pin หรือ netlist/LEF ไม่ตรง | เทียบ model, LEF และ mapping |
| มี `vss` ใน LEF แต่ model ไม่มี | Physical/electrical views ไม่สอดคล้อง | แก้ model/wrapper ที่ใช้จริง |
| PDN island | Ring/straps ไม่สัมผัสหรือไม่มี via | ตรวจ geometry และ connection layer pairs |
| Connectivity ผ่านแต่ DRC fail | Enclosure, spacing หรือ landing ผิด | ตรวจ cut และ landing จาก GDS/ODB |
| SRAM ไม่ต่อไฟ | Straps ไม่ intersect supply pins | ปรับ macro location หรือ PDN strategy |
| ตั้ง PDN width แล้วไม่เปลี่ยน | Custom script ไม่อ่านตัวแปรนั้น | ตรวจ resolved config และ Tcl |
| `wrong # args` ใน Tcl | Script/API revision หรือ argument list ไม่ตรง | อ่าน failing command และ API ของเวอร์ชันจริง |
| Routing congestion หลังเพิ่ม PDN | Straps/landings ใช้ routing resources มาก | ทดลอง pitch/geometry พร้อมตรวจ power ใหม่ |
| ขาด supply terminals | Top-level power interface ไม่ครบ | ตรวจ block pins และ power-pad integration |

### 8.21 ผลงานที่ต้องส่ง

1. **Floorplan report**: เหตุผลการเลือก die/core และการคำนวณพื้นที่
2. **Power-net mapping**: mapping ครบทุก supply pin ของ cells, IO และ macros
3. **Configuration และ PDN script**: ระบุ revision และค่าที่ใช้จริง
4. **ODB/DEF และ GDS สำหรับ audit**: ระบุ state ที่สร้างแต่ละไฟล์
5. **Connectivity reports**: แยก floorplan-stage กับ post-placement
6. **Via geometry audit**: พิกัด, screenshots, measurements และ DRC rule IDs
7. **Floorplan comparison**: ผลอย่างน้อยสามขนาดหรือสามตัวเลือกที่มีเหตุผล
8. **Acceptance matrix**: แยก PASS, FAIL, PENDING และ N/A

### 8.22 เกณฑ์ผ่าน Lab

| Gate | เกณฑ์ |
|---|---|
| Input consistency | Netlist และ physical views ตรงกัน |
| Floorplan geometry | Die/core ถูกต้องและไม่มี illegal overlaps |
| Rows/sites | ใช้ site ถูกต้องและไม่อยู่ใน forbidden regions |
| Pad integration | Pads/corners/fillers ตรงกับ Lab 6 |
| Macro access | มีพื้นที่สำหรับ signal และ power connections |
| Supply mapping | ทุก required supply pin มี mapping ถูกต้อง |
| Stage connectivity | ตรวจครบตามขอบเขตของ stage |
| PDN geometry | Required PDN checks ผ่านหรือมี approved waiver |
| Reproducibility | มี versions, config, scripts และ state references |
| Deferred checks | IR/EM และ final-stage checks ระบุสถานะชัดเจน |

**ผ่าน Lab 8** หมายถึง floorplan และ PDN พร้อมเข้าสู่ขั้น implementation ถัดไปตามขอบเขตที่ตรวจแล้ว ส่วน final DRC/LVS, density, post-route connectivity และ power integrity ต้องตรวจตาม methodology ในขั้นต่อไป

### 8.23 คำถามท้าย Lab

1. เหตุใด die size ของ full-chip จึงอาจถูกกำหนดโดย pads มากกว่า logic area?
2. Memory แบบ flops กับ hard SRAM ทำให้ floorplan ต่างกันอย่างไร?
3. Core utilization แตกต่างจาก placement density อย่างไร?
4. เหตุใด `check_power_grid -floorplanning` ผ่านจึงยังไม่ยืนยันว่า standard cells ทุกตัวต่อไฟ?
5. Via เชื่อม metal สองชั้นได้ แต่ยังผิด enclosure ได้อย่างไร?
6. เมื่อเพิ่ม strap width ต้องตรวจผลกระทบใดบ้าง?
7. เหตุใดจึงต้องตรวจ merged GDS เพิ่มจาก DEF?
8. ต้องมีข้อมูลอะไรจึงจะสรุป IR drop และ EM ได้?
9. เมื่อเปลี่ยน PDN via strategy ต้องตรวจอะไรซ้ำ?
10. หลักฐานใดใช้สนับสนุนการเลือก floorplan ที่เหมาะสมที่สุด?