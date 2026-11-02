**เฉลยคำถามท้าย Lab 5 — Technology Synthesis ใน LibreLane**

**1. เหตุใด generic synthesis ผ่านจึงยังไม่ยืนยันว่า technology mapping สำเร็จ?**

Generic synthesis แปลง RTL เป็นโครงสร้างวงจรทั่วไป เช่น adder, mux และ flip-flop โดยยังไม่จำเป็นต้องเลือก standard cells ของ PDK เป้าหมาย

Technology mapping ต้องแปลงโครงสร้างเหล่านั้นเป็น cells ที่มีอยู่จริงใน library เช่น NAND, AOI และ DFF พร้อมรองรับคุณสมบัติที่ RTL ต้องการ

จึงต้องตรวจเพิ่มเติมว่า:

- Netlist ใช้ cells จาก target library ที่ถูกต้อง
- ไม่มี generic cells ที่ขั้นตอนนี้กำหนดว่าต้อง map ให้เสร็จ
- ไม่มี unknown cells หรือ blackboxes นอกเหนือจากรายการที่อนุญาต
- รายงาน area และ timing อ้างอิง library/corner ที่ตั้งใจใช้

**2. Liberty กับ LEF มีหน้าที่ต่างกันอย่างไร?**

| ประเด็น | Liberty (`.lib`) | LEF (`.lef`) |
|---|---|---|
| หน้าที่หลัก | อธิบายพฤติกรรมและคุณสมบัติทางไฟฟ้าของ cells | อธิบายข้อมูลกายภาพแบบ abstract |
| ข้อมูลสำคัญ | Logic function, timing arcs, setup/hold, capacitance, power และ area | ขนาด cell, pin geometry, routing layers และ obstructions |
| ใช้ในงาน | Technology mapping, STA และ power analysis | Floorplan, placement และ routing |
| ข้อจำกัด | ไม่ได้บอกตำแหน่งรูปทรงขาอย่างเพียงพอสำหรับ routing | ไม่ได้ให้ timing model สำหรับ STA |

ชื่อ cell และ pin ของทั้งสอง views ต้องสอดคล้องกัน รวมถึงสอดคล้องกับ netlist และ layout views อื่นที่นำมาใช้

**3. `CLOCK_PERIOD: 20.0` รับประกันการทำงานที่ 50 MHz หรือไม่?**

**ไม่รับประกัน** ค่านี้กำหนดเป้าหมายคาบเวลา 20 ns ซึ่งเท่ากับความถี่ 50 MHz:

\[
f=\frac{1}{20\ \mathrm{ns}}=50\ \mathrm{MHz}
\]

ต้องตรวจว่า flow นำค่านี้ไปสร้าง constraint ให้ clock ที่ถูกต้องจริง และตรวจ timing ภายใต้เงื่อนไขที่ครบ เช่น I/O delays, uncertainty และ operating corners

ผล synthesis ยังใช้ข้อมูล interconnect ที่จำกัด การยืนยัน timing จึงต้องทำซ้ำหลัง CTS และ routing ด้วย extracted parasitics รวมทั้งตรวจ setup, hold และ constraint coverage

ค่านี้ยังไม่ได้เปลี่ยนความถี่ oscillator จริงหรือค่า `CLOCK_FREQUENCY` ที่ firmware ใช้โดยอัตโนมัติ

**4. เหตุใด RAM 32 Kbits จึงอาจทำให้ standard-cell area เพิ่มขึ้นมาก?**

หาก RAM ไม่ถูก map ไปยัง SRAM macro เครื่องมืออาจสร้างหน่วยความจำจาก flip-flops และ logic แทน

ตัวอย่าง RAM ขนาด \(1024\times32\) มีข้อมูลทั้งหมด:

\[
1024\times32=32{,}768\ \text{bits}
\]

การสร้างแบบตรงไปตรงมาอาจต้องใช้ storage ถึง 32,768 flip-flops และยังมี write decoder, enable logic และ read multiplexers เพิ่มเติม ส่วน multiple read ports อาจทำให้ read logic ใหญ่ขึ้นอีก

จึงต้องตรวจทั้ง **จำนวน storage cells และโครงสร้างการอ่าน/เขียน** พร้อมยืนยันว่ามี SRAM mapping หรือ macro instantiation เกิดขึ้นจริงตามที่ตั้งใจ

**5. การไม่พบชื่อ CPU module หลัง flatten หมายถึง CPU ถูกลบเสมอหรือไม่?**

**ไม่เสมอไป** เพราะ flatten นำ logic ภายใน CPU มารวมกับ module ระดับบน ชื่อ module หรือขอบเขต instance เดิมจึงอาจหายไป แม้ logic ของ CPU ยังอยู่

ควรตรวจจากหลักฐานร่วมกัน ได้แก่:

- Log ของ flatten และ optimization
- Sequential/combinational cell statistics
- เส้นทาง logic ของ PC, instruction decode และ datapath
- การเชื่อมต่อ CPU กับ memory และ peripherals
- ผล simulation หรือ equivalence check ตามขอบเขตที่กำหนด

การค้นหาชื่อ module เพียงอย่างเดียวจึงแยกไม่ได้ระหว่าง “ถูก flatten” กับ “ถูก optimize ทิ้ง”

**6. เหตุใด ROM ที่ใช้ constants จึงมี cell count ต่ำกว่าจำนวน ROM bits มากได้?**

เพราะ ROM เป็นฟังก์ชันคงที่ของ address เครื่องมือจึงสามารถลดรูปเป็น logic โดยไม่ต้องสร้าง storage cell หนึ่งตัวต่อหนึ่งบิต

ตัวอย่างเช่น ROM ที่คืนค่า `0` ทุก address อาจเหลือเพียง constant connections ส่วน ROM ที่มีข้อมูลซ้ำกันมากสามารถแชร์และลดรูป logic ได้

ดังนั้น **ROM capacity ไม่เท่ากับจำนวน standard cells** ต้องตรวจเนื้อหา address-to-data mapping, read latency และผลการอ่าน มากกว่าตัดสินจาก cell count เพียงอย่างเดียว

**7. Synthesis exit code เป็นศูนย์เพียงอย่างเดียวเพียงพอต่อการผ่าน Lab หรือไม่?**

**ไม่เพียงพอ** เพราะ exit code เป็นศูนย์บอกว่าโปรแกรมจบโดยไม่พบเงื่อนไขที่ถูกกำหนดให้เป็น fatal error แต่ยังอาจมี warnings หรือผลลัพธ์ที่ผิดจากเป้าหมาย

หลักฐานผ่าน Lab ควรประกอบด้วย:

- Mapped netlist ที่อ่านกลับและ link ได้
- ผล structural checks และรายการ intentional blackboxes
- Cell statistics และ area ที่สมเหตุสมผล
- หลักฐานว่า CPU, memories และ peripherals ยังอยู่ตาม configuration
- Timing report พร้อมข้อจำกัดของผล pre-layout
- การทบทวน warnings ที่มีผลต่อความถูกต้อง
- Source/configuration และ tool/PDK versions ที่ตรวจสอบย้อนกลับได้

**8. ต้องมีหลักฐานอะไรบ้างก่อนเลือก `DELAY 0` แทน `AREA 0`?**

ต้องทดลองเปรียบเทียบภายใต้ source, configuration, library, corner และ constraints เดียวกัน โดยเปลี่ยนเฉพาะ strategy

| หลักฐาน | สิ่งที่ต้องพิจารณา |
|---|---|
| Mapping/structural checks | ทั้งสองผลต้องผ่านเกณฑ์เดียวกัน |
| Timing | Setup slack, TNS และ critical paths ดีขึ้นจริงหรือไม่ |
| Constraint coverage | ไม่มี path สำคัญหลุดจากการวิเคราะห์ |
| Area/cell mix | Area และจำนวน buffers/cells เพิ่มขึ้นเท่าใด |
| Electrical constraints | Slew, capacitance และ fanout มีปัญหาหรือไม่ |
| Physical feasibility | ผลหลัง placement หรือ routing ยังให้ประโยชน์หรือไม่ |

ชื่อ `DELAY` สื่อถึงเป้าหมายของ strategy แต่ไม่ได้รับรองว่าจะเร็วกว่า `AREA` สำหรับทุกวงจร เอกสาร LibreLane ระบุว่าต้องทดลองจึงจะทราบว่า strategy ใดเหมาะที่สุด และควรตรวจด้วยว่าไม่ได้เปิดตัวเลือกที่ override `SYNTH_STRATEGY` อยู่ [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/reference/step_config_vars.html?utm_source=chatgpt.com)

**9. การ copy `state_out.json` เพียงไฟล์เดียวเพียงพอต่อการ resume บนเครื่องใหม่หรือไม่?**

**โดยทั่วไปไม่เพียงพอ** เพราะไฟล์นี้บันทึก state และอ้างอิง design artifacts ไม่ได้บรรจุ netlist, database และ dependencies ทั้งหมดไว้ภายใน

การย้ายเครื่องต้องเตรียม:

- Artifacts ที่ state อ้างถึง เช่น netlist, ODB, SDC หรือ SPEF ตาม stage
- Resolved configuration และไฟล์ประกอบ เช่น macro views และ scripts
- Tool environment และ PDK revision ที่เข้ากันได้
- Paths ที่ใช้ได้บนเครื่องใหม่ หรือการปรับ paths อย่างถูกต้อง
- Logs และ metrics สำหรับตรวจสอบที่มาของ checkpoint

LibreLane ใช้ `state_out.json` เป็น initial state ได้ แต่เอกสารแยกไฟล์ state ออกจาก output artifacts ภายใน step directory อย่างชัดเจน จึงควรทดสอบการโหลด checkpoint หลังย้ายจริงก่อนถือว่า resume ได้สำเร็จ [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/getting_started/newcomers/index.html?utm_source=chatgpt.com)

**10. ผลตรวจใดที่ยังต้องทำก่อนเรียก design ว่าพร้อม sign-off?**

Synthesis ผ่านเป็นเพียงหนึ่ง milestone ก่อน sign-off ยังต้องมีหลักฐานตามขอบเขตโครงการ เช่น:

| ด้าน | ผลตรวจที่ต้องมี |
|---|---|
| Functional verification | Regression, assertions และ boot/application tests ตาม test plan |
| Equivalence | ตรวจความเทียบเท่าระหว่าง RTL และ implementation ตาม flow |
| Timing | Post-route STA ด้วย extracted parasitics ครอบคลุม setup/hold, corners และ modes ที่กำหนด |
| Clock/reset | ตรวจ generated clocks, CDC/RDC และ reset assumptions ตามความจำเป็น |
| Physical verification | DRC, LVS, antenna และ connectivity ผ่านเกณฑ์ |
| Power integrity | PDN connectivity, IR drop และ EM ตามเกณฑ์ที่กำหนด |
| Macro/full-chip integration | SRAM, pads, power domains และ views สอดคล้องกัน |
| Release audit | Final artifacts, checksums, versions, reports และรายการ waiver ที่อนุมัติแล้ว |

คำว่า **“พร้อม sign-off” ต้องผูกกับ checklist และเกณฑ์ของ PDK/foundry หรือโครงการ** โดยแยกผลที่ตรวจผ่านแล้วออกจากรายการที่ยังไม่ได้ตรวจอย่างชัดเจน