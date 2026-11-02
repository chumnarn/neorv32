**เฉลยคำถามท้าย Lab 10 — Routing, Extraction และ Finishing**

**1. Global routing guides ต่างจาก detailed routing geometry อย่างไร?**

| ประเด็น | Global routing guides | Detailed routing geometry |
|---|---|---|
| ลักษณะ | พื้นที่และชั้นโลหะที่แนะนำให้แต่ละ net เดินผ่าน | รูปทรง wires และ vias ที่มีตำแหน่งจริง |
| ระดับรายละเอียด | วางแผนเส้นทางและใช้ทรัพยากรโดยประมาณ | กำหนด tracks, widths, spacing และ pin connections |
| สิ่งที่ยังไม่รับรอง | การเข้าถึงทุก pin และการเดินสายถูกกฎครบ | ต้องตรวจ connectivity และ DRC ของผลที่ได้อีกครั้ง |

Guides จึงเป็นแผนสำหรับ detailed router ไม่ใช่สายโลหะสุดท้าย โดย detailed routing ยังมีขั้นตอน pin-access analysis, track assignment และ search-and-repair เพิ่มเติม [OpenROAD documentation](https://openroad.readthedocs.io/en/latest/main/src/drt/README.html?utm_source=chatgpt.com)

**2. เหตุใด total congestion ต่ำจึงยังเกิด pin-access failure ได้?**

เพราะ congestion โดยรวมกับ pin access เป็นปัญหาคนละระดับ

แม้พื้นที่ส่วนใหญ่มี routing tracks ว่าง แต่ pin บางตัวอาจอยู่ในบริเวณที่:

- Pin geometry เล็กหรือมี access points น้อย
- Cells ข้างเคียงหรือ macro obstructions บังทางเข้า
- วาง via ไม่ได้เพราะ spacing/enclosure rules
- PDN หรือสายอื่นใช้ทรัพยากรเฉพาะจุดไปแล้ว

ค่า congestion รวมจึงอาจซ่อน local hotspot ได้ ต้องตรวจ pin-access reports และ geometry รอบ pin ที่ล้มเหลวโดยตรง

**3. Router DRC ผ่านแล้วเหตุใดต้องตรวจ GDS DRC อีก?**

Router DRC ตรวจตาม geometry และชุดกฎที่ router รองรับ ส่วน GDS DRC ตรวจ layout ที่ stream-out และรวมกับ cell/macro layouts แล้วตาม rule deck ที่เลือก

GDS อาจเผยปัญหาเพิ่มเติม เช่น:

- Geometry ภายใน cells/macros หรือบริเวณขอบที่ abstract ไม่แสดงครบ
- Layer mapping หรือ library revision ไม่ตรงกัน
- Enclosure, density หรือกฎเฉพาะที่ router ไม่ครอบคลุม
- รูปทรงที่เพิ่มภายหลัง เช่น seal ring และ metal fill

ดังนั้น router DRC ผ่านเป็น milestone ของ routing แต่ยังไม่แทน final GDS DRC

**4. PVT corner ต่างจาก RC extraction corner อย่างไร?**

| Corner | สิ่งที่แปรผันหลัก | ข้อมูลที่ใช้ |
|---|---|---|
| **PVT** | Transistor process, supply voltage และ temperature | Cell timing/power models ใน Liberty |
| **RC extraction** | คุณสมบัติ interconnect เช่น metal width/thickness, resistivity และ dielectric | Extraction rules และ parasitic network เช่น SPEF |

STA ต้องจับคู่ทั้งสองด้านตาม analysis views ที่กำหนด เช่น cell corner หนึ่งกับ interconnect corner หนึ่ง ไม่ควรสมมติว่า slow-cell corner และ RC แบบใดแบบหนึ่งจะเป็น worst case สำหรับทุก path

LibreLane ระบุว่า OpenRCX ใน flow ปกติคำนึงถึง interconnect corner ส่วนอิทธิพลอื่นต่อ parasitics ขึ้นกับ extraction methodology ที่ใช้ [LibreLane Documentation](https://librelane.readthedocs.io/en/latest/usage/timing_corners.html?utm_source=chatgpt.com)

**5. เหตุใดไฟล์ SPEF มีอยู่จึงยังไม่พิสูจน์ว่า STA annotation ครบ?**

เพราะ SPEF อาจสร้างสำเร็จ แต่ STA ไม่ได้โหลด หรือโหลดแล้วจับคู่กับ netlist ได้เพียงบางส่วน เช่น:

- Net/instance names หรือ hierarchy ไม่ตรงกัน
- SPEF มาจากคนละ revision
- โหลดผิด corner หรือผิด hierarchical scope
- มี nets ที่ไม่ได้ extract หรือไม่มี parasitics ตามที่คาด
- อ่าน SPEF แล้วถูกขั้นตอนภายหลังแทนด้วย estimated parasitics

ต้องตรวจ loading logs, unmatched objects และ annotation coverage โดยแยก clock/signal nets ตามความเหมาะสม พร้อมดู timing report ว่า paths ใช้ extracted RC จริง

**Coverage สูงก็ต้องอธิบายรายการที่ขาด** ไม่ควรรับรองจากเปอร์เซ็นต์รวมอย่างเดียว

**6. Standard-cell filler ต่างจาก dummy metal fill อย่างไร?**

| ประเด็น | Standard-cell filler | Dummy metal fill |
|---|---|---|
| ตำแหน่ง | ช่องว่างใน standard-cell rows | พื้นที่โลหะที่อนุญาตให้เติม |
| จุดประสงค์ | รักษาความต่อเนื่องของ well/implant และ power rails ตาม library | ช่วยให้ metal density ตรงข้อกำหนดการผลิต |
| ลักษณะ | เป็น library cell instance | เป็น metal shapes |
| ผลต่อวงจร | โดยทั่วไปไม่มี functional logic | เปลี่ยนสภาพแวดล้อมทางไฟฟ้าและ capacitance ได้ |

Filler ไม่จำเป็นต้องเป็น decap และไม่ควรใช้สองคำนี้แทนกัน ส่วน OpenROAD metal-fill module สร้าง floating metal shapes เพื่อช่วยให้ผ่าน density rules ภายใต้ข้อกำหนด DRC [OpenROAD documentation](https://openroad.readthedocs.io/en/latest/main/src/fin/README.html?utm_source=chatgpt.com)

**7. หลัง ECO เหตุใดจึงต้อง reroute และ re-extract?**

ECO อาจเปลี่ยน cell type, connectivity, pin positions หรือเพิ่ม buffers ทำให้สายเดิมไม่สอดคล้องกับ netlist หรือมี geometry/parasitics เปลี่ยนไป

จึงต้อง:

1. Legalize placement หากมีการเปลี่ยน cells
2. Reroute เส้นทางที่ได้รับผลกระทบ
3. ตรวจ connectivity และ DRC
4. Re-extract parasitics ของ layout ที่แก้แล้ว
5. รัน STA และ verification ที่เกี่ยวข้องใหม่

อาจใช้ incremental routing/extraction ได้หาก flow รองรับและติดตามผลกระทบครบ ไม่จำเป็นต้องรื้อทุก net เสมอ แต่ **SPEF เก่าใช้รับรอง geometry ใหม่โดยอัตโนมัติไม่ได้**

**8. Post-fill geometry อาจกระทบ parasitics อย่างไร?**

Metal fill เปลี่ยนสภาพแวดล้อมรอบ signal wires จึงอาจเปลี่ยน:

- Capacitance ต่อสภาพแวดล้อมและ coupling ระหว่างตัวนำ
- Signal delay และ transition time
- Crosstalk/noise ตามโมเดลที่ใช้
- Switching power จาก capacitance ที่เปลี่ยน

ผลขึ้นกับตำแหน่ง ชั้นโลหะ ระยะห่าง และว่า fill เป็น floating หรือเชื่อมกับ net ใด

ต้องยืนยันว่า extraction methodology รับรู้ final fill geometry หรือใช้วิธีประมาณที่โครงการอนุมัติ หากเติมโลหะเฉพาะใน GDS แต่ extractor อ่าน database ก่อน fill ผล extraction นั้นยังไม่สะท้อน post-fill layout

**9. GDS เปิดดูได้ต่างจาก layout ที่พร้อม sign-off อย่างไร?**

**GDS เปิดดูได้** ยืนยันเพียงว่า viewer อ่านโครงสร้างไฟล์ได้และแสดง geometry บางส่วนได้ ไม่ได้พิสูจน์ว่าชิปถูกต้องหรือผลิตได้

Layout ที่พร้อม sign-off ต้องมีหลักฐานตาม checklist เช่น:

- ไม่มี missing cells และ layer mapping ถูกต้อง
- DRC, LVS, antenna และ density ผ่านเกณฑ์
- Final extracted timing ผ่าน พร้อม constraints/annotation coverage
- Power connectivity และ IR/EM ผ่านตามขอบเขตที่กำหนด
- Pads, bond plan และ package mapping สอดคล้องกัน
- ใช้ final netlist และ layout revision เดียวกัน
- มีรายงานและรายการ waiver ที่ตรวจสอบย้อนกลับได้

ภาพ layout ที่ดูเรียบร้อยจึงเป็นเพียงหลักฐานประกอบการตรวจ

**10. จะพิสูจน์อย่างไรว่า GDS, netlist และ SPEF เป็น design revision เดียวกัน?**

ต้องมีทั้ง **หลักฐานที่มาของไฟล์** และ **ผลตรวจความสอดคล้องของเนื้อหา**

| หลักฐาน | สิ่งที่ยืนยัน |
|---|---|
| Source commit และสถานะ working tree | RTL/scripts ต้นทางที่ใช้จริง |
| Resolved configuration และ tool/PDK versions | เงื่อนไขการสร้าง |
| Final checkpoint ID/hash | Database ต้นทางของ artifacts |
| Build logs และ dependency records | ขั้นตอนที่สร้าง GDS, netlist และ SPEF |
| SHA-256 manifest | ระบุไฟล์ที่ตรวจและส่งมอบอย่างแน่นอน |
| LVS | Layout connectivity สอดคล้องกับ final netlist ตามขอบเขตการตรวจ |
| SPEF annotation report | Parasitic names/scope จับคู่กับ final netlist ได้ |
| Final STA logs | วิเคราะห์ด้วย netlist, SPEF และ constraints ชุดที่ระบุจริง |

แนวทางที่ชัดเจนคือ freeze final database หลัง ECO/finishing แล้วสร้าง artifacts ตาม flow ที่ควบคุม dependencies พร้อมบันทึก checksums และ reports

**ชื่อไฟล์เหมือนกันหรือ timestamp ใกล้กันยังไม่เพียงพอ** และ checksum เพียงอย่างเดียวระบุว่าเป็นไฟล์ใด แต่ไม่ได้พิสูจน์ว่าไฟล์ต่างชนิดมีเนื้อหาสอดคล้องกัน จึงต้องใช้ LVS, annotation checks และ build provenance ร่วมกัน