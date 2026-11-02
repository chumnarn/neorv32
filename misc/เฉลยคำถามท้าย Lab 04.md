**เฉลยคำถามท้าย Lab 4 — Structural Synthesis Check**

**1. `hierarchy -check` กับ `hierarchy -simcheck` ต่างกันอย่างไร?**

ทั้งสองคำสั่งตรวจความครบถ้วนของ design hierarchy แต่เข้มงวดต่างกัน:

| คำสั่ง | สิ่งที่ทำให้เกิด error | การยอมรับ blackbox |
|---|---|---|
| `hierarchy -check` | ใช้ cell type ที่ไม่พบ module definition | ยอมรับ module ที่ประกาศเป็น blackbox |
| `hierarchy -simcheck` | เช่นเดียวกับ `-check` และเพิ่มการตรวจว่าไม่มี top module หรือมี blackbox instance | ไม่ยอมรับ blackbox instance |

ดังนั้น ASIC design ที่มี SRAM หรือ I/O pad เป็น intentional blackbox อาจผ่าน `-check` แต่ไม่ผ่าน `-simcheck` ได้ตามที่คาดไว้ ทั้งนี้ `-simcheck` **ไม่ได้รัน simulation** หรือพิสูจน์พฤติกรรมของวงจร [yosyshq.readthedocs.io](https://yosyshq.readthedocs.io/_/downloads/yosys/en/latest/pdf/?utm_source=chatgpt.com)

**2. เหตุใดต้องตรวจโครงสร้างทั้งก่อนและหลัง optimization?**

เพราะแต่ละช่วงตอบคำถามต่างกัน:

- **ก่อน optimization:** ตรวจว่าอ่าน source ครบ เลือก top ถูกต้อง และสร้าง CPU, memory, peripheral รวมถึงการเชื่อมต่อได้ตาม configuration
- **หลัง optimization:** ตรวจว่าวงจรที่เหลือยังสอดคล้องกับสิ่งที่ต้องการ และไม่มี logic สำคัญถูกตัดเพราะ output ไม่ได้ใช้งาน หรือ control ถูกผูกเป็นค่าคงที่ผิดพลาด

ตัวอย่างเช่น CPU อาจปรากฏใน hierarchy ตอนแรก แต่ถูกตัดภายหลังเพราะผลลัพธ์ทั้งหมดไม่ส่งถึง output ที่สังเกตได้ การตรวจเพียงก่อน optimization จึงไม่พบปัญหานี้

**3. Undriven signal ต่างจาก uninitialized register อย่างไร?**

| ประเด็น | Undriven signal | Uninitialized register |
|---|---|---|
| ความหมาย | สัญญาณที่ถูกใช้งานแต่ไม่มีแหล่งขับ | มีวงจร register และ driver แต่ยังไม่ทราบค่าเริ่มต้น |
| ตัวอย่าง | ลืมต่อ output ของ module เข้ากับ wire ที่นำไปใช้ | Flip-flop ไม่มี reset และยังไม่เคยรับค่าที่ทราบแน่นอน |
| วิธีแก้ | ตรวจ assignment และ port connections | กำหนด reset หรือ protocol ที่เขียนค่าก่อนอ่าน |

Yosys `check` ตรวจ used wires ที่ไม่มี driver รวมถึง conflicting drivers และ combinational loops แต่การผ่านคำสั่งนี้ไม่ได้รับรองว่า register ทุกตัวมีค่าเริ่มต้นที่ถูกต้อง [yosyshq.net](https://yosyshq.net/yosys/files/yosys_manual.pdf?utm_source=chatgpt.com)

**4. ทำไมการเพิ่ม `keep` จึงไม่ใช่วิธีแก้แรกเมื่อ CPU ถูกตัดออก?**

เพราะ `keep` อาจเก็บวงจรที่เชื่อมต่อผิดไว้ โดยไม่แก้สาเหตุที่ทำให้ CPU ไม่มีผลต่อระบบ

ควรตรวจตามลำดับ:

1. เลือก top module และ wrapper ถูกต้องหรือไม่
2. Configuration เปิดใช้งาน CPU และ peripheral ตามต้องการหรือไม่
3. Reset, clock enable หรือ bus control ถูกผูกเป็นค่าคงที่หรือไม่
4. CPU มีเส้นทางส่งผลถึง UART, GPIO, memory หรือ output ที่ใช้งานจริงหรือไม่
5. Log ระบุว่า logic ถูกตัดจาก constant propagation หรือ unused logic จุดใด

ใช้ `keep` เมื่อมีเหตุผลเฉพาะ เช่น เก็บจุดสังเกตสำหรับ debug พร้อมบันทึกเหตุผล ไม่ควรใช้แทนการแก้ integration

**5. เมื่อใด blackbox จึงถือเป็น intentional macro และเมื่อใดถือเป็น source omission?**

**Intentional macro** คือบล็อกที่ตั้งใจใช้ implementation ภายนอก RTL และมีหลักฐานรองรับ เช่น:

- อยู่ในรายการ macro ที่อนุญาต
- มีชื่อ module, ports, widths และ directions ตรงกับ macro จริง
- มี models/views ที่จำเป็นสำหรับขั้นตอนถัดไป เช่น simulation model, Liberty, LEF และ GDS
- มีแผนตรวจ functional behavior และ physical integration

**Source omission** คือบล็อกที่ควรมี RTL implementation แต่หายไป เช่น ลืมใส่ไฟล์ใน file list, ใช้ชื่อ module ผิด หรือ conditional compilation ตัด source ออก

การประกาศ stub เป็น blackbox จน `hierarchy -check` ผ่าน จึงยังไม่ใช่หลักฐานว่า macro integration ถูกต้อง

**6. เหตุใดต้องตรวจ integration หลัง flatten เพิ่ม?**

Flatten ทำให้ logic ของหลาย module อยู่ในขอบเขตเดียวกัน จึงช่วยให้ตรวจและ optimize เส้นทางข้าม module ได้ชัดขึ้น โดยเฉพาะปัญหา driver, constant propagation และ logic ที่ไม่ส่งผลถึง output

ควรตรวจทั้งก่อนและหลัง flatten เพราะ:

- ก่อน flatten ติดตามปัญหาตามชื่อ instance และขอบเขต module ได้ง่าย
- หลัง flatten เห็นโครงสร้างรวมและผล optimization ข้ามขอบเขตได้ชัดเจน

อย่างไรก็ตาม flatten ไม่ได้พิสูจน์ว่าต่อสัญญาณถูกความหมาย เช่น สลับบิต GPIO ที่มีความกว้างเท่ากันอาจยังผ่าน structural check

**7. Generic cells ก่อน technology mapping ถือเป็น error หรือไม่?**

**ไม่ถือเป็น error หากเหมาะสมกับ stage นั้น** เช่น `$mux`, `$add` และ `$dff` เป็นตัวแทนโครงสร้างวงจรภายใน Yosys ก่อนแปลงเป็น standard cells ของเทคโนโลยีเป้าหมาย

ต้องตัดสินตามจุดตรวจ:

| Stage | สิ่งที่คาดหวัง |
|---|---|
| หลัง elaboration | อาจยังมี processes และ memories |
| หลัง `proc` | Processes ถูกแปลงเป็นโครงสร้าง cells |
| ก่อน technology mapping | Generic cells เป็นเรื่องปกติ |
| หลัง final mapping | ต้องไม่มี generic cells ที่ flow กำหนดว่าต้อง map ให้เสร็จ |

จึงต้องแยก **generic cell ที่เครื่องมือรู้จัก** ออกจาก **unknown module ที่หา implementation ไม่พบ**

**8. ROM ถูกแปลงเป็น mux/constants แล้วไม่เหลือ memory cell ถือว่าผิดหรือไม่?**

**ไม่จำเป็นต้องผิด** เพราะ synthesis สามารถแทน constant ROM ด้วย decoder, mux และ logic constants โดยคงพฤติกรรมการอ่านเดิมได้

สิ่งที่ต้องตรวจคือ:

- Address-to-data mapping และเนื้อหา firmware ถูกต้อง
- Read latency และ enable behavior ตรงกับ specification
- พฤติกรรมของ address ที่ไม่ได้ใช้งานเป็นไปตามข้อกำหนด
- Area และ timing เหมาะสม

หากข้อกำหนดต้องการ **ROM macro โดยเฉพาะ** การได้ mux/constants แม้ทำงานถูกต้องก็ยังไม่ผ่านข้อกำหนดด้าน implementation จำนวน memory cells เพียงอย่างเดียวจึงใช้ตัดสินไม่ได้

**9. Structural check ผ่านยืนยัน functional correctness ได้มากน้อยเพียงใด?**

ยืนยันได้เฉพาะคุณสมบัติทางโครงสร้างที่คำสั่งและ script ตรวจจริง เช่น module ครบ ไม่มี used signal ที่ไร้ driver และไม่มี conflicting drivers หรือ combinational loops ที่เครื่องมือตรวจพบ [yosyshq.net](https://yosyshq.net/yosys/files/yosys_manual.pdf?utm_source=chatgpt.com)

ยังไม่ยืนยันว่า:

- CPU execute instructions ถูกต้อง
- Address decoder เลือก peripheral ถูกตัว
- UART ส่งข้อมูลและ baud rate ถูกต้อง
- Reset sequence และ boot firmware ทำงานครบ
- Netlist มีพฤติกรรมเทียบเท่า RTL ต้นทาง

จึงต้องใช้ self-checking simulation, assertions, regression และ formal verification ตามเป้าหมายของการตรวจด้วย แม้ netlist จะเทียบเท่า RTL ก็ยังต้องตรวจว่า RTL ตรงกับ specification

**10. เพราะเหตุใด power-net connectivity ใน RTL จึงยังไม่ยืนยันว่า PDN ผ่าน?**

เพราะ RTL แสดง **ความตั้งใจในการเชื่อมต่อเชิงตรรกะ** ขณะที่ PDN ต้องสร้างเส้นทางจ่ายไฟทางกายภาพจริง

แม้ RTL จะต่อ `VDD`, `VSS`, `IOVDD` และ `IOVSS` ครบ ยังต้องตรวจว่า:

- Power pins ของ standard cells, SRAM และ pads ถูกผูกกับ net/domain ถูกต้อง
- Rails, straps, rings และ vias เชื่อมถึงกันจริง
- ไม่มี power pins หรือส่วนของ grid ที่ลอย
- การเข้าถึง macro power pins สอดคล้องกับ LEF และ routing layers
- ผ่าน physical connectivity, DRC และการตรวจ IR drop/EM ตามเกณฑ์โครงการ

ตัวอย่างเช่น SRAM อาจต่อชื่อ net `VDD` ถูกต้องใน netlist แต่ไม่มี via เชื่อม strap ลงถึง power pin ของ macro จึงยังจ่ายไฟไม่ได้ ต้องใช้หลักฐานจาก physical database และรายงานตรวจ PDN เพิ่มเติม
