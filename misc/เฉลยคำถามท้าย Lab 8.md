**เฉลยคำถามท้าย Lab 8 — Floorplan และ PDN**

**1. เหตุใด die size ของ full-chip จึงอาจถูกกำหนดโดย pads มากกว่า logic area?**

เพราะ pads ต้องใช้พื้นที่รอบขอบ die ตามจำนวน ขนาด pitch และ spacing รวมถึง corner cells, power pads และข้อกำหนดการ bonding

แม้ logic ภายในมีขนาดเล็ก แต่แต่ละด้านต้องยาวพอสำหรับ pads ที่จัดไว้ และต้องเผื่อ core-to-pad clearance, power ring และ seal ring ด้วย จึงเกิด **pad-limited design** ซึ่ง die size ถูกจำกัดโดยการจัด I/O มากกว่า logic area

การลด logic area จึงอาจไม่ช่วยลด die size หากข้อจำกัดรอบขอบชิปยังเท่าเดิม

**2. Memory แบบ flops กับ hard SRAM ทำให้ floorplan ต่างกันอย่างไร?**

| ประเด็น | Memory แบบ flops | Hard SRAM |
|---|---|---|
| โครงสร้าง | Standard cells จำนวนมาก พร้อม decoder/mux | Macro ที่มีขนาดและรูปทรงคงที่ |
| Placement | กระจายใน standard-cell rows ได้ | ต้องกำหนดตำแหน่งและ orientation ของ macro |
| Routing | มีการเชื่อมต่อกระจายและอาจหนาแน่นมาก | ต้องเข้าถึง macro pins และเผื่อ routing channels |
| PDN | ใช้ standard-cell rails เป็นหลัก | ต้องเชื่อม macro power pins โดยเฉพาะ |
| พื้นที่ | มักใช้พื้นที่มากสำหรับ memory ขนาดใหญ่ | มักมี storage density สูงกว่า แต่ต้องเผื่อ halo/blockages |

Hard SRAM จึงเปลี่ยน floorplan จากการจัด standard cells เป็นหลัก ให้ต้องวางแผน **macro placement, pin access, channels และ macro PDN** เพิ่มด้วย

**3. Core utilization แตกต่างจาก placement density อย่างไร?**

**Core utilization** เป็นอัตราส่วนพื้นที่ cells ต่อพื้นที่อ้างอิงของ core ใช้ช่วยกำหนดขนาด floorplan ส่วน **placement density** เป็นเป้าหมายความหนาแน่นที่ placer ใช้กระจาย cells ในพื้นที่ที่วางได้

ตัวอย่างเชิงแนวคิด:

\[
U_{\text{core}}=\frac{A_{\text{cells}}}{A_{\text{core}}}
\]

แต่พื้นที่ที่ placer ใช้ได้จริงอาจลดลงจาก macros, halos และ blockages ทำให้ core utilization ต่ำ แต่บางบริเวณยังมี placement density สูงและเกิด congestion ได้

ต้องตรวจนิยามของ flow ว่านับ macro area และพื้นที่ที่ถูกกีดกันอย่างไร จึงไม่ควรนำค่าทั้งสองมาเทียบว่าเป็นเปอร์เซ็นต์ชนิดเดียวกันโดยตรง

**4. เหตุใด `check_power_grid -floorplanning` ผ่านจึงยังไม่ยืนยันว่า standard cells ทุกตัวต่อไฟ?**

เพราะตัวเลือก `-floorplanning` **ละเว้น non-fixed instances** เพื่อให้ตรวจ power grid ได้ในช่วงที่ instances ยังวางไม่เรียบร้อย ดังนั้น standard cells ที่ไม่ได้ fixed อาจอยู่นอกขอบเขตการตรวจนี้ [OpenROAD documentation](https://openroad.readthedocs.io/en/latest/main/src/psm/README.html?utm_source=chatgpt.com)

ผลผ่านจึงยืนยันได้เฉพาะ connectivity ตามขอบเขตที่คำสั่งตรวจ ไม่ได้ยืนยันว่า standard-cell power pins ทุกตัวต่อถึงแหล่งจ่ายแล้ว

หลัง placement ต้องตรวจใหม่โดยไม่ใช้ข้อยกเว้นนี้ พร้อมตรวจ power-pin mapping, rails, vias และทุก supply domain ที่ใช้งานจริง

**5. Via เชื่อม metal สองชั้นได้ แต่ยังผิด enclosure ได้อย่างไร?**

เพราะ **electrical connectivity** กับ **geometric design rules** เป็นคนละเงื่อนไข

Via cut อาจสัมผัส metal ทั้งสองชั้นจนเกิดการเชื่อมต่อ แต่ metal รอบ cut ยื่นออกไปไม่พอตาม enclosure rule เช่น:

- Metal ด้านหนึ่งสั้นหรือแคบเกินไป
- Via array ต้องการ enclosure ต่างจาก single cut
- กฎขึ้นกับ metal width หรือทิศทาง
- Intermediate metal ของ stacked vias มีรูปทรงไม่ถูกต้อง

ดังนั้น connectivity ผ่านยังต้องตรวจ DRC บน geometry จริง โดยเฉพาะ via stacks และบริเวณที่ metal หลายรูปทรงรวมกัน

**6. เมื่อเพิ่ม strap width ต้องตรวจผลกระทบใดบ้าง?**

การเพิ่มความกว้างอาจลดความต้านทานของ strap แต่ต้องประเมินผลร่วมกัน:

- **PDN:** IR drop, current distribution และคอขวดที่ vias หรือทางเชื่อมส่วนอื่น
- **Routing:** จำนวน tracks ที่เหลือ, congestion และ signal detours
- **DRC:** Width-dependent spacing, enclosure, minimum area และ via-array rules
- **Integration:** การชน macro obstructions, pin access และระยะห่างจาก ring/pads
- **Timing:** ผลจาก routing ที่เปลี่ยนและ parasitic coupling
- **Manufacturing:** Metal density และข้อกำหนด slotting หากเทคโนโลยีกำหนด

Strap ที่กว้างขึ้นจึงไม่ได้รับรองว่าทั้ง PDN และ signal routing จะดีขึ้นเสมอ

**7. เหตุใดจึงต้องตรวจ merged GDS เพิ่มจาก DEF?**

เพราะ DEF ใช้ข้อมูล physical abstracts จาก LEF และบันทึก placement/routing ขณะที่ merged GDS รวม **geometry จริงของ standard cells, macros, pads และ vias**

ปัญหาที่อาจเห็นชัดเมื่อรวม GDS ได้แก่:

- LEF กับ GDS คนละ revision หรือ geometry ไม่ตรงกัน
- Layer mapping ผิดหรือ cell layout หาย
- รูปทรงบริเวณ cell boundaries ผิด spacing/enclosure
- Via geometry หรือ macro internal geometry ที่ abstract ไม่ได้แสดงครบ

จึงต้องทำ DRC และ layout connectivity/LVS ตาม flow บน layout ที่รวมแล้ว ไม่ใช้ภาพ DEF หรือ connectivity ใน database เพียงอย่างเดียวแทนผลตรวจ final layout

**8. ต้องมีข้อมูลอะไรจึงจะสรุป IR drop และ EM ได้?**

| ข้อมูล | เหตุผลที่ต้องใช้ |
|---|---|
| PDN geometry และ connectivity | ระบุเส้นทางกระแสจริง |
| Metal/via resistance และ operating temperature | คำนวณแรงดันตกและความต้านทาน |
| Supply voltage และตำแหน่งแหล่งจ่าย | กำหนด boundary conditions |
| Cell/macro currents และ spatial distribution | ระบุตำแหน่งและขนาดโหลด |
| Activity, workload และ operating mode | ทำให้โหลดสัมพันธ์กับการใช้งานจริง |
| Package/bond-wire model ตามขอบเขตวิเคราะห์ | รวมผลของเส้นทางจ่ายไฟภายนอก die |
| Foundry limits และ reliability conditions | ใช้ตัดสินผล EM ของ wires และ vias |

ต้องระบุด้วยว่าผลเป็น **static IR** หรือ **dynamic IR** เพราะใช้โมเดลและตอบคำถามต่างกัน OpenROAD PDNSim เป็นเครื่องมือวิเคราะห์ static IR จึงไม่ควรนำผลไปอ้างว่าครอบคลุม transient droop โดยอัตโนมัติ [OpenROAD documentation](https://openroad.readthedocs.io/en/latest/main/src/psm/README.html?utm_source=chatgpt.com)

การสรุป EM ต้องมีเกณฑ์กระแสที่ตรงกับชนิดตัวนำ อุณหภูมิ และเงื่อนไขอายุการใช้งาน ไม่ใช่อาศัยเพียงข้อความว่า grid connected

**9. เมื่อเปลี่ยน PDN via strategy ต้องตรวจอะไรซ้ำ?**

อย่างน้อยต้องตรวจ:

1. **Via generation:** Vias ถูกสร้างครบหรือมี failed/removed vias
2. **Connectivity:** ทุก supply net, macro power pin และ standard-cell rail ยังเชื่อมถึงกัน
3. **DRC:** Cut spacing, enclosure, metal spacing, minimum area และ manufacturing grid
4. **IR drop/EM:** Via resistance, จำนวน cuts และการกระจายกระแสเปลี่ยนไปหรือไม่
5. **Routing feasibility:** Via arrays หรือ metal extensions กีดขวาง signal routing หรือไม่
6. **Merged GDS verification:** Geometry สุดท้ายยังถูกต้องหลัง stream-out และ merge

OpenROAD มี failed-via report สำหรับ vias ที่สร้างไม่ได้หรือถูกลบ และมีการซ่อม PDN vias หลัง routing แต่ยังต้องตรวจผลหลังการซ่อมอีกครั้ง [OpenROAD documentation](https://openroad.readthedocs.io/en/latest/main/src/pdn/README.html?utm_source=chatgpt.com)

Checkpoint ตั้งแต่ขั้นตอนสร้าง PDN ที่ได้รับผลกระทบและขั้นตอนถัดไปต้องสร้างใหม่ ไม่ควรใช้ผล IR หรือ DRC ของ via strategy เดิมรับรองแบบใหม่

**10. หลักฐานใดใช้สนับสนุนการเลือก floorplan ที่เหมาะสมที่สุด?**

ต้องเปรียบเทียบหลาย candidate ภายใต้ configuration และเงื่อนไขเดียวกัน โดยใช้หลักฐานมากกว่าภาพ floorplan หรือ utilization เพียงค่าเดียว

| ด้าน | หลักฐานที่ใช้เปรียบเทียบ |
|---|---|
| ขนาดและ I/O | Die/core dimensions, pad fit, bond/package feasibility |
| Placement | Legal placement, density distribution และพื้นที่เผื่อ optimization |
| Routing | Congestion, overflow, pin access และผล detailed routing |
| Timing | Setup/hold slack และ critical paths ใน stage ที่เทียบกัน |
| Power integrity | Connectivity, IR drop และ EM ตามข้อมูลที่มี |
| Physical verification | DRC/LVS และ unresolved violations |
| ความทำซ้ำได้ | Source/configuration, tool/PDK versions และ reports ของแต่ละ candidate |

ในช่วง floorplan เลือกได้เพียง **candidate ที่มีหลักฐานสนับสนุนดีที่สุดในขณะนั้น** การอ้างว่าเหมาะสมที่สุดสำหรับ release ต้องอาศัยผล downstream เช่น post-route timing, extraction และ physical verification ด้วย