**เฉลยคำถามท้าย Lab — VHDL Analysis, Elaboration และ ROM Smoke Test**

**1. เพราะเหตุใด `ghdl -i` ผ่านจึงยังไม่เท่ากับ VHDL analysis ผ่าน?**

`ghdl -i` เป็นขั้นตอน **import** โดยอ่านและ parse ไฟล์เพื่อบันทึกว่า design units เช่น entity, architecture และ package อยู่ในไฟล์ใด แต่ยังไม่ได้ทำ semantic analysis เต็มรูปแบบ จึงอาจยังไม่ตรวจพบปัญหาเกี่ยวกับชนิดข้อมูล การอ้างอิงชื่อ หรือ dependencies บางประเภท

ดังนั้น ต้องใช้ `ghdl -a` หรือกระบวนการ build ที่เรียก analysis จริงก่อน จึงจะยืนยันว่า analysis ผ่านได้ [7.0.0-dev](https://ghdl.github.io/ghdl/using/InvokingGHDL.html?utm_source=chatgpt.com)

**2. Analysis ผ่านแล้ว แต่ elaboration ล้มเหลวได้จากสาเหตุใด?**

Analysis ตรวจแต่ละ design unit ส่วน elaboration นำ units มาประกอบเป็น hierarchy ตาม top-level และค่าคอนฟิกที่เลือก ปัญหาจึงอาจปรากฏเมื่อประกอบระบบ เช่น

- หา entity หรือ architecture ที่ต้องใช้ไม่พบ หรือ component binding ไม่สมบูรณ์
- ค่า generic ทำให้ขนาด port, array หรือช่วง index ไม่สอดคล้องกัน
- เงื่อนไข `generate` เลือกเปิดส่วนวงจรที่มีปัญหา
- ค่าเริ่มต้นหรือค่าคงที่ละเมิด range constraint
- ขาด simulation model ของ memory หรือ IP ที่ instantiate ไว้

ต้องอ่าน warning ด้วย เพราะบางกรณี เช่น unbound component อาจไม่ได้ทำให้เครื่องมือหยุดทันที ทั้งนี้ การตรวจบางส่วนอาจเกิดตอนเริ่มรัน ขึ้นกับ backend และวิธีเรียก GHDL

**3. ทำไมต้องใช้ `--std=08` และ library settings ให้ตรงกันทุก stage?**

`--std=08` ระบุให้ใช้กติกา VHDL-2008 ซึ่งจำเป็นเมื่อ source หรือ testbench ใช้คุณสมบัติของมาตรฐานนี้ เช่น `process(all)` หรือ `std.env.finish` ส่วน library settings กำหนดว่าเครื่องมือต้องบันทึกและค้นหา design units จากที่ใด

จึงต้องใช้ standard และ mapping ของ `--work`, `--workdir` และ `-P` ให้สอดคล้องตั้งแต่ analysis ถึงการรัน มิฉะนั้นอาจหา unit ไม่พบหรือใช้ compiled units คนละชุด สำหรับโครงการหลาย libraries ค่า `--work` อาจต่างกันตามไฟล์ได้ แต่ต้องตรงกับ library ที่ตั้งใจใช้งาน [7.0.0-dev](https://ghdl.github.io/ghdl/quick_start/simulation/index.html?utm_source=chatgpt.com)

**4. ถ้า UART มี activity แต่ decode ไม่ได้ ควรตรวจอะไรเป็นอันดับแรก?**

**วัดความกว้างของหนึ่งบิตบน TX แล้วเทียบกับ baud rate ที่ decoder ตั้งไว้** โดยตรวจ clock จริงของ testbench ควบคู่กัน

ตัวอย่าง UART 115200 baud มีระยะเวลาหนึ่งบิตประมาณ

\[
T_{\text{bit}}=\frac{1}{115200}\approx 8.681\ \mu s
\]

จากนั้นตรวจว่า decoder ต่อกับ TX ถูกเส้นหรือไม่, idle เป็น high หรือไม่ และรูปแบบ frame ตรงกันหรือไม่ เช่น 8 data bits, no parity, 1 stop bit หรือ **8N1** ระวังช่วงเริ่ม simulation และ reset ซึ่งอาจสร้าง edges ที่ไม่ใช่ UART frame

**5. เพราะเหตุใดการพบ startup marker จึงเป็นหลักฐานที่ดีกว่าการพบ TX edges?**

TX edges ยืนยันเพียงว่าสัญญาณเปลี่ยนระดับ ซึ่งอาจเกิดจาก reset, ค่าไม่ทราบสถานะ หรือโปรแกรมที่ทำงานผิด

แต่การ decode ได้ข้อความ startup marker ที่คาดไว้ เช่น `BOOT_OK` เป็นหลักฐานว่าระบบทำงานต่อเนื่องได้หลายส่วน ได้แก่ CPU ออกจาก reset, fetch และ execute firmware, เข้าถึง UART และส่งข้อมูลด้วย timing ที่ decoder รับได้

Marker จึงเป็น **หลักฐานการทำงานระดับระบบที่ชัดกว่า** แต่ยังไม่ยืนยันคำสั่ง CPU, peripherals หรือ memory ทุกตำแหน่ง ควรให้ monitor ตรวจจากขา TX ของ DUT จริง และเลือก marker ที่ระบุ firmware ของการทดสอบได้

**6. Clock จริงกับ `CLOCK_FREQUENCY` ไม่ตรงกันจะกระทบ UART อย่างไร?**

หาก firmware ใช้ค่า `CLOCK_FREQUENCY` คำนวณ baud divider แต่ clock ที่ป้อนจริงไม่ตรงกัน baud rate ที่ส่งออกก็จะผิดตามสัดส่วน โดยประมาณว่า

\[
B_{\text{actual}}
\approx B_{\text{requested}}
\frac{f_{\text{actual}}}{f_{\text{configured}}}
\]

ตัวอย่าง กำหนดความถี่ไว้ 50 MHz แต่ป้อน clock จริง 100 MHz เมื่อ firmware ตั้ง UART เป็น 115200 baud ผลที่ได้จะอยู่ประมาณ 230400 baud โดยยังมีผลจากการปัดค่า divider

นอกจากนี้ delay loops, timers และ timeout ที่อาศัยค่าความถี่เดียวกันอาจผิดไปด้วย การกำหนด `CLOCK_FREQUENCY` **ไม่ได้สร้างหรือเปลี่ยนความถี่ของ clock ที่ป้อนเข้า DUT**

**7. การถึง `--stop-time` โดยไม่มี assertion failure ถือว่าผ่านหรือไม่?**

**ยังถือว่าผ่านไม่ได้** เพราะ `--stop-time` เป็นเพียงขอบเขตเวลาที่สั่งให้ simulator หยุด ไม่ได้ตรวจว่า DUT ทำงานสำเร็จ โปรแกรมอาจค้างอยู่ใน reset หรือไม่ส่ง UART เลยก็ได้ [7.0.0-dev](https://ghdl.github.io/ghdl/using/Simulation.html?utm_source=chatgpt.com)

Testbench ต้องมีเงื่อนไขสำเร็จชัดเจน เช่น decode startup marker ครบก่อน deadline และมี watchdog ที่ทำให้การทดสอบล้มเหลวหากไม่พบ marker

หลักการตรวจรับคือ **พบผลที่คาดไว้ภายในเวลาที่กำหนด** พร้อมไม่มีข้อผิดพลาดที่เกี่ยวข้อง ไม่ใช่เพียง simulation จบโดยเงียบ ๆ

**8. Negative test ช่วยตรวจความน่าเชื่อถือของ testbench อย่างไร?**

Negative test จงใจสร้างกรณีผิดที่ควรถูกตรวจพบ เพื่อพิสูจน์ว่า checker สามารถรายงาน failure ได้จริง เช่น

| การจงใจเปลี่ยนเงื่อนไข | ผลที่ testbench ควรตรวจพบ |
|---|---|
| จับ reset ค้าง | ไม่พบ startup marker และ timeout |
| บังคับ TX เป็น high ตลอด | ไม่มี frame และ timeout |
| เปลี่ยน expected marker ให้ผิด | ไม่ผ่านการตรวจข้อความ |
| ตั้ง baud ของ decoder ให้ผิดมากพอ | Decode ผิดหรือไม่พบ marker |

หากกรณีเหล่านี้ยังรายงาน PASS แสดงว่าเกณฑ์ตรวจรับอาจไม่ผูกกับ output จริง หรือสคริปต์ไม่ได้ตรวจผลอย่างถูกต้อง ต้องแยกแต่ละ negative test ให้ชัดเจนและระบุ failure ที่คาดหวังไว้ล่วงหน้า

**9. Baseline ผ่านแต่ ASIC wrapper ไม่ผ่าน ควรเปรียบเทียบ configuration และ connections จุดใด?**

เริ่มจากทำให้ firmware และ CPU configuration เหมือน baseline แล้วเปรียบเทียบดังนี้

| จุดตรวจ | สิ่งที่ต้องเทียบ |
|---|---|
| Clock | ความถี่จริง ขา clock และ `CLOCK_FREQUENCY` |
| Reset | Polarity ระยะเวลาจับ reset และวิธีปล่อย reset |
| CPU configuration | ISA extensions และตัวเลือกที่ firmware ต้องใช้ |
| Boot configuration | Reset/boot address, bootloader และ executable image |
| Memory | ขนาด address mapping, initialization, latency และ handshake |
| UART | Enable, instance, TX/RX mapping และ RX idle high |
| Inputs ที่ไม่ใช้ | Interrupt, debug, bus response และ tie-off ที่เหมาะสม |
| Libraries และ hierarchy | Top-level, architecture และ simulation models ที่เลือก |
| Pad integration | ทิศทาง pad, output enable และเส้นทางจาก core TX ถึงขาที่ monitor |

วิธีแยกปัญหาที่ได้ผลคือ **ตรวจ UART ที่ด้าน core ก่อน แล้วตรวจหลัง wrapper/pad** หาก core ส่งถูกแต่ด้านนอกผิด ให้ไล่ connections และ pad control หาก core ไม่ส่งตั้งแต่แรก ให้ย้อนตรวจ clock, reset, boot และ memory interface

**10. ROM smoke ผ่านแล้ว เหตุใดจึงยังไม่ยืนยันว่า SRAM จะมีข้อมูลถูกต้องหลังเปิดชิปจริง?**

ROM smoke อาจใช้ ROM image และ behavioral RAM ที่ simulator กำหนดค่าเริ่มต้นไว้ แต่ SRAM บน ASIC โดยทั่วไป **ไม่รับประกันข้อมูลหลัง power-up** เว้นแต่มีคุณสมบัติหรือวงจร initialization ที่ระบุไว้ชัดเจน การ reset CPU ก็ไม่ได้หมายความว่าทุกบิตใน SRAM ถูกล้าง

แม้ marker จะพิมพ์สำเร็จ ก็ยืนยันเพียงเส้นทางและตำแหน่ง memory ที่ถูกใช้งานในการทดสอบนั้น ไม่ได้ยืนยัน SRAM ทั้งหมด

ระบบจริงจึงต้องมีขั้นตอนเตรียมข้อมูล เช่น boot code คัดลอก `.data` จากแหล่งเก็บ image ลง RAM และล้าง `.bss` ก่อนเรียกโปรแกรม C รวมถึงเขียนข้อมูลก่อนอ่านพื้นที่อื่นที่ต้องการค่าแน่นอน ส่วนการแทน behavioral RAM ด้วย SRAM macro ต้องตรวจ interface, read latency, write mask และ timing เพิ่มเติมด้วย