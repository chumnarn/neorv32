**เฉลยคำถามท้าย Lab 7 — Timing Constraints และ Clock/Reset**

**1. ทำไม full-chip clock ควรถูกนิยามที่ package-side input port?**

เพราะเป็นจุดที่ clock ภายนอกเข้าสู่ขอบเขต design ทำให้ STA สามารถติดตามเส้นทางผ่าน **clock input pad → internal clock network → clock pins ของ registers** ได้ครบ

หากสร้าง clock ที่ด้าน core หลัง input pad โดยไม่จำลองส่วนที่ข้ามไป อาจทำให้ pad delay และความสัมพันธ์กับ external I/O timing หายไปจากการวิเคราะห์

อย่างไรก็ตาม ต้องกำหนดขอบเขตให้ชัดเจน: top-level port ของ die netlist มักอยู่ที่ขา pad ฝั่งภายนอก แต่ **ไม่ได้รวม package/bond-wire delay โดยอัตโนมัติ** ส่วนนี้ต้องรวมใน external timing model ตาม reference point ที่เลือก และตรวจว่า timing model ของ clock pad รองรับการส่งผ่าน clock จริง

**2. การใส่ input delay โดยรวม pad delay ซ้ำทำให้ผล STA เปลี่ยนอย่างไร?**

`set_input_delay` ควรอธิบายเวลาที่ข้อมูลมาถึง **input port ของขอบเขต STA** ส่วน pad delay ที่อยู่ภายใน netlist จะถูกคำนวณจาก timing model อีกครั้ง

หากนับ pad delay ซ้ำ:

- **Setup analysis:** ค่า arrival time ช้าเกินจริง ทำให้ setup slack แย่เกินจริง
- **Hold analysis:** หากเพิ่มซ้ำในค่า `-min` ด้วย จะทำให้ earliest arrival ช้าเกินจริง และอาจซ่อน hold violation

ตัวอย่าง external maximum delay เท่ากับ 3 ns และ internal pad delay เท่ากับ 1 ns:

\[
AT_{\text{ถูกต้อง}}=3+1=4\ \text{ns}
\]

หากกำหนด input delay เป็น 4 ns เพราะรวม pad ไว้แล้ว เครื่องมือจะได้ \(4+1=5\) ns

จึงต้องกำหนด reference point และจัดสรร delay แต่ละส่วนให้ถูกนับเพียงครั้งเดียว

**3. ทำไม `set_output_delay -min` จึงอาจเป็นค่าลบ?**

เพราะค่านี้ใช้กำหนดข้อกำหนด **hold ของอุปกรณ์รับภายนอก** ไม่ได้หมายถึง propagation delay ทางกายภาพที่ติดลบ

ในกรณีง่ายที่ละเลย clock skew:

\[
D_{\text{out,min}}
=
D_{\text{external data,min}}-t_{\text{hold,receiver}}
\]

เช่น เส้นทางข้อมูลภายนอกเร็วสุด 0.2 ns และ receiver ต้องการ hold 0.8 ns:

\[
D_{\text{out,min}}=0.2-0.8=-0.6\ \text{ns}
\]

จึงกำหนดได้ว่า:

```tcl
set_output_delay -clock ext_clk -min -0.6 [get_ports data_out]
```

ค่าลบทำให้ข้อกำหนด hold เข้มขึ้น โดยต้องรักษาข้อมูลเก่าไว้นานพอก่อนเปลี่ยนค่า ทั้งนี้ระบบจริงต้องรวมความสัมพันธ์ของ external clock paths ด้วย [2026-06-05](https://docs.altera.com/r/docs/683068/18.1/intel-quartus-prime-standard-edition-user-guide/correlating-constraints-to-the-timing-report?contentId=NY5hFwza_KBQ2CYcvzf3NQ\&utm_source=chatgpt.com)

**4. Clock enable ต่างจาก generated clock อย่างไร?**

| ประเด็น | Clock enable | Generated clock |
|---|---|---|
| สิ่งที่ควบคุม | การรับข้อมูลของ register | Clock waveform ที่ไปยัง clock pins |
| Clock edges | Clock เดิมยังเดินทุก cycle | อาจหาร คูณ กลับเฟส หรือเปลี่ยน waveform |
| ตัวอย่าง | `if (enable) q <= d;` | Clock divider หรือ PLL output |
| การกำหนด timing | ใช้ clock เดิม | นิยามความสัมพันธ์กับ master clock |

การมี enable ทุก 4 cycles **ไม่ได้ทำให้ path เป็น multicycle 4 โดยอัตโนมัติ** ต้องพิสูจน์จังหวะ launch/capture และความเสถียรของข้อมูลก่อน

หากใช้ enable ควบคุม integrated clock-gating cell จะเป็นกรณี gated clock ซึ่งต้องวิเคราะห์ clock propagation และ clock-gating checks ตามโครงสร้างและเครื่องมือที่ใช้

**5. ทำไม UART RX exception จึงควรจบที่ first-stage synchronizer?**

เพราะ UART RX ภายนอกเป็น asynchronous ต่อ system clock จึงไม่มี phase relationship ที่แน่นอนสำหรับ setup/hold ที่ D pin ของ first-stage synchronizer

Exception จึงควรครอบคลุมเฉพาะเส้นทาง asynchronous ที่ตั้งใจยกเว้น เช่น:

> UART RX input → input pad/logic → D ของ first-stage synchronizer

ส่วนเส้นทาง **first-stage Q → second-stage D** ต้องยังได้รับ timing analysis เพราะเป็นเส้นทางภายใน clock domain เดียวกัน และ delay ของเส้นทางนี้มีผลต่อเวลาที่เหลือให้ metastability คลี่คลาย

การ false-path ไปถึงทุก register ใน UART อาจซ่อนปัญหาใน synchronizer และ receive logic การยกเว้น STA ยังต้องมี CDC review ยืนยันว่าไม่มีเส้นทาง bypass synchronizer

**6. Blanket reset false path ซ่อนการตรวจอะไรได้บ้าง?**

การยกเว้น reset แบบกว้างเกินไปอาจซ่อน:

- **Recovery/removal checks** ของ asynchronous reset release
- **Setup/hold checks** เมื่อ reset ถูกใช้เป็น synchronous control หรือเข้าสู่ data logic
- เส้นทาง reset ที่ bypass reset synchronizer
- การต่อ reset ผิด endpoint หรือกระจายข้าม clock domains โดยไม่เหมาะสม

Recovery/removal เป็น timing requirements ของการปล่อย asynchronous reset เมื่อเทียบกับ clock จึงไม่ควรถูกปิดทั้งหมดโดยไม่มีเหตุผลและวิธีตรวจทดแทน [UG1387](https://docs.amd.com/r/en-US/ug1387-acap-hardware-ip-platform-dev-methodology/Synchronous-Reset-vs.-Asynchronous-Reset?utm_source=chatgpt.com)

ส่วน minimum reset pulse width เป็นอีกข้อกำหนดที่ต้องตรวจแยก การมี false-path exception หรือรายงาน timing สะอาดไม่ได้รับรองว่าพัลส์ reset ยาวพอ

**7. Functional reset-inactive scenario ต่างจาก reset-release verification อย่างไร?**

| ประเด็น | Functional reset-inactive scenario | Reset-release verification |
|---|---|---|
| สมมติฐาน | Reset ถูกปล่อยแล้วและคง inactive | Reset กำลังเปลี่ยนจาก active เป็น inactive |
| เป้าหมาย | ตรวจ timing ของการทำงานปกติ | ตรวจการออกจาก reset อย่างปลอดภัย |
| ตัวอย่างวิธี | Case analysis กำหนด reset inactive | Recovery/removal analysis, RDC review และ assertions |
| สิ่งที่ไม่ยืนยัน | ความปลอดภัยระหว่างปล่อย reset | Functional behavior หลัง boot ทั้งหมด |

การกำหนด reset inactive ใน STA ช่วยเลือก functional mode แต่ตัดเหตุการณ์ reset release ออกจาก scenario นั้น จึงต้องมีหลักฐานอีกชุดสำหรับการปล่อย reset รวมถึงการ synchronize แยกตาม destination clock domain [UG906](https://docs.amd.com/r/en-US/ug906-vivado-design-analysis/Asynchronous-Reset-Synchronizer?utm_source=chatgpt.com)

**8. ทำไมลดความถี่แล้ว hold violation ยังอาจอยู่?**

เพราะ hold ของเส้นทาง same-clock แบบทั่วไปตรวจว่า **ข้อมูลใหม่มาถึงเร็วเกินไปเมื่อเทียบกับ capture edge เดียวกันหรือไม่** ไม่ได้ตรวจว่ามีเวลาพอถึง clock edge ถัดไปหรือไม่

สมการอย่างง่ายคือ:

\[
t_{cq,\min}+t_{data,\min}
\ge
t_{hold}+(t_{capture}-t_{launch})+U_{hold}
\]

สมการนี้ไม่มี clock period โดยตรง ดังนั้นการเพิ่ม period มักช่วย setup แต่ไม่แก้ hold

การแก้ hold อาจต้องเพิ่ม data-path delay หรือปรับ clock skew และ placement ทั้งนี้ generated clocks, phase shifts และ multicycle relationships ต้องพิจารณา edge relationship จริงเพิ่มเติม

**9. “ไม่มี violation” ต่างจาก “constraints ครบและถูกต้อง” อย่างไร?**

**ไม่มี violation** หมายถึง paths ที่เครื่องมือวิเคราะห์ภายใต้ constraints ปัจจุบันผ่านเกณฑ์ ส่วน **constraints ครบและถูกต้อง** หมายถึงโมเดลนั้นครอบคลุมความต้องการจริงของระบบ

รายงานอาจไม่มี violation เพราะ:

- ลืมสร้าง clock หรือสร้างผิดจุด
- ขาด I/O delays หรือกำหนดเฉพาะ max โดยไม่ตรวจ min
- มี unconstrained endpoints
- False paths ครอบคลุมกว้างเกินไป
- Case analysis ปิดเส้นทางที่ควรตรวจ
- เลือก mode/corner ไม่ครบ

จึงต้องมีทั้ง timing reports และ **constraint coverage audit** รวมถึงตรวจว่า collections ที่คำสั่งเลือกตรงกับ objects ที่ตั้งใจ และ exceptions ครอบคลุมเฉพาะ paths ที่อนุมัติ

**10. หลักฐานใดทำให้อนุมัติ multicycle path ได้?**

ต้องมีหลักฐานจากพฤติกรรมของวงจรว่าปลายทาง **ไม่จำเป็นต้องรับข้อมูลใน cycle ปกติที่ถูกยกเว้น** และข้อมูลสำหรับ capture ที่ตั้งใจยังถูกต้อง

หลักฐานควรประกอบด้วย:

1. **Specification/protocol:** ระบุ launch edge, capture edge และจำนวน cycles ที่อนุญาต
2. **RTL/control review:** แสดง enable, handshake หรือ state sequencing ที่บังคับพฤติกรรมนั้นจริง
3. **Verification:** Assertions, formal proof หรือการทดสอบตามขอบเขตที่เหมาะสม ยืนยันว่าปลายทางไม่รับข้อมูลก่อนเวลาและข้อมูลไม่ถูกเขียนทับผิดจังหวะ
4. **Exact path scope:** ระบุ startpoints/endpoints ชัดเจน ไม่มี single-cycle paths ปะปน
5. **STA reports:** ยืนยัน setup และ hold edge relationships หลังใช้ exception ตรงกับเจตนา

สำหรับ same-clock กรณีมาตรฐาน มักใช้ setup \(N\) คู่กับ hold \(N-1\) เพื่อคง hold relationship เดิม แต่ต้องตรวจ `-start`/`-end` และรายงาน edges จริง ไม่ควรนำสูตรไปใช้กับทุก clock relationship โดยอัตโนมัติ [2026-10-01](https://docs.altera.com/r/docs/683068/18.1/quartus-prime-standard-edition-user-guide-timing-analyzer/relaxing-setup-with-multicycle-set_multicyle_path?contentId=ckImEtTccC~b8kRTo6xrWA\&utm_source=chatgpt.com)

**การพบ setup violation หรือเห็น combinational logic ยาว ไม่ใช่หลักฐานเพียงพอสำหรับอนุมัติ multicycle path**
