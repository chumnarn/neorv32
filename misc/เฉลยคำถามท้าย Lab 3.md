**เฉลยคำถามท้าย Lab 3 — Export VHDL เป็น Verilog**

**1. เหตุใด export สำเร็จจึงยังไม่เท่ากับ design ถูกต้อง?**

Export สำเร็จหมายความว่าเครื่องมือสามารถแปลง design ที่เลือกเป็น Verilog ได้ แต่ยังไม่ได้ยืนยันว่าเลือก top, generics, source files และ firmware ถูกต้อง หรือวงจรทำงานตาม specification

จึงต้องตรวจต่อทั้ง syntax, hierarchy, ports, memory และ functional simulation โดยเฉพาะการบูตและผลลัพธ์ UART ที่คาดหวัง การแปลงวงจรที่ตั้งค่าผิดก็สามารถสำเร็จได้เช่นกัน

**2. VHDL generics ที่ถูก resolve แล้วต่างจาก Verilog parameters อย่างไร?**

VHDL generics ใช้กำหนด configuration ระหว่าง elaboration เช่น ขนาดหน่วยความจำและการเปิดใช้ peripheral เมื่อ export เครื่องมืออาจนำค่าเหล่านี้ไปสร้างวงจรเฉพาะ configuration นั้นแล้ว เช่น คลี่ `generate` และตัด logic ที่ไม่ได้ใช้ออก

ส่วน Verilog parameters ที่ยังอยู่ใน module สามารถถูก override ระหว่าง elaboration ฝั่ง Verilog ได้ ดังนั้น **อย่าสมมติว่า generics เดิมจะกลายเป็น parameters ที่ปรับได้ทุกตัว** ต้องตรวจ generated HDL จริง หากค่าถูกฝังไปแล้ว ต้อง export ใหม่เมื่อเปลี่ยน configuration

**3. เพราะเหตุใดต้องใช้ wrapper เดียวกันใน VHDL smoke และ export?**

เพื่อให้วงจรที่ทดสอบกับวงจรที่ export มี configuration และการเชื่อมต่อเดียวกัน ทั้ง generics, reset polarity, clock assumptions, UART, memory และค่า tie-off ของ input ที่ไม่ได้ใช้

ถ้าใช้คนละ wrapper ผล VHDL smoke ที่ผ่านอาจเป็นหลักฐานของวงจรคนละชุดกับ generated Verilog จึงนำมาอ้างยืนยันกันโดยตรงไม่ได้ ทั้งนี้ testbench รอบ wrapper สามารถต่างกันได้ตามภาษาที่ใช้จำลอง

**4. การรวม stderr ลงใน generated HDL มีผลอย่างไร?**

ข้อความ warning, diagnostic หรือ error อาจปะปนเข้าไปในไฟล์ Verilog ทำให้ parser อ่านไม่ผ่าน หรือทำให้ไฟล์ผลลัพธ์ไม่ใช่ HDL ที่สะอาดและตรวจสอบได้

ตัวอย่างการแยกช่องทาง output:

```bash
ghdl --synth --std=08 --out=verilog asic_wrapper \
  > generated.v 2> export.log
```

ต้องตรวจ exit status และเนื้อหาไฟล์ด้วย เพราะการ redirect สามารถสร้างไฟล์ว่างหรือไฟล์ที่เขียนไม่ครบได้ แม้คำสั่ง export ล้มเหลว ห้ามใช้เพียงเงื่อนไขว่า “มีไฟล์ `generated.v`” เป็นเกณฑ์ผ่าน

**5. Constant ROM initialization ต่างจาก RAM power-up initialization อย่างไร?**

| ประเด็น | Constant ROM | RAM power-up initialization |
|---|---|---|
| ความหมาย | กำหนดข้อมูลคงที่ที่อ่านได้ตาม address | กำหนดสถานะเริ่มต้นของหน่วยความจำที่เขียนได้ |
| การสร้างใน ASIC | อาจเป็น combinational logic หรือ ROM macro ที่รองรับเนื้อหานั้น | SRAM ทั่วไปไม่รับประกันข้อมูลเริ่มต้น เว้นแต่ macro ระบุไว้ |
| สิ่งที่ต้องตรวจ | ข้อมูล ROM ถูกเก็บไว้และอ่านถูกต้องหลัง synthesis/mapping | มีวิธีโหลดหรือเขียนข้อมูลก่อนอ่าน เช่น bootloader หรือ startup code |

การเห็น RAM มีค่าเริ่มต้นถูกต้องใน simulation จึงยังไม่ยืนยันว่า SRAM บนชิปจริงจะมีค่าเดียวกันหลังเปิดไฟ ส่วน ROM ก็ต้องตรวจว่า flow สร้างเนื้อหานั้นเป็น hardware ได้จริง

**6. ทำไมไม่ควรลบ `initial` blocks ทั้งหมดโดยอัตโนมัติ?**

เพราะ `initial` แต่ละ block อาจมีหน้าที่ต่างกัน เช่น กำหนดข้อมูล ROM, โหลด memory image, กำหนดค่าเริ่มต้นของ register หรือทำ simulation checks การลบทิ้งทั้งหมดอาจทำให้ firmware หายหรือเปลี่ยนพฤติกรรมวงจรโดยไม่ตั้งใจ

ต้องจำแนกหน้าที่และตรวจการรองรับของ target flow ก่อน หากเป็น RAM initialization ที่ ASIC ไม่รองรับ ควรแก้ด้วยกลไก boot หรือ initialization ที่สร้างเป็น hardware ได้ ไม่ใช่ลบข้อความเพื่อให้ synthesis ผ่านอย่างเดียว

**7. Port check และ structural check พิสูจน์อะไร และยังไม่พิสูจน์อะไร?**

| การตรวจ | สิ่งที่ยืนยันได้ภายในขอบเขตการตรวจ | สิ่งที่ยังยืนยันไม่ได้ |
|---|---|---|
| Port check | ชื่อ ทิศทาง และความกว้างของ interface ตรงตามที่คาด | ความหมายของสัญญาณและพฤติกรรมถูกต้อง |
| Structural check | hierarchy และการเชื่อมต่อผ่านกฎที่ตรวจ เช่น ไม่มี unresolved module หรือ multiple drivers ที่ผิดกฎ | การคำนวณ การบูต firmware และ protocol ถูกต้อง |
| ทั้งสองร่วมกัน | design มี interface และโครงสร้างพร้อมสำหรับขั้นตอนถัดไป | timing closure, physical correctness หรือ equivalence ทั้งหมด |

ตัวอย่างเช่น ต่อ UART TX เข้ากับสัญญาณผิดเส้นแต่มีความกว้างถูกต้อง อาจผ่าน port check ได้

**8. ROM smoke ที่ตรงกันถือเป็น formal equivalence หรือไม่?**

**ไม่ถือเป็น formal equivalence** เพราะ smoke simulation ตรวจเพียง stimulus และช่วงเวลาที่กำหนด เช่น reset แล้วรอ startup marker ผ่าน UART

ผลที่ตรงกันเป็นหลักฐานว่า VHDL และ generated Verilog ให้ผลตรงกันในกรณีที่ทดสอบ แต่ยังไม่ครอบคลุมทุก input sequence และทุกสถานะที่เกี่ยวข้อง ส่วน formal equivalence ต้องใช้กระบวนการพิสูจน์ภายใต้สมมติฐานและการจับคู่ design ที่ระบุชัดเจน

**9. หากเปลี่ยน memory size ต้องสร้างและตรวจ artifacts ใดใหม่?**

ต้องตรวจผลกระทบทั้ง hardware และ software โดยอย่างน้อยมีรายการต่อไปนี้:

- **Configuration และ wrapper:** ขนาด memory, address range, address decoding และข้อกำหนด alignment
- **Software artifacts:** linker script หรือ linker settings, ELF, map file และ binary หาก memory layout ที่ software ใช้เปลี่ยน
- **Memory images:** ROM/RAM image และไฟล์ VHDL หรือ Verilog ที่สร้างจาก image ตาม flow
- **Exported hardware:** elaborate และ export Verilog ใหม่ รวมทั้งปรับ memory adapter หรือ macro mapping หากได้รับผลกระทบ
- **Verification artifacts:** อัปเดต expected values และทดสอบ boot, ขอบเขต address, byte enables และการ alias ของ address
- **Implementation results:** รัน synthesis และ STA ใหม่ รวมถึง physical flow หากต้องประเมิน implementation ใหม่

ต้องตรวจด้วยว่า program, data, heap และ stack ยังอยู่ภายในพื้นที่ที่กำหนด และ memory macro มีทั้งความจุ ความกว้าง และ latency ตรงกับ interface

**10. ต้องเก็บข้อมูลใดจึงตรวจสอบที่มาของ generated Verilog ได้ครบ?**

ควรเก็บข้อมูลให้สามารถย้อนจาก generated Verilog กลับไปยัง source และสร้างซ้ำได้ ได้แก่:

| กลุ่ม | ข้อมูลที่ควรเก็บ |
|---|---|
| Source | Repository URL, commit, submodule revisions และ local patch หรือสถานะ working tree |
| Input files | รายชื่อไฟล์ ลำดับการอ่าน library mapping และ checksum ของ input |
| Configuration | Top entity, wrapper, generic values, VHDL standard และ options ที่ใช้ |
| Toolchain | เวอร์ชัน GHDL, backend, plugin ที่เกี่ยวข้อง และ environment หรือ dependency lock |
| Firmware | Source revision, compiler version, build flags, linker settings และ checksum ของ ELF/memory image |
| Execution | คำสั่งหรือ script ที่รัน, stdout/stderr logs และ exit status |
| Output | Generated Verilog พร้อม checksum และรายละเอียด post-processing ทุกขั้น |
| Validation | ผล port/structural checks และ VHDL/Verilog smoke พร้อม testbench และเกณฑ์ PASS |

วันที่สร้างและชื่อไฟล์ช่วยจัดระเบียบ แต่ **commit, configuration, toolchain และ checksums** เป็นหลักฐานสำคัญในการยืนยันที่มาและความสามารถในการสร้างซ้ำ