**เฉลยคำถามท้าย Lab 9 — Placement, CTS และ Timing Closure**

**1. เหตุใด global placement ผ่านแล้วจึงต้องทำ detailed placement?**

Global placement เน้นกระจาย cells ให้เหมาะกับ wirelength, density และ timing แต่ตำแหน่งที่ได้อาจยังมี cells ซ้อนกันหรือไม่ตรง legal placement sites

Detailed placement จึงต้องจัด cells ให้ลง rows/sites ที่ถูกต้อง ไม่ overlap และเคารพ placement constraints โดยพยายามขยับจากตำแหน่งเดิมให้น้อยที่สุด นอกจากนี้ยังใช้ทำ legalization ซ้ำหลัง resizing หรือ buffer insertion ได้ด้วย [OpenROAD documentation](https://openroad.readthedocs.io/en/latest/main/src/dpl/README.html?utm_source=chatgpt.com)

ดังนั้น global placement ผ่านยังไม่เท่ากับ placement ถูกกฎครบ และ detailed placement ผ่านก็ยังไม่ยืนยันว่า routing จะผ่าน

**2. เหตุใด timing ก่อน CTS และหลัง CTS จึงต่างกัน?**

ก่อน CTS มักใช้ **ideal clock หรือ estimated clock model** จึงยังไม่เห็น clock tree ที่สร้างจริง ส่วนหลัง CTS มี clock buffers และเส้นทาง clock ไปยังแต่ละ register ทำให้เห็น insertion delay, skew และ clock slew ชัดขึ้น

นอกจากนี้ CTS และ timing repair อาจเพิ่ม cells หรือเปลี่ยนตำแหน่งจน data-path delay เปลี่ยนด้วย

การเปรียบเทียบจึงต้องตรวจว่า:

- ใช้ ideal หรือ propagated clocks
- ใช้ parasitics แบบใด
- Constraints, corners และ uncertainty สอดคล้องกันหรือไม่
- มี netlist/placement changes ระหว่างสองรายงานหรือไม่

ผลหลัง CTS อาจดีขึ้นหรือแย่ลงได้ ไม่ควรสมมติว่า CTS จะทำให้ timing ดีขึ้นทุก path

**3. Positive skew ช่วย setup แต่กระทบ hold อย่างไร?**

กำหนดให้:

\[
S=t_{\text{capture clock}}-t_{\text{launch clock}}
\]

Positive skew หมายถึง capture clock มาถึงช้ากว่า launch clock สำหรับเส้นทาง same-clock ทั่วไป:

\[
Slack_{\text{setup}}
\approx T+S-t_{cq,\max}-t_{data,\max}-t_{setup}-U_{setup}
\]

\[
Slack_{\text{hold}}
\approx t_{cq,\min}+t_{data,\min}-S-t_{hold}-U_{hold}
\]

ดังนั้น positive skew:

- **ช่วย setup:** ปลายทางมีเวลารับข้อมูลเพิ่ม
- **ทำให้ hold ยากขึ้น:** ข้อมูลใหม่อาจมาถึงเร็วเกินไปเมื่อเทียบกับ capture clock ที่มาช้า

ตัวอย่าง skew เพิ่มขึ้น 0.2 ns จะช่วย setup ประมาณ 0.2 ns แต่ลด hold slack ประมาณ 0.2 ns หากปัจจัยอื่นคงเดิม ทั้งนี้ผลจริงต้องพิจารณา early/late clock paths, variation และ CPPR ด้วย

**4. เหตุใดเพิ่ม clock period จึงไม่แก้ same-edge hold violation?**

เพราะ same-edge hold ตรวจว่าข้อมูลใหม่เปลี่ยนเร็วเกินไปหลัง **clock edge เดียวกัน** หรือไม่ ขณะที่ clock period กำหนดระยะถึง edge ถัดไป

จากสมการ hold ในข้อ 3 ไม่มี \(T\) โดยตรง การเพิ่ม period จึงช่วย setup แต่ไม่เปลี่ยนปัญหาข้อมูลมาถึงเร็วเกินไปสำหรับ hold

แนวทางแก้คือเพิ่ม minimum data-path delay เช่น hold-buffer insertion หรือปรับ skew/placement โดยต้องตรวจ setup ซ้ำด้วย

**5. เมื่อ cell delay สูง ควรเริ่มแก้ที่ placement หรือ logic?**

**ควรเริ่มจากวิเคราะห์สาเหตุใน timing report ก่อน** เพราะ cell delay ขึ้นกับทั้งชนิด cell, input slew และ output load

| สิ่งที่พบ | จุดเริ่มแก้ที่เหมาะสม |
|---|---|
| Logic levels มากหรือมี priority chain ยาว | ทบทวน RTL, logic structure หรือ pipeline |
| Cell ขับโหลดมากเพราะ fanout สูง | พิจารณา buffering, replication หรือ sizing |
| Wire ยาวทำให้ capacitance สูง | ทบทวน placement และ routing topology |
| Input slew แย่ | ย้อนตรวจ driver และ interconnect ก่อนหน้า |
| Cell/arc ช้าแม้ slew/load เหมาะสม | ทบทวน mapping หรือเลือก cell ที่เหมาะสม |

Cell delay สูงจึงไม่ได้แปลว่าต้องแก้ logic เสมอ เพราะ placement ที่ไม่ดีอาจเพิ่มโหลดจน cell ช้าลงได้

**6. การแทรก hold buffers กระทบ setup และ congestion อย่างไร?**

Hold buffers เพิ่ม minimum delay เพื่อชะลอข้อมูลใหม่ แต่โดยทั่วไปเพิ่ม maximum delay ด้วย จึงอาจลด setup slack โดยเฉพาะ path ที่มี setup margin น้อย

ผลกระทบอื่น ได้แก่:

- เพิ่ม cell area และ power
- ใช้ placement sites และ routing resources
- เพิ่ม wires, vias และ local congestion
- ต้องทำ legalization และตรวจ electrical constraints ใหม่

ตำแหน่งแทรกสำคัญมาก หากแทรกก่อนจุดแตกแขนง อาจกระทบหลาย endpoints รวมถึง paths ที่ setup critical จึงต้องตรวจทั้ง setup และ hold หลัง repair ไม่ใช่นับเพียงจำนวน hold violations ที่ลดลง

**7. เหตุใด timing ผ่านแต่มี unconstrained endpoints จึงยังรับรองผลไม่ได้?**

เพราะ “ผ่าน” อาจหมายถึงเฉพาะ endpoints ที่มี timing requirements ครบ ส่วน unconstrained endpoints อาจไม่มี required time ให้เปรียบเทียบ จึงไม่ถูกรายงานเป็น violation แม้มีปัญหาจริง

ต้องตรวจแต่ละรายการว่าเกิดจาก:

- Clock หรือ generated clock หาย
- Output delay ขาด
- Mode/case analysis ไม่ถูกต้อง
- Exception กว้างเกินไป
- Endpoint ที่ตั้งใจไม่ตรวจ ซึ่งต้องมีเหตุผลและวิธี verification รองรับ

เกณฑ์ตรวจรับจึงควรเป็น **ไม่มี unexplained unconstrained endpoints** พร้อมหลักฐาน constraint coverage ไม่ใช่ดู WNS/TNS เพียงอย่างเดียว

**8. ผล post-CTS กับ final extracted timing ต่างกันอย่างไร?**

| ประเด็น | Post-CTS timing | Final extracted timing |
|---|---|---|
| Clock tree | สร้างแล้ว แต่ routing อาจยังไม่สมบูรณ์ | ใช้เส้นทาง clock หลัง routing |
| Signal interconnect | มักใช้ estimated parasitics | ใช้ parasitics ที่ extract จาก routed geometry |
| สิ่งที่สะท้อน | แนวโน้ม timing หลังสร้าง clock tree | ผลของ routing layers, detours และ vias ตาม extraction model |
| วัตถุประสงค์ | ชี้จุดแก้ก่อน routing | ตรวจ timing ของ implementation ขั้นสุดท้าย |

OpenROAD แยก placement/global-routing parasitic estimates ออกจาก parasitics ที่อ่านจาก SPEF จึงต้องตรวจแหล่งข้อมูลจริงของแต่ละรายงาน แม้ชื่อไฟล์จะระบุว่า “post-CTS” หรือ “final” ก็ตาม [OpenROAD documentation](https://openroad.readthedocs.io/en/latest/main/src/est/README.html?utm_source=chatgpt.com)

Final extracted timing ยังต้องใช้ netlist, constraints, extraction corner และ libraries ที่สอดคล้องกัน พร้อมยืนยัน annotation/coverage และตรวจทุก mode/corner ที่กำหนด จึงจะใช้เป็นหลักฐาน timing sign-off ได้
