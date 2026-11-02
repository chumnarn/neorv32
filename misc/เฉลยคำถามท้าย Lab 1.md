**เฉลยคำถามท้าย Lab 1 — NEORV32 สำหรับ ASIC**

### 1. เหตุใด Git commit เดียวกันจึงสร้าง NEORV32 hardware ต่างกันได้?

Git commit ระบุเวอร์ชันของ source code แต่ไม่ได้ระบุ configuration และสภาพแวดล้อมที่ใช้สร้าง hardware ทั้งหมด

NEORV32 สามารถปรับ hardware ผ่าน generics และตัวเลือกการ build เช่น

- ชุดคำสั่งและส่วนขยายของ CPU
- ขนาด instruction/data memory
- การเปิดหรือปิด UART, SPI, GPIO และ peripheral อื่น
- Boot configuration และ memory implementation
- Top-level ที่ใช้ รวมถึงค่าที่ wrapper ส่งต่อ
- เวอร์ชันเครื่องมือ ตัวเลือก elaboration และไฟล์ที่นำเข้า

ดังนั้น source code commit เดียวกันอาจถูก elaborate เป็นวงจรคนละแบบได้ เช่น รุ่นหนึ่งมี UART และ RAM 32 KiB แต่อีกรุ่นไม่มี UART และใช้ RAM 64 KiB

**ประเด็นสำคัญ:** การทำซ้ำผลต้องระบุทั้ง **source revision + configuration + toolchain + build inputs** ไม่ใช่ Git commit เพียงอย่างเดียว

### 2. `neorv32_top`, ASIC wrapper และ `chip_top` มีหน้าที่ต่างกันอย่างไร?

ชื่อ wrapper และ `chip_top` อาจต่างกันในแต่ละโครงการ แต่โดยทั่วไปแบ่งหน้าที่ดังนี้

| ระดับ | หน้าที่ |
|---|---|
| `neorv32_top` | Top-level ของ NEORV32 SoC รวม CPU, memory subsystem, peripheral และ interface ตาม generics |
| ASIC wrapper | กำหนด configuration ที่ใช้จริง และปรับ interface ให้เหมาะกับ ASIC เช่น การเชื่อม SRAM macro, reset และการกำหนดค่าขาที่ไม่ได้ใช้ |
| `chip_top` | Top-level ของชิป รวม core กับ I/O pad cells และการเชื่อมต่อสัญญาณระดับชิป รวมถึง power/ground ตามโครงสร้างของ PDK และ flow |

ภาพรวมเชิงลำดับชั้นคือ

```text
chip_top → ASIC wrapper → neorv32_top
```

ทั้งนี้ SRAM adapter หรือ reset logic อาจอยู่คนละระดับได้ ขึ้นอยู่กับการออกแบบจริง ส่วน power ring และ power grid ต้องอาศัย physical implementation เพิ่มเติม ไม่ได้เกิดครบถ้วนจาก RTL เพียงอย่างเดียว

### 3. หาก `CLOCK_FREQUENCY` ระบุ 50 MHz แต่ SDC ใช้ period 10 ns จะกระทบ hardware/software assumptions อย่างไร?

สองค่านี้มีหน้าที่ต่างกัน

| ค่า | ความหมาย |
|---|---|
| `CLOCK_FREQUENCY = 50_000_000` | ข้อมูลความถี่ที่ RTL หรือ software ใช้คำนวณพฤติกรรมที่อ้างอิงเวลา |
| SDC `create_clock -period 10` | กำหนดให้ synthesis/STA วิเคราะห์ timing สำหรับคาบ 10 ns หรือ 100 MHz |

**SDC ไม่ได้สร้างหรือเปลี่ยนความถี่ clock ที่ป้อนให้ชิปจริง**

ผลกระทบขึ้นอยู่กับ clock ที่ใช้งานจริง:

- **ถ้าป้อน 100 MHz:** ส่วนที่คำนวณโดยสมมติ 50 MHz จะมีเวลาไม่ตรง เช่น UART ที่เลือก divider จาก 50 MHz อาจส่งเร็วประมาณสองเท่า และ software delay ที่คำนวณจากจำนวน cycle จะสั้นลงประมาณครึ่งหนึ่ง
- **ถ้าป้อน 50 MHz:** พฤติกรรมที่อิงค่า 50 MHz อาจถูกต้อง แต่ flow ถูกสั่งให้ปิด timing ที่ 100 MHz ซึ่งอาจเพิ่มพื้นที่หรือกำลังไฟ หรือทำให้รายงาน timing ไม่ผ่านโดยไม่จำเป็นต่อเป้าหมายจริง

ความคลาดเคลื่อนของ UART ไม่จำเป็นต้องเป็นสองเท่าพอดี เนื่องจากการปัดเศษและโครงสร้าง divider

**แนวทางที่ถูกต้อง:** ให้ clock จริง, RTL configuration, SDC และข้อมูลความถี่ที่ software ใช้สอดคล้องกัน หากมีหลาย clock domain ต้องกำหนดแยกแต่ละ domain

### 4. เพราะเหตุใด UART RX ที่ไม่ใช้งานจึงไม่ควร tie low?

UART แบบปกติซึ่งไม่กลับขั้วมีระดับ **idle เป็น logic 1** และใช้การเปลี่ยนลงเป็น logic 0 เพื่อเริ่ม start bit

การ tie RX เป็น 0 จึงอาจทำให้ receiver เห็นสัญญาณเหมือนเริ่มรับข้อมูลตลอดเวลา หรือเกิดเงื่อนไข break/framing error ตาม implementation ส่งผลให้มี error status หรือ interrupt ที่ไม่ต้องการ

หากเปิด UART receiver แต่ไม่ใช้งาน RX ควรกำหนดให้รับค่า **logic 1**

- ที่ระดับ core: ใช้ค่าคงที่ `'1'` ตามโครงสร้าง RTL
- ที่ขาภายนอก: จัดให้ขามีระดับ idle ที่แน่นอน เช่น ใช้ pull-up ที่เหมาะสม
- หากไม่ใช้ UART ทั้งชุด: พิจารณาปิด peripheral ผ่าน configuration

การผูกค่าคงที่ใน ASIC ต้องให้ flow จัดการ tie cell ตามข้อกำหนดของ standard-cell library ด้วย

### 5. FPGA memory initialization กับ ASIC power-up memory contents ต่างกันอย่างไร?

| ประเด็น | FPGA | ASIC |
|---|---|---|
| การกำหนดค่าเริ่มต้น | FPGA และเครื่องมือหลายตระกูลรองรับการใส่ค่าเริ่มต้นของ memory ใน configuration bitstream | SRAM ทั่วไปไม่มีการรับประกันค่าเริ่มต้นหลังเปิดไฟ |
| RTL initialization | อาจถูกแปลงเป็นข้อมูล initialization ของ block RAM หรือ distributed memory | ไม่ควรสมมติว่าจะกลายเป็นการ initialize SRAM จริง |
| การเตรียมข้อมูลก่อนใช้งาน | อาจพร้อมหลัง configuration ตามที่อุปกรณ์รองรับ | ต้องมี boot/loading/clearing mechanism หรือใช้ memory ที่มีข้อมูลกำหนดไว้ เช่น ROM |

ตัวอย่างเช่น การเห็น instruction memory มีโปรแกรมตั้งแต่เริ่ม simulation ไม่ได้พิสูจน์ว่า ASIC SRAM จะมีโปรแกรมนั้นหลัง power-up

สำหรับ ASIC ต้องกำหนดกระบวนการชัดเจน เช่น

1. CPU เริ่มจาก boot ROM
2. Boot code โหลดโปรแกรมเข้า SRAM
3. Software กำหนดค่า data memory ที่จำเป็นก่อนอ่านใช้งาน

**การ reset CPU ไม่ได้หมายความว่า SRAM ถูกล้างเป็นศูนย์** เว้นแต่มีวงจรหรือขั้นตอนที่ทำเช่นนั้นจริง

### 6. ถ้า SRAM macro มีขนาดตรงกับ RTL แต่ read latency ต่างกัน สามารถแทนกันโดยตรงได้หรือไม่?

**โดยทั่วไปแทนกันโดยตรงไม่ได้** เพราะขนาดตรงกันเป็นเพียงเงื่อนไขด้านความจุ แต่ interface ยังต้องตรงกันด้านเวลาและพฤติกรรมด้วย

ตัวอย่างเช่น RTL memory ให้ข้อมูลใน cycle เดียวกับที่ส่ง address แต่ macro ให้ข้อมูลหลังรับ address ที่ขอบ clock ถัดไป หากต่อแทนทันที CPU หรือ bus controller อาจรับข้อมูลเก่าหรือรับข้อมูลผิด transaction

สิ่งที่ต้องตรวจอย่างน้อย ได้แก่

- Read latency และขอบ clock ที่รับคำสั่ง
- จำนวน port และจำนวนคำสั่งที่รับได้ต่อ cycle
- Polarity ของ chip enable และ write enable
- Byte-write mask
- Read-during-write behavior
- พฤติกรรมของ output เมื่อไม่ได้ enable

วิธีแก้คือใช้ **memory adapter** และปรับ handshake หรือ wait state ให้รับข้อมูลเมื่อ valid จริง หาก interface เดิมไม่รองรับการรอ อาจต้องแก้ microarchitecture เพิ่มเติม

### 7. เหตุใดต้องตรวจ pad placement configuration ควบคู่กับ padframe RTL?

เพราะทั้งสองส่วนอธิบายชิปคนละมิติ และต้องอ้างถึง instance ชุดเดียวกันอย่างสอดคล้องกัน

| ส่วน | สิ่งที่กำหนด |
|---|---|
| Padframe RTL | Pad cell ที่ instantiate, ชื่อ instance, ทิศทางสัญญาณ และการเชื่อมต่อ |
| Pad placement configuration | ตำแหน่งด้านของชิป ลำดับ pad และข้อกำหนดการวางตาม flow |

หากไม่ตรวจร่วมกัน อาจเกิดปัญหา เช่น

- Configuration อ้างชื่อ instance ที่ไม่มีอยู่ใน netlist
- Pad บางตัวตกหล่นหรือถูกระบุซ้ำ
- ลำดับขาไม่ตรงกับ package pinout
- จำนวน pad มากกว่าพื้นที่ด้านที่กำหนด
- ใช้ชนิด pad หรือ power domain ไม่ตรงกับการเชื่อมต่อจริง

ผลอาจเป็น flow หยุดทำงาน หรือสร้าง layout สำเร็จแต่ pinout ไม่ตรงกับที่ต้องการ

**เกณฑ์ตรวจรับ:** ตรวจความครบถ้วนของ pad instances, ชื่อ, ชนิด, ด้าน, ลำดับ, power connectivity และความสอดคล้องกับ package pinout

### 8. ต้องเก็บข้อมูลใดบ้างจึงพิสูจน์ได้ว่า generated Verilog มาจาก configuration ที่ระบุจริง?

ต้องมีหลักฐานเชื่อมโยงจาก **source → configuration → generation process → output** อย่างตรวจสอบย้อนกลับได้

| ข้อมูลที่ต้องเก็บ | จุดประสงค์ |
|---|---|
| Repository URL และ commit SHA | ระบุ source revision |
| Submodule revisions และ local changes | ระบุ source ที่ใช้จริงให้ครบ |
| Top-level และ wrapper source | ยืนยันจุดเริ่ม elaboration และการส่งค่า generics |
| ค่า generics/parameters ที่ resolve แล้ว | ยืนยัน configuration ที่มีผลจริง รวมค่า default |
| Source file list และลำดับ compile | ระบุไฟล์ที่เครื่องมืออ่านจริง |
| เวอร์ชันเครื่องมือและ dependencies | ระบุสภาพแวดล้อมการสร้าง |
| คำสั่งและ build scripts | ทำซ้ำขั้นตอน generation ได้ |
| Generation log และ exit status | ตรวจว่ากระบวนการสำเร็จและมี warning ใด |
| Generated Verilog และ checksum | ระบุ output ที่ส่งมอบอย่างแน่นอน |
| รายงานตรวจโครงสร้าง | ตรวจว่า peripheral, memory และองค์ประกอบสำคัญตรงกับ configuration |

หากมี boot image หรือไฟล์ initialization เป็น input ต้องเก็บไฟล์และ checksum ด้วย

**Checksum ของ Verilog เพียงอย่างเดียวพิสูจน์ได้ว่าไฟล์ตรงกัน แต่ยังพิสูจน์ไม่ได้ว่าสร้างจาก configuration ใด** หลักฐานที่แข็งแรงควรรวม manifest, build log และการสร้างซ้ำในสภาพแวดล้อมที่ควบคุมได้ พร้อมตรวจโครงสร้างของผลลัพธ์ด้วย