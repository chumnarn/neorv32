**เฉลยคำถามท้าย Lab 11 — โปรแกรม C, UART/GPIO และ Application Test**

คำตอบข้อ 7 อิงกรณี wrapper ที่แยก GPIO input/output และข้อ 9 อิง boot mode 2 ที่นำ firmware image ไปสร้างเป็น ROM logic สำหรับ ASIC

**1. ทำไม `riscv64-unknown-elf-gcc` สามารถสร้างโปรแกรมให้ RV32 ได้?**

เพราะชื่อ prefix ไม่ได้จำกัดว่า compiler ต้องสร้าง RV64 เท่านั้น Toolchain ที่รองรับ RV32 และมี libraries ที่เข้ากันได้สามารถเลือก target ผ่าน `-march` และ `-mabi` เช่น:

```bash
riscv64-unknown-elf-gcc -march=rv32i -mabi=ilp32 ...
```

ต้องเลือก ISA ให้ตรงกับ CPU จริง และตรวจ multilib ที่ติดตั้งไว้:

```bash
riscv64-unknown-elf-gcc -print-multi-lib
riscv64-unknown-elf-readelf -h firmware.elf
```

Toolchain แบบ multilib สามารถรองรับทั้ง RV32 และ RV64 แม้ใช้ prefix `riscv64-unknown-elf-` แต่ไม่ได้แปลว่าทุก installation จะมี RV32 libraries ครบเสมอ [GitHub](https://github.com/riscv-collab/riscv-gnu-toolchain?utm_source=chatgpt.com)

**2. ทำไม ELF size ทั้งไฟล์ไม่ใช่ ROM usage?**

เพราะ ELF มีข้อมูลนอกเหนือจากโปรแกรมที่ต้องเก็บใน ROM เช่น headers, symbol tables, debug information และ section metadata

ROM usage ต้องพิจารณา **loadable contents ที่ถูกบรรจุใน ROM จริง** ได้แก่ code, read-only data, ค่าเริ่มต้นของ `.data` รวมถึง alignment และ padding ตาม linker/image-generation flow

ควรตรวจร่วมกันจาก:

```bash
riscv64-unknown-elf-size -A firmware.elf
riscv64-unknown-elf-readelf -l firmware.elf
riscv64-unknown-elf-objdump -h firmware.elf
```

พร้อม linker map และ final ROM image ไม่ใช้ขนาดไฟล์จาก `ls -l` เพียงอย่างเดียว

**3. ทำไม `.data` ใช้ทั้ง ROM load space และ RAM runtime space?**

เพราะ `.data` เก็บตัวแปรที่มีค่าเริ่มต้นและแก้ไขได้ เช่น:

```c
uint32_t counter = 123;
```

ระบบแบบ ROM-based ต้องเก็บค่า `123` ไว้ใน firmware image เพื่อให้ยังอยู่หลังปิดไฟ แล้ว startup code คัดลอกไปยัง RAM ก่อนเข้า `main()`

จึงมีสอง address:

- **LMA — Load Memory Address:** ตำแหน่งเก็บค่าเริ่มต้นใน ROM
- **VMA — Virtual Memory Address:** ตำแหน่งที่ตัวแปรใช้งานจริงใน RAM

GNU linker รองรับการกำหนด load address ต่างจาก runtime address สำหรับรูปแบบนี้โดยตรง [sourceware.org](https://sourceware.org/binutils/docs/ld.html?utm_source=chatgpt.com)

ส่วน `.bss` โดยทั่วไปจอง RAM และให้ startup code เคลียร์เป็นศูนย์ จึงไม่ต้องเก็บข้อมูลศูนย์ทุกไบต์ใน ROM image

**4. ทำไม linker ผ่านยังไม่รับรอง stack safety?**

Linker ตรวจการจัดวาง sections ตาม linker script แต่โดยทั่วไปไม่ทราบ stack usage สูงสุดระหว่างรันจริง ซึ่งขึ้นกับ:

- Call depth และ recursion
- Local arrays และ variable-length arrays
- Library functions ที่เรียกใช้
- Interrupt context และ nested interrupts
- Heap growth และเส้นทางการทำงานจริง

แม้ linker script จอง stack ไว้ ก็ยังไม่พิสูจน์ว่าโปรแกรมใช้ไม่เกินพื้นที่นั้น

ต้องประเมิน call graph, compiler stack-usage reports เช่น `-fstack-usage` และ interrupt overhead ร่วมกับ stack watermark/canary tests โดยการทดสอบ runtime ครอบคลุมได้เฉพาะสถานการณ์ที่รันทดสอบ

**5. ทำไม software division helper จึงไม่เท่ากับเปิด RISC-V M extension?**

Software helper เช่น `__divsi3` หรือ `__udivsi3` เป็น routine ที่คำนวณการหารด้วยชุดคำสั่งที่ target รองรับ ส่วน M extension เป็นความสามารถของ CPU สำหรับคำสั่ง multiply/divide

ดังนั้น CPU แบบ RV32I สามารถหารในภาษา C ได้ผ่าน software routine โดยไม่ต้องมี hardware divider หาก helper ถูก build ด้วย ISA ที่เข้ากันได้

ผลที่ต่างกันคือ latency, code size และประสิทธิภาพ ไม่ใช่ความหมายของ operation ในภาษา C

ต้องตรวจ disassembly ของ executable สุดท้ายด้วย เพราะ helper จาก library ที่ build ผิด ISA อาจมีคำสั่งที่ CPU ไม่รองรับได้

**6. ทำไมเปลี่ยน `CLOCK_PERIOD` เพียงอย่างเดียวจึงทำให้ UART baud ผิดได้?**

ต้องแยกสองกรณี:

- **เปลี่ยนเฉพาะ SDC `CLOCK_PERIOD`:** ไม่ได้เปลี่ยน clock จริงหรือ UART baud โดยตรง แต่ทำให้ timing target ไม่ตรงกับสมมติฐานระบบ
- **เปลี่ยน clock จริงหรือ testbench period ตาม SDC แต่ไม่แก้ข้อมูลความถี่ที่ firmware ใช้:** UART divisor จะคำนวณผิด

NEORV32 ใช้ `CLOCK_FREQUENCY` ระบุความถี่ของ `clk_i` จึงต้องรักษาความสอดคล้องกับ clock จริงและข้อมูลที่ software ใช้ตั้ง UART [GitHub](https://github.com/stnolting/neorv32/blob/main/docs/datasheet/soc.adoc?utm_source=chatgpt.com)

ตัวอย่าง firmware คำนวณ divisor สำหรับ 50 MHz แต่ clock จริงเป็น 100 MHz จะได้ baud ประมาณสองเท่าของที่ตั้งใจ โดยขึ้นกับการปัดค่า divisor

**7. ทำไม GPIO input register ไม่ใช่ output readback ใน wrapper นี้?**

เพราะ input register อ่านค่าจาก **เส้นทาง GPIO input** ส่วน output register ควบคุม **เส้นทาง GPIO output** การเขียน output ไม่ได้ทำให้ input อ่านค่าเดียวกันโดยอัตโนมัติ

ตัวอย่าง wrapper ที่แยกเส้นทาง:

```text
External input pad → gpio_i → input register
Output register → gpio_o → external output pad
```

ถ้า `gpio_i` ต่อกับ input pads คนละชุด หรือ tie เป็นค่าคงที่ การอ่าน input register ย่อมไม่ใช่การอ่านค่าที่เขียนออกไป

ต้องแยกเป้าหมายการทดสอบ:

- อ่าน output latch: ตรวจค่าที่ software สั่ง
- ตรวจ output pin: ตรวจค่าที่ออกสู่ภายนอก
- ทำ external loopback: ตรวจ output-to-input path

สำหรับ bidirectional pad ที่เปิด input receiver ไว้ อาจอ่านสถานะขาได้ แต่ต้องยืนยันจาก wiring, direction และ pad model จริง

**8. ทำไม UART simulation mode ไม่รับรอง serial pin behavior?**

เพราะ simulation mode อาจใช้เส้นทางเร่งการแสดงผล เช่น ส่งอักขระออก simulator console หรือไฟล์ แทนการตรวจทุก serial bit บนขา TX เอกสาร NEORV32 แนะนำโหมดนี้เพื่อให้ simulation output เร็วขึ้น [stnolting.github.io](https://stnolting.github.io/neorv32/ug/?utm_source=chatgpt.com)

การเห็นข้อความถูกต้องจึงยังไม่ยืนยัน:

- Baud rate และ bit duration
- Start/data/stop bits และ bit ordering
- Idle polarity และ pad connection
- UART RX sampling และ synchronizer
- พฤติกรรมที่ขาภายนอกของ full-chip wrapper

ต้องปิดโหมดเร่งตาม configuration ที่ใช้ แล้วให้ testbench decode TX pin และ drive RX pin ด้วย serial waveform จริง

**9. ทำไม C firmware ใหม่ต้องทำ synthesis ใหม่ใน boot mode 2?**

ใน Lab นี้ firmware ถูกแปลงเป็น IMEM initialization image และนำไปสร้างเป็น ROM logic ดังนั้น **firmware เป็นส่วนหนึ่งของ hardware implementation**

เมื่อแก้ C จึงต้องสร้างลำดับใหม่:

```text
C → ELF → memory image → generated RTL → synthesis → physical implementation
```

NEORV32 ระบุ boot mode 2 ว่า boot จาก IMEM image แต่การที่ image จะกลายเป็น ROM logic หรือ memory initialization แบบใดขึ้นกับ implementation flow [stnolting.github.io](https://stnolting.github.io/neorv32/ug/?utm_source=chatgpt.com)

สำหรับ ASIC ที่สร้าง ROM เป็น constants/mux ต้องสร้าง netlist และผล downstream ใหม่ ส่วน FPGA ที่รองรับ memory-content update อาจมีกระบวนการต่างออกไป จึงไม่ควรนำวิธีนั้นมาสมมติกับ ASIC

**10. ถ้าต้องการ UART upload หลังผลิต ต้องเปลี่ยน memory/boot architecture อะไรบ้าง?**

ต้องมีเส้นทาง boot ที่ทำงานได้หลัง power-up โดยไม่พึ่ง application image ซึ่งฝังเป็น constants อย่างเดียว เช่น:

```text
Reset → Boot ROM → รับโปรแกรมผ่าน UART
      → เขียน executable SRAM → ตรวจ image → เริ่ม application
```

สิ่งที่ต้องเตรียม ได้แก่:

- **Boot ROM:** เก็บ loader ที่มีข้อมูลแน่นอนหลังผลิต
- **Writable executable memory:** เช่น SRAM ที่ CPU fetch instructions ได้ และ loader เขียนได้
- **UART RX/TX จริง:** เชื่อม pads และตั้ง clock/baud ถูกต้อง
- **Boot selection/reset vector:** ให้เข้า loader ตามเงื่อนไขที่กำหนด
- **Loader protocol:** ตรวจขนาด address range ความครบถ้วน และ entry point ของ image
- **Memory map/linker/startup:** จัด application, `.data`, `.bss`, stack และพื้นที่ loader ให้ไม่ชนกัน

NEORV32 มี bootloader รองรับ UART upload แต่ต้องเปิดใช้ boot configuration และ hardware resources ที่ตรงกับ revision ของโปรเจกต์ [GitHub](https://github.com/stnolting/neorv32/blob/main/docs/datasheet/software_bootloader.adoc?utm_source=chatgpt.com)

**SRAM สูญเสียข้อมูลเมื่อปิดไฟ** หากต้องการให้ application ใหม่คงอยู่ ต้องเพิ่ม nonvolatile storage เช่น external flash และให้ bootloader โหลดโปรแกรมกลับเข้า SRAM หลัง reset พร้อมทดสอบ cold boot และ UART upload บน architecture นี้จริง