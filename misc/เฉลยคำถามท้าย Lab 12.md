**เฉลยคำถามท้าย Lab 12 — SRAM Macro Integration และ Verification**

คำตอบนี้อิง SRAM แบบ synchronous single-port ขนาด 1024 × 32 บิต ส่วนชื่อขา polarity และรายละเอียด read/write ต้องตรวจจาก macro model และ datasheet ของ PDK revision ที่ใช้จริง

**1. เหตุใด RTL memory array จึงไม่รับประกันว่าจะได้ IHP SRAM macro?**

เพราะ RTL array อธิบาย **พฤติกรรมหน่วยความจำ** แต่ไม่ได้ระบุ implementation ทางกายภาพ

เครื่องมืออาจแปลงเป็น flip-flops และ muxes หากไม่มี macro-mapping rules หรือพฤติกรรม RTL ไม่ตรงกับ macro เช่น read latency, จำนวน ports, write mask หรือ reset behavior

จึงต้องใช้ explicit macro instantiation หรือ mapping flow ที่รองรับ แล้วตรวจว่า:

- Mapped netlist มี SRAM instance ตามต้องการ
- ไม่เหลือ flop-based memory โดยไม่ตั้งใจ
- โหลด views ของ macro ถูกต้องและตรง revision เช่น Verilog, Liberty, LEF และ GDS

IHP จัดส่ง SRAM เป็น macro library พร้อม views สำหรับขั้นตอนต่าง ๆ การมีไฟล์เหล่านี้อยู่ใน PDK ไม่ได้ทำให้ RTL array ถูก map ให้อัตโนมัติ [IHP OpenPDK 11181f4 documentation](https://ihp-open-pdk-docs.readthedocs.io/en/latest/contents/reference_libraries/sram.html?utm_source=chatgpt.com)

**2. เหตุใด SRAM 1024 × 32 จึงใช้ word address `[11:2]` สำหรับ byte-addressed bus?**

เพราะหนึ่ง word มี 32 บิต เท่ากับ 4 bytes จึงใช้ address สองบิตล่างเลือก byte ภายใน word ส่วน 1024 words ต้องใช้ word address 10 บิต:

\[
A_{\text{word}}=(A_{\text{byte}}-BASE)\gg2
\]

สำหรับ local byte address ที่มีการจัดแนวเหมาะสม จึงได้:

```verilog
word_addr = local_byte_addr[11:2];
```

| Byte offset | Word address |
|---|---:|
| `0x000`–`0x003` | 0 |
| `0x004`–`0x007` | 1 |
| `0xFFC`–`0xFFF` | 1023 |

ต้องตรวจ address range แยกด้วย มิฉะนั้นการตัดเฉพาะ `[11:2]` อาจทำให้ address นอกช่วงวนกลับมาเข้าหน่วยความจำเดิม และต้องจัดการ misaligned accesses ตาม bus/CPU specification

**3. การขยาย byte enable เป็น bit mask ต้องระวังอะไร?**

ต้องตรวจ **polarity, lane ordering และความหมายของ mask** ให้ตรง macro

หาก `byte_en[0]` ควบคุม `wdata[7:0]` และ macro ใช้ active-high bit-write mask:

```verilog
assign bit_mask = {
    {8{byte_en[3]}},
    {8{byte_en[2]}},
    {8{byte_en[1]}},
    {8{byte_en[0]}}
};
```

หาก macro ใช้ active-low mask ต้องกลับ polarity ตามนิยามจริง

ประเด็นที่ต้องตรวจเพิ่ม ได้แก่:

- Bus master เลื่อน write data และ byte enables มาแล้วหรือไม่
- Endianness และ byte-lane mapping
- Global write enable กับ bit mask ทำงานร่วมกันอย่างไร
- Mask ต้อง stable รอบ clock edge ใด

ควรทดสอบ byte/halfword writes ทุก lane และยืนยันว่า bytes ที่ไม่ได้เลือก **ไม่เปลี่ยนค่า**

**4. เหตุใด read data อาจยังเป็นค่าเดิมที่ clock edge ซึ่ง SRAM รับ address?**

เพราะ synchronous SRAM รับ address ที่ clock edge แล้วจึงสร้าง read data หลัง edge ตาม clock-to-output delay

หาก adapter register จับ `sram_rdata` ที่ edge เดียวกัน มันอาจจับค่าเดิม เพราะข้อมูลของ address ใหม่ยังไม่ออกมา ใน RTL simulation ที่ใช้ nonblocking assignments ก็เกิดลำดับเช่นนี้ได้เช่นกัน

จึงต้องออกแบบ read response ให้สัมพันธ์กับ latency จริง เช่น:

1. ส่ง address/control ให้พร้อมก่อน edge
2. SRAM รับคำสั่งที่ edge
3. รอข้อมูล valid ตาม macro timing
4. Capture หรือส่ง response ในจังหวะที่ bus รองรับ

ไม่ควรแก้ด้วยการเพิ่ม delay เฉพาะใน testbench แต่ต้องกำหนด transaction timing ให้ถูกต้องทั้ง adapter และ testbench

**5. Adapter ป้องกัน write ซ้ำจาก request ที่ค้างได้อย่างไร?**

ต้องแยก **request ที่กำลังค้าง** ออกจาก **การรับ transaction ใหม่** แล้วสร้าง SRAM write strobe เพียงครั้งเดียวต่อ transaction ที่ยอมรับ

ตัวอย่างแนวคิดสำหรับ bus ที่กำหนดให้ request ค้างจนได้รับ ACK:

```text
IDLE → รับและ latch request → ISSUE หนึ่งครั้ง
     → RESPONSE → รอจบ transaction → IDLE
```

Address, data และ mask ต้องถูก latch ตาม acceptance rule และไม่สร้าง write pulse ซ้ำระหว่างรอ response

อย่างไรก็ตาม หากเป็น ready/valid protocol การที่ `valid` ค้างพร้อม `ready` สูงอาจหมายถึง **transaction ใหม่ทุก cycle** จึงห้ามใช้ “รอ request ลด” โดยไม่ตรวจ protocol เพราะจะทำให้ back-to-back transfers หายได้

**6. เหตุใด timing black box จึงไม่เพียงพอสำหรับ sign-off?**

ต้องแยกสองกรณี:

- **Blackbox stub ที่มีเพียง ports:** ไม่มี timing arcs จึงไม่เพียงพอสำหรับ STA
- **Macro timing model ใน Liberty:** ใช้วิเคราะห์ macro boundary timing ได้ หากมี arcs, constraints และ corners ครบตามที่ต้องใช้

แม้มี Liberty ถูกต้อง ก็ยังไม่เพียงพอสำหรับ **sign-off ทั้งชิป** ต้องตรวจ functional behavior, physical integration, power connections, LVS/DRC และข้อกำหนดของ macro เพิ่มด้วย

จึงไม่จำเป็นต้องเปิดวงจรภายใน SRAM ทั้งหมดให้ STA วิเคราะห์ แต่ต้องมี boundary model ที่เชื่อถือได้ และไม่มี missing arcs ที่ทำให้เส้นทางสำคัญหลุดจากการตรวจ

**7. Logical power connection ต่างจาก physical PDN connection อย่างไร?**

| ประเด็น | Logical power connection | Physical PDN connection |
|---|---|---|
| ความหมาย | Power pin ถูกกำหนดให้อยู่บน supply net ที่ถูกต้อง | มี metal/vias เชื่อม pin ไปยังแหล่งจ่ายจริง |
| หลักฐาน | Netlist หรือ database net assignment | Routed geometry และ connectivity reports |
| ปัญหาตัวอย่าง | Macro VDD ถูกผูกผิด domain | Net ถูกชื่อ แต่ไม่มี via ลงถึง macro pin |

จึงต้องตรวจทั้งสองระดับ และตรวจต่อว่าเส้นทางจ่ายไฟรองรับกระแสได้ตามเกณฑ์ IR drop/EM การ “ต่อถึง” อย่างเดียวไม่ได้ยืนยันคุณภาพการจ่ายไฟ

**8. การเพิ่ม wait state โดยหน่วง ACK อย่างเดียวช่วย timing path หรือไม่?**

**ไม่จำเป็นต้องช่วย** เพราะการหน่วง ACK เปลี่ยนเวลาตอบ bus แต่ไม่ได้เพิ่ม register หรือเปลี่ยน combinational path โดยอัตโนมัติ

ตัวอย่าง CPU register → address decode → SRAM input ยังต้องผ่าน setup ตาม capture edge ของ SRAM แม้ ACK จะมาช้าลง ส่วน SRAM output → response register ก็ยังต้องผ่าน timing ของมันเอง

Wait state จะช่วยได้เมื่อสถาปัตยกรรมใช้เวลาที่เพิ่มขึ้นจริง เช่น:

- Register request/address ก่อนเข้า SRAM
- Capture read data ใน cycle ถัดไป
- เพิ่ม pipeline stage
- ใช้ enable sequencing ที่พิสูจน์ได้ว่ามีหลาย cycles สำหรับ path นั้น

หากจะใช้ multicycle constraint ต้องมีหลักฐาน launch/capture behavior รองรับ การหน่วง ACK เพียงอย่างเดียวไม่ใช่เหตุผลเพียงพอ

**9. SRAM สามารถใช้แทน boot ROM ได้ทันทีหรือไม่?**

**ไม่ได้โดยทั่วไป** เพราะ SRAM เป็น volatile memory และไม่มีข้อมูลโปรแกรมที่รับประกันหลังเปิดไฟ ขณะที่ boot ROM ต้องให้ instructions ที่แน่นอนตั้งแต่เริ่ม boot

หากต้องการ execute จาก SRAM ต้องมีวิธีใส่โปรแกรมก่อน CPU fetch เช่น:

- Boot ROM โหลดจาก external flash
- UART loader เขียนโปรแกรมเข้า SRAM
- Debug interface โหลดโปรแกรมขณะ CPU ถูก hold/reset
- Hardware loader ที่ทำงานได้โดยไม่พึ่งโปรแกรมใน SRAM

นอกจากนี้ต้องตรวจ reset vector, memory map, read latency และ bus protocol ให้เข้ากัน การ preload ใน simulation ไม่ใช่หลักฐานว่า silicon จะมีข้อมูลเดียวกันหลัง power-up

**10. Firmware memory test ต่างจาก manufacturing memory test อย่างไร?**

| ประเด็น | Firmware memory test | Manufacturing memory test |
|---|---|---|
| เป้าหมาย | ตรวจการใช้งาน memory ผ่าน CPU และระบบ bus | ตรวจ manufacturing defects ตาม fault models และเกณฑ์ทดสอบ |
| การเข้าถึง | ผ่าน software-visible address space | อาจใช้ MBIST, test controller หรือ tester interface |
| ตัวอย่าง | Walking-one/zero, address patterns, byte writes | March algorithms และการทดสอบตาม voltage/frequency conditions |
| ข้อจำกัด | อาจไม่ครอบคลุม timing หรือ defect classes ทั้งหมด | ต้องมี test architecture และ coverage evidence ที่เหมาะสม |

Firmware test เหมาะกับการตรวจ adapter, address decode, byte enables และ data integrity แต่ต้องระวังไม่ทดสอบทับ code, stack หรือข้อมูลที่ใช้งานอยู่ รวมถึง cache ที่อาจทำให้ไม่ได้เข้าถึง SRAM จริง

Manufacturing test ต้องกำหนด fault coverage, test conditions และวิธีอ่านผลอย่างเป็นระบบ **การมีชื่อ macro ลงท้ายด้วย `bist` หรือ firmware test ผ่าน ไม่ได้ยืนยันว่า manufacturing test integration และ coverage ครบแล้ว**