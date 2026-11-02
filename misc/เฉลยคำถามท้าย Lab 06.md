**เฉลยคำถามท้าย Lab 6 — Pad Ring และ Pin Mapping**

คำตอบนี้ใช้ชื่อไฟล์เป็นตัวอย่าง ต้องเทียบกับชื่อไฟล์และลำดับ steps ของโปรเจกต์ที่ใช้งานจริง

**1. ย้าย UART pads ไปอีกด้าน: ไฟล์ใดต้องเปลี่ยนและ checkpoints ใดใช้ต่อไม่ได้?**

ต้องแก้ไฟล์กำหนดตำแหน่ง pad เช่น `config.yaml` โดยย้ายชื่อ UART pad instances ระหว่าง `PAD_WEST`, `PAD_EAST`, `PAD_NORTH` หรือ `PAD_SOUTH` ตามต้องการ ตัวแปรเหล่านี้เป็นส่วนหนึ่งของการตั้งค่า pad ring ใน LibreLane [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/reference/step_config_vars.html?utm_source=chatgpt.com)

ไฟล์ที่ต้องทบทวนเพิ่มเติม ได้แก่:

- **Pad/pin assignment และ bond plan:** ปรับด้าน ลำดับ พิกัด และการเชื่อมไปยัง package
- **Placement constraints หรือ scripts:** แก้รายการที่อ้างตำแหน่งเดิม
- **Testbench/เอกสาร interface:** แก้หากอ้าง physical pin mapping
- **Padframe RTL:** เปลี่ยนเมื่อเปลี่ยนชื่อ instance หรือ logical connection เท่านั้น การย้ายตำแหน่งอย่างเดียวอาจไม่ต้องแก้ RTL

Checkpoint ตั้งแต่ **ขั้นตอนแรกที่ใช้ตำแหน่ง pad** และผลทั้งหมดที่สืบต่อจากนั้นต้องสร้างใหม่ เช่น pad placement, PDN ที่เกี่ยวข้อง, placement, CTS, routing, extraction และ physical verification

Synthesis checkpoint อาจใช้ต่อได้ หาก RTL, cell types และ logical connections ไม่เปลี่ยน แต่ต้องยืนยัน dependencies ก่อน resume

**2. ลบ input pad ออกจาก `PAD_WEST` หนึ่งตัว: flow ตรวจพบเมื่อใด และเหตุใดต้องมี independent inventory?**

หากลบเฉพาะชื่อจาก placement list แต่ instance ยังอยู่ใน RTL:

- Synthesis อาจผ่าน เพราะวงจรเชิงตรรกะยังครบ
- Inventory check ควรพบก่อนเริ่ม physical implementation
- หากไม่มี inventory check อาจพบเมื่อสร้าง pad ring หรือตรวจ unplaced instances ภายหลัง ทั้งนี้ขึ้นกับเวอร์ชันและ checks ของ flow

จึงไม่ควรรับรองว่าจะเกิด error ที่ step เดียวกันในทุก flow

**Independent inventory** คือรายการ pad ที่ต้องมี ซึ่งจัดทำจาก interface specification แยกจากรายการ placement แล้วนำมาเปรียบเทียบ:

\[
\text{Expected pads}=\text{RTL pad instances}=\text{Placed pads}
\]

ต้องตรวจทั้งรายการหาย รายการเกิน รายการซ้ำ cell type และ signal mapping หากสร้าง expected inventory จาก `PAD_WEST` และรายการด้านอื่นชุดเดียวกัน การลบ pad อาจทำให้ทั้งข้อมูลจริงและข้อมูลอ้างอิงหายพร้อมกันจนตรวจไม่พบ

**3. ทำ GPIO bus concatenation กลับลำดับ: ทำไม synthesis จึงอาจผ่านแต่ walking-one test fail?**

เพราะการสลับบิตยังเป็นวงจรที่ถูกต้องตาม syntax และโครงสร้าง เช่น:

```verilog
// Mapping ที่ต้องการ
assign gpio_pad = gpio_core;

// Mapping ที่กลับลำดับบิต
assign gpio_pad = {
    gpio_core[0], gpio_core[1], gpio_core[2], gpio_core[3]
};
```

สำหรับ bus `[3:0]` แบบหลังทำให้ `gpio_core[0]` ไปออกที่ `gpio_pad[3]` เครื่องมือ synthesis ไม่มีข้อมูลว่าข้อกำหนดต้องการให้บิตใดออกขาใด จึงอาจผ่านได้

Walking-one test เขียนค่า `0001`, `0010`, `0100`, `1000` แล้วตรวจ **ขาภายนอกตาม mapping ที่ระบุใน specification** จึงพบว่าขาที่ตอบสนองผิดตำแหน่ง

Expected result ของ testbench ต้องเป็นอิสระจาก concatenation ที่ใช้ใน DUT มิฉะนั้นอาจทำผิดเหมือนกันและผ่านลวงได้

**4. เปรียบเทียบ top view กับ bottom view ของ bond plan: pin numbering เปลี่ยนการมองอย่างไร?**

การเปลี่ยนจาก top view เป็น bottom view ทำให้ตำแหน่งที่เห็นเป็นภาพสะท้อน โดยต้องกำหนดแกนที่พลิกและ orientation ให้ชัดเจน

**หมายเลขขาจริงและชื่อ net ไม่เปลี่ยน** แต่ตำแหน่งของหมายเลขบนภาพเปลี่ยน และลำดับที่เห็นตามเข็ม/ทวนเข็มนาฬิกาจะกลับกันเมื่อเทียบใน orientation ที่สอดคล้องกัน

Bond plan จึงควรระบุ:

- มองจากด้านใด: die active side, package top หรือ package bottom
- Pin 1 marker และจุดอ้างอิง orientation
- ทิศแกน X/Y และหน่วยพิกัด
- Die pad ID, package pin number และ net name แยกคอลัมน์กัน

อย่าใช้คำว่า “เรียงจากซ้ายไปขวา” โดยไม่มี view และ orientation กำกับ และอย่าเปลี่ยนหมายเลขขาเพื่อชดเชยการกลับภาพ

**5. เหตุใด VDD/IOVDD แยก nets และ connected rail ยังไม่ยืนยัน ESD/latch-up compliance?**

การแยก nets และตรวจ rail connectivity ยืนยันได้เพียงส่วนหนึ่งของ power-domain integration เช่น ไม่ short ผิด domain และมีเส้นทางโลหะเชื่อมต่อ

แต่ **ESD** ยังขึ้นกับ protection devices, clamp/discharge paths, ความต้านทานของเส้นทาง และความสัมพันธ์กับ package ส่วน **latch-up** ยังขึ้นกับ well/substrate contacts, guard structures, spacing และเงื่อนไขการฉีดกระแส

จึงต้องตรวจเพิ่มเติมตามกฎของ PDK/foundry ได้แก่:

- ชนิด จำนวน และตำแหน่ง power/ground/protection cells
- ความต่อเนื่องของ protection paths และ domain crossings
- Well/substrate connections และกฎด้าน latch-up
- Power sequencing และแรงดันที่ขารับได้ขณะบาง domain ปิด
- Package/bond-wire connections ที่ใช้จริง

แม้ I/O library จะมีอุปกรณ์ป้องกัน ESD อยู่ในโครงสร้าง ก็ยังต้องตรวจการประกอบระดับชิปตามข้อกำหนด ไม่สามารถสรุป compliance จาก connectivity report อย่างเดียวได้ [IHP OpenPDK 11181f4 documentation](https://ihp-open-pdk-docs.readthedocs.io/en/latest/contents/io_library/02_drive_strength.html?utm_source=chatgpt.com)

**6. ความต่างระหว่าง pad-cell origin, pad terminal center และ bondpad center คืออะไร?**

| จุดอ้างอิง | ความหมาย | ใช้ทำอะไร |
|---|---|---|
| **Pad-cell origin** | จุดกำเนิดพิกัดของ cell ใน library | วาง instance และแปลงพิกัดตาม orientation |
| **Pad terminal center** | จุดกึ่งกลางของรูปทรง terminal ที่เลือก เช่น external `pad` pin | ตรวจตำแหน่ง electrical pin และสร้างรายงานพิกัด |
| **Bondpad center** | จุดกึ่งกลางของพื้นที่ที่กำหนดให้ bonding | จัดทำ bond plan และตรวจ assembly geometry |

ทั้งสามจุดไม่จำเป็นต้องตรงกัน และ terminal หนึ่งอาจมีหลาย shapes จึงต้องกำหนดวิธีเลือกจุดอ้างอิงให้ชัดเจน

พิกัดบน die ต้องคำนวณจาก **local geometry + rotation/mirroring + placement translation** ไม่ควรใช้ origin แทน bond center โดยตรง หาก bonding opening รวมอยู่ใน I/O cell ก็ต้องอ่านตำแหน่งจาก geometry ของ cell นั้น

**7. เหตุใด mock-pad simulation และ final powered-netlist/PDK simulation จึงครอบคลุมต่างกัน?**

| การทดสอบ | สิ่งที่ตรวจได้เด่น |
|---|---|
| **Mock-pad simulation** | Logical mapping, polarity, bus ordering, input/output direction และ OE behavior ตาม mock model |
| **Final powered-netlist + PDK models** | การเชื่อมต่อของ netlist จริง ความตรงกันของ cell/pin interfaces และพฤติกรรมที่ PDK models รองรับ |
| **Timing simulation พร้อม SDF** | เพิ่มผลของ delays และ timing checks ที่ models/SDF รองรับ |

Mock model มักเรียบง่าย เช่น input pad เป็น `assign p2c = pad;` จึงไม่จำเป็นต้องครอบคลุม power pins, unknown states หรือรายละเอียดของ cell จริง ขณะที่ IHP ระบุ interface ของ output pad ที่มีทั้งขาสัญญาณและขา supply หลาย domain [IHP OpenPDK 11181f4 documentation](https://ihp-open-pdk-docs.readthedocs.io/en/latest/contents/io_library/04_pin_interface.html?utm_source=chatgpt.com)

อย่างไรก็ตาม **powered-netlist simulation ก็ตรวจได้เท่าที่ model จำลองไว้** การมี power pins ไม่ได้แปลว่าจะจำลอง power-off behavior, ESD, latch-up หรือ analog signal integrity ได้ครบ และไม่ทดแทน LVS/physical verification

**8. เมื่อเปลี่ยน output drive strength ต้องทบทวน timing/load, power, package และ geometry อะไรบ้าง?**

| ด้าน | สิ่งที่ต้องทบทวน |
|---|---|
| **Timing/load** | Pad delay, rise/fall time, external capacitance, slew limits, VOH/VOL และ timing budget |
| **Power** | Switching current, simultaneous switching outputs, IR drop, ground bounce และ EM |
| **Package/board** | Bond-wire/package RLC, PCB loading, overshoot, undershoot, ringing และ receiver limits |
| **Geometry** | Cell dimensions, pin shapes, origin/bond-center offsets, legal orientation, rail alignment และ pad spacing |
| **Views/integration** | RTL cell type, simulation model, Liberty, LEF, GDS และ CDL/SPICE ต้องเป็นรุ่นที่สอดคล้องกัน |

ต้องตรวจใหม่แม้ cell variants ดูคล้ายกัน เพราะไม่ควรสมมติว่า geometry และ electrical characteristics เหมือนกันทั้งหมด

สำหรับ IHP ชื่อ drive strength อ้างถึง **short-circuit current** ตามนิยามของ library จึงไม่ควรตีความตัวเลข mA ว่าเป็นกระแสใช้งานต่อเนื่องที่รับประกันได้ในทุกโหลดและทุกเงื่อนไข ต้องพิจารณา characterization และข้อกำหนดที่เกี่ยวข้องด้วย [IHP OpenPDK 11181f4 documentation](https://ihp-open-pdk-docs.readthedocs.io/en/latest/contents/io_library/02_drive_strength.html?utm_source=chatgpt.com)

**แหล่งอ้างอิง**

- [LibreLane — Built-in Steps and Configuration Variables](https://librelane.readthedocs.io/en/latest/reference/step_config_vars.html): การตั้งค่า `OpenROAD.PadRing` และรายการ pads
- [IHP — I/O Pin Interface](https://ihp-open-pdk-docs.readthedocs.io/en/latest/contents/io_library/04_pin_interface.html): Signal และ supply pins ของ I/O cells
- [IHP — Output Drive Strength](https://ihp-open-pdk-docs.readthedocs.io/en/latest/contents/io_library/02_drive_strength.html): นิยาม drive strength และข้อมูล electrical characterization
- [LibreLane — Power Distribution Networks](https://librelane.readthedocs.io/en/latest/usage/pdn.html): การตั้งค่า power connections และ PDN

สำหรับผลตรวจรับจริง ต้องใช้เอกสารและ library views ที่ตรงกับ **PDK revision ของโปรเจกต์** รวมถึง package drawing และ assembly rules ที่เลือกใช้
