## Lab 2 — VHDL analysis และ ROM smoke

Lab นี้ตรวจว่า source และ configuration ที่เลือกจาก Lab 1 สามารถสร้างแบบจำลอง NEORV32 ที่สมบูรณ์ได้ และ CPU สามารถเริ่มทำงานจาก boot ROM จนส่งข้อความออกทาง UART ได้จริง

การตรวจแบ่งเป็นสามระดับ:

| ระดับ | สิ่งที่ตรวจ | หลักฐานที่ต้องได้ |
|---|---|---|
| VHDL analysis | Syntax, types, packages และ dependencies | GHDL วิเคราะห์ source ที่ใช้ผ่าน |
| Elaboration | Hierarchy, generic values และ port bindings | สร้างแบบจำลอง top-level ได้ |
| ROM smoke | Clock/reset, boot ROM execution และ UART | Testbench อ่านข้อความที่คาดหวังได้ภายใน timeout |

**เป้าหมายหลัก:** หลังปล่อย reset ต้องได้รับข้อความที่ตรวจสอบกับ bootloader source ของ revision ที่เลือกไว้ การเห็น UART เปลี่ยนระดับเพียงอย่างเดียวยังไม่ถือว่า ROM smoke ผ่าน

ตัวอย่างใน Lab ใช้ upstream bootloader test setup เป็น baseline ก่อนนำ testbench ไปใช้กับ ASIC wrapper ของโครงการ การผ่าน baseline ต้องรายงานแยกจากการผ่าน ASIC wrapper

### 2.1 วัตถุประสงค์

เมื่อจบ Lab ผู้เรียนต้องสามารถ:

1. ตั้งค่า VHDL-2008 และ logical library ให้สอดคล้องกัน
2. อธิบายความต่างระหว่าง import, analysis, elaboration และ simulation
3. ตรวจ dependency ของ packages, entities และ memory images
4. สร้าง testbench สำหรับ clock, reset และ UART receiver
5. ตรวจ boot ROM execution ผ่านข้อความ UART
6. ใช้ waveform แยกปัญหา reset, instruction fetch และ UART timing
7. สร้างผลทดสอบที่แสดง PASS/FAIL ด้วย assertion และ timeout

### 2.2 สิ่งที่ต้องเตรียมจาก Lab 1

ต้องมีข้อมูลต่อไปนี้ก่อนเริ่ม:

- NEORV32 commit ที่กำหนดแน่นอน
- Source inventory และ compile plan
- Configuration ของ boot mode, clock, memories และ peripherals
- Boot ROM image ที่ใช้จริง
- Interface ของ top-level ที่จะทดสอบ
- ประเด็นค้างที่มีผลต่อ elaboration ได้รับการแก้แล้ว

สำหรับตัวอย่างนี้ใช้:

| รายการ | ค่าเริ่มต้นของ Lab |
|---|---|
| Simulator | GHDL |
| VHDL standard | VHDL-2008 |
| Logical library | `neorv32` |
| Baseline DUT | `neorv32_test_setup_bootloader` |
| Clock | 50 MHz |
| Clock period | 20 ns |
| External reset | Active low |
| UART RX input | High ตลอดการตรวจ startup |
| UART decoder | 8 data bits, no parity, 1 stop bit |
| Baud rate | 19,200 เมื่อใช้ default bootloader ที่ตรงกัน |
| Simulation timeout | 100 ms เป็นค่าเริ่มต้น |

เอกสาร bootloader ปัจจุบันระบุ default console เป็น 19,200 baud แบบ 8-N-1 และแนะนำ UART0, CLINT และ GPIO สำหรับ configuration มาตรฐาน ต้องตรวจค่ากับ source และ ROM image ใน revision ของโครงการอีกครั้ง [GitHub](https://github.com/stnolting/neorv32/blob/main/docs/datasheet/software_bootloader.adoc?utm_source=chatgpt.com)

---

### 2.3 ขั้นตอนที่ 1 — เตรียม directory และบันทึก environment

ใช้ workspace เดียวกับ Lab 1:

```bash
cd /path/to/neorv32-ihp

export PROJECT_ROOT="$PWD"
export NEORV32_ROOT="$PROJECT_ROOT/upstream/neorv32"
export LAB2_BUILD="$PROJECT_ROOT/build/lab02"
export LAB2_REPORT="$PROJECT_ROOT/reports/lab02"

mkdir -p "$PROJECT_ROOT/tb"
mkdir -p "$LAB2_BUILD"
mkdir -p "$LAB2_REPORT"
```

ตรวจเครื่องมือ:

```bash
command -v ghdl
ghdl --version | tee "$LAB2_REPORT/ghdl-version.txt"

git -C "$NEORV32_ROOT" rev-parse HEAD \
  > "$LAB2_REPORT/neorv32-commit.txt"

git -C "$NEORV32_ROOT" status --short \
  > "$LAB2_REPORT/neorv32-status.txt"
```

อ่าน `ghdl-version.txt` แล้วบันทึก:

- GHDL version
- Backend เช่น mcode, LLVM หรือ GCC
- Host platform
- Environment ที่ใช้ เช่น Nix shell หรือ container

Backend อาจมีผลต่อวิธีสร้าง executable และความเร็ว simulation จึงควรเก็บข้อมูลนี้ร่วมกับผลทดสอบ

**เกณฑ์ตรวจ:** หาก working tree เปลี่ยนจาก Lab 1 ต้องอธิบายว่าเปลี่ยนอะไร และมีผลต่อ hardware หรือ ROM image หรือไม่

---

### 2.4 ขั้นตอนที่ 2 — ตรวจ baseline DUT และ bootloader configuration

ตรวจไฟล์:

```bash
test -f \
  "$NEORV32_ROOT/rtl/test_setups/neorv32_test_setup_bootloader.vhd"

less \
  "$NEORV32_ROOT/rtl/test_setups/neorv32_test_setup_bootloader.vhd"
```

ค้นหาค่าที่มีผลต่อ startup:

```bash
rg -n \
  'CLOCK_FREQUENCY|BOOT_MODE_SELECT|IMEM|DMEM|RISCV_ISA|IO_GPIO|IO_CLINT|IO_UART0' \
  "$NEORV32_ROOT/rtl/test_setups/neorv32_test_setup_bootloader.vhd"
```

ตรวจให้ครบ:

1. Clock generic ถูกส่งเข้า SoC
2. Boot configuration เลือก internal bootloader
3. Instruction/data memories มี configuration ที่รองรับ boot path
4. UART0 เปิดใช้งาน
5. Peripherals ที่ bootloader ใช้เปิดใช้งาน
6. UART RX/TX และ GPIO ports ตรงกับ testbench
7. CPU configuration รองรับ instruction set ของ boot ROM image

ตัวอย่าง upstream setup มี clock/reset, UART RX/TX และ GPIO output 8 bits แต่ชื่อและ configuration ต้องตรวจจาก commit ที่ checkout จริง [GitHub](https://github.com/stnolting/neorv32/blob/main/rtl/test_setups/neorv32_test_setup_bootloader.vhd?utm_source=chatgpt.com)

ตรวจ baud rate และข้อความ startup:

```bash
rg -n \
  'UART_BAUD|UART_EN|UART_HWFC|STATUS_LED' \
  "$NEORV32_ROOT/sw/bootloader"

rg -n \
  'NEORV32|Bootloader|bootloader|printf|puts' \
  "$NEORV32_ROOT/sw/bootloader"
```

เลือกข้อความหนึ่งที่ bootloader ส่งออกแน่นอน เช่น `NEORV32` **หลังยืนยันว่ามีอยู่ใน source และ ROM image ที่ใช้**

ข้อความที่เลือกควร:

- อยู่ใน startup path ที่ทดสอบ
- ยาวพอแยกจากข้อมูลสุ่ม
- ไม่เปลี่ยนไปตาม clock หรือ revision โดยไม่จำเป็น
- ไม่ต้องรอให้ผู้ใช้ส่งคำสั่งทาง UART

บันทึกใน `boot-expectation.md`:

```markdown
# Boot expectation

- Source commit:
- Boot mode:
- Boot ROM image:
- Boot ROM image hash:
- Bootloader source/config:
- UART baud:
- UART format:
- Expected startup marker:
- Evidence for marker:
- Startup timeout:
```

หาก ROM image เป็น prebuilt image แต่ยังพิสูจน์ไม่ได้ว่าสอดคล้องกับ bootloader source ใด ให้รายงานข้อจำกัดนี้ อย่านับการอ่าน C source เพียงอย่างเดียวว่าได้ยืนยัน contents ของ ROM แล้ว

---

### 2.5 ขั้นตอนที่ 3 — ทำความเข้าใจ GHDL compilation stages

| คำสั่ง | หน้าที่ |
|---|---|
| `ghdl -i` | Import/index design units เพื่อให้ GHDL หา dependencies |
| `ghdl -a` | Analyze source ตรวจ syntax และ semantics |
| `ghdl -m` | วิเคราะห์ dependencies ที่จำเป็นและ elaborate top ที่เลือก |
| `ghdl -e` | Elaborate จาก analyzed units |
| `ghdl -r` | Run simulation |

`ghdl -i` เพียงอย่างเดียวไม่ใช่หลักฐานว่า semantic analysis ผ่าน ส่วน `ghdl -m` ช่วยจัดการ dependency ของ design ที่เข้าถึงได้จาก top ที่เลือก ตาม workflow ในเอกสาร GHDL [7.0.0-dev](https://ghdl.github.io/ghdl/using/InvokingGHDL.html?utm_source=chatgpt.com)

ใน Lab นี้ใช้ `-i` แล้ว `-m` เพื่อสร้าง baseline โดยไม่อาศัยการเรียงชื่อไฟล์เป็น compile order

**ข้อจำกัด:** source ที่ import แต่ไม่ได้เป็น dependency ของ top อาจยังไม่ได้รับ semantic analysis จึงต้องรายงานว่า “configuration ที่เลือกผ่าน” ไม่ใช่ “ทุก optional configuration ของ upstream ผ่าน”

---

### 2.6 ขั้นตอนที่ 4 — Import source เข้า library

เพื่อให้คำสั่งเรียก executable ของ GCC/LLVM backend มีตำแหน่งแน่นอน ให้ทำงานภายใน build directory:

```bash
cd "$LAB2_BUILD"

mkdir -p lib/neorv32

set -o pipefail

ghdl -i \
  --std=08 \
  --work=neorv32 \
  --workdir="$LAB2_BUILD/lib/neorv32" \
  "$NEORV32_ROOT"/rtl/core/*.vhd \
  "$NEORV32_ROOT/rtl/test_setups/neorv32_test_setup_bootloader.vhd" \
  2>&1 | tee "$LAB2_REPORT/import.log"
```

การใช้ `rtl/core/*.vhd` ในขั้นตอนนี้เป็น **source indexing ของ baseline** ไม่ใช่ synthesis source list ของ LibreLane

หาก Lab 1 ระบุว่า dependency อยู่ใน directory เพิ่มเติม ต้อง import ไฟล์เหล่านั้นด้วยตาม compile plan ห้ามเพิ่ม source จาก testbench หรือ FPGA integration โดยไม่มีเหตุผล

ความหมายของ options:

| Option | ความหมาย |
|---|---|
| `--std=08` | ใช้ VHDL-2008 |
| `--work=neorv32` | กำหนด logical library |
| `--workdir=...` | กำหนดตำแหน่ง library database |

ใช้ standard และ library settings เดียวกันตลอด import, analysis, elaboration และ simulation

---

### 2.7 ขั้นตอนที่ 5 — Analysis และ elaboration ของ baseline

```bash
ghdl -m \
  --std=08 \
  --work=neorv32 \
  --workdir="$LAB2_BUILD/lib/neorv32" \
  neorv32_test_setup_bootloader \
  2>&1 | tee "$LAB2_REPORT/baseline-build.log"
```

หากคำสั่งผ่าน แสดงว่า hierarchy ของ baseline สามารถสร้างได้ด้วย generic defaults ของ setup นั้น

ตรวจ log:

```bash
rg -n -i \
  'error|warning|unbound|not bound|cannot|failed' \
  "$LAB2_REPORT/import.log" \
  "$LAB2_REPORT/baseline-build.log"
```

ผลค้นหาใช้ช่วย review ไม่ใช่ตัดสิน PASS/FAIL ด้วยคำศัพท์อย่างเดียว ให้ใช้ exit code ของคำสั่งและความหมายของ diagnostics เป็นหลัก

เมื่อใช้ pipeline ผ่าน `tee` ต้องเปิด `set -o pipefail` มิฉะนั้น exit status ของ `tee` อาจทำให้ build ที่ล้มเหลวดูเหมือนสำเร็จ

**ผลที่คาดหวัง:**

- ไม่มี analysis/elaboration error
- ไม่มี component binding ที่ไม่สมบูรณ์ใน design ที่ใช้งาน
- ไม่มี package หรือ memory image หาย
- ไม่มีข้อผิดพลาดเรื่อง generic/port mismatch

---

### 2.8 ขั้นตอนที่ 6 — สร้าง ROM smoke testbench

สร้างไฟล์ `tb/tb_rom_smoke.vhd` ด้วยเนื้อหาต่อไปนี้

Testbench นี้:

- สร้าง clock
- Assert reset ช่วงเริ่มต้น
- ปล่อย reset ที่ falling edge
- ให้ UART RX อยู่ใน idle state
- Decode UART TX แบบ 8-N-1
- เขียน decoded bytes ลงไฟล์
- ตรวจ expected marker
- Fail หาก framing ผิดหรือ timeout

```vhdl
library ieee;
use ieee.std_logic_1164.all;

library std;
use std.textio.all;
use std.env.all;

library neorv32;

entity tb_rom_smoke is
  generic (
    CLK_HZ          : positive := 50_000_000;
    UART_BAUD       : positive := 19_200;
    EXPECTED_MARKER : string   := "NEORV32"
  );
end entity;

architecture sim of tb_rom_smoke is

  constant CLK_PERIOD : time := 1 sec / CLK_HZ;
  constant BIT_PERIOD : time := 1 sec / UART_BAUD;
  constant TIMEOUT    : time := 100 ms;

  signal clk   : std_ulogic := '0';
  signal rstn  : std_ulogic := '0';
  signal rx    : std_ulogic := '1';
  signal tx    : std_ulogic;
  signal gpio  : std_ulogic_vector(7 downto 0);

  signal boot_ok : boolean := false;

  -- Raw decoded UART bytes. Open this file as text after simulation.
  file uart_capture : text open write_mode is "uart_capture.txt";

begin

  clk <= not clk after CLK_PERIOD / 2;

  dut : entity neorv32.neorv32_test_setup_bootloader
    generic map (
      CLOCK_FREQUENCY => CLK_HZ,
      IMEM_SIZE       => 16 * 1024,
      DMEM_SIZE       => 8 * 1024
    )
    port map (
      clk_i       => clk,
      rstn_i      => rstn,
      gpio_o      => gpio,
      uart0_txd_o => tx,
      uart0_rxd_i => rx
    );

  reset_driver : process
  begin
    rstn <= '0';

    for i in 1 to 20 loop
      wait until rising_edge(clk);
    end loop;

    -- Avoid changing reset at the same edge used by the DUT.
    wait until falling_edge(clk);
    rstn <= '1';

    report "Reset released" severity note;
    wait;
  end process;

  uart_receiver : process
    variable value  : natural range 0 to 255;
    variable ch     : character;

    -- Sliding window for substring matching.
    variable window : string(1 to EXPECTED_MARKER'length);
    variable count  : natural := 0;
  begin
    assert EXPECTED_MARKER'length > 0
      report "EXPECTED_MARKER must not be empty"
      severity failure;

    window := (others => character'val(0));

    wait until rstn = '1';

    loop
      -- Standard UART falling edge: idle high -> start low.
      wait until falling_edge(tx);

      -- Sample the center of the start bit.
      wait for BIT_PERIOD / 2;

      assert tx = '0'
        report "UART false start or baud mismatch"
        severity failure;

      value := 0;

      -- Sample D0 through D7, least significant bit first.
      for bit_index in 0 to 7 loop
        wait for BIT_PERIOD;

        assert tx = '0' or tx = '1'
          report "Unknown UART data bit"
          severity failure;

        if tx = '1' then
          value := value + 2**bit_index;
        end if;
      end loop;

      -- Sample the center of the stop bit.
      wait for BIT_PERIOD;

      assert tx = '1'
        report "UART stop-bit error"
        severity failure;

      ch := character'val(value);
      count := count + 1;

      -- Append a raw character to the capture file.
      write(uart_capture, ch);

      report "UART byte " &
             integer'image(count) &
             " = " &
             integer'image(value)
        severity note;

      if window'length > 1 then
        for i in 1 to window'length - 1 loop
          window(i) := window(i + 1);
        end loop;
      end if;

      window(window'high) := ch;

      if window = EXPECTED_MARKER then
        boot_ok <= true;

        report "ROM_SMOKE_PASS: expected UART marker received"
          severity note;

        file_close(uart_capture);
        finish;
      end if;
    end loop;
  end process;

  watchdog : process
  begin
    wait for TIMEOUT;

    assert boot_ok
      report "ROM_SMOKE_FAIL: startup marker timeout"
      severity failure;

    wait;
  end process;

end architecture;
```

**ก่อนใช้งาน:** ตรวจชื่อ entity, generics และ ports ให้ตรงกับ source revision ของโครงการ หากต่างจากตัวอย่าง ให้แก้ DUT instance ตาม interface matrix จาก Lab 1

ค่าขนาด memories ในตัวอย่างใช้สำหรับ baseline ต้องตรวจความเหมาะสมกับ configuration ที่จะนำไป implement อีกครั้ง

#### เหตุผลที่เลือก UART decoder แบบ sampling

สำหรับ UART 8-N-1 หนึ่ง character ใช้:

$$
N_{\mathrm{bits}}=1+8+1=10
$$

ที่ 19,200 baud:

$$
T_{\mathrm{bit}}=\frac{1}{19200}
\approx 52.083\;\mu s
$$

$$
T_{\mathrm{character}}\approx 520.833\;\mu s
$$

ข้อความ 40 characters จึงใช้เวลาส่งประมาณ:

$$
40\times520.833\;\mu s\approx20.833\;ms
$$

การตั้ง simulation เพียง 100 µs อาจยังไม่เห็นแม้แต่ character แรกครบหนึ่งตัว

UART divider ของ DUT อาจสร้าง baud ที่คลาดจากค่าทฤษฎีเล็กน้อย ต้องตรวจว่าความคลาดเคลื่อนยังอยู่ในช่วงที่ decoder sample ถูกต้อง

---

### 2.9 ขั้นตอนที่ 7 — Analyze และ elaborate testbench

```bash
cd "$LAB2_BUILD"

set -o pipefail

ghdl -a \
  --std=08 \
  --work=neorv32 \
  --workdir="$LAB2_BUILD/lib/neorv32" \
  "$PROJECT_ROOT/tb/tb_rom_smoke.vhd" \
  2>&1 | tee "$LAB2_REPORT/tb-analysis.log"

ghdl -m \
  --std=08 \
  --work=neorv32 \
  --workdir="$LAB2_BUILD/lib/neorv32" \
  tb_rom_smoke \
  2>&1 | tee "$LAB2_REPORT/tb-build.log"
```

ตัวอย่างนี้เก็บ testbench ไว้ใน library `neorv32` เช่นเดียวกับ DUT เพื่อให้คำสั่งสั้นและตรวจสอบง่าย การแยก testbench เป็น library อีกชื่อสามารถทำได้ แต่ต้องกำหนด library search path เพิ่มอย่างถูกต้อง

**ผลที่คาดหวัง:** สร้าง `tb_rom_smoke` ได้ โดยไม่มี generic/port binding error

---

### 2.10 ขั้นตอนที่ 8 — รัน ROM smoke

```bash
ghdl -r \
  --std=08 \
  --work=neorv32 \
  --workdir="$LAB2_BUILD/lib/neorv32" \
  tb_rom_smoke \
  -gCLK_HZ=50000000 \
  -gUART_BAUD=19200 \
  -gEXPECTED_MARKER=NEORV32 \
  --assert-level=error \
  2>&1 | tee "$LAB2_REPORT/rom-smoke.log"
```

GHDL รองรับการกำหนด generics ตอน run และ runtime options สำหรับ assertions/waveforms ตามเอกสาร simulator [7.0.0-dev](https://ghdl.github.io/ghdl/using/Simulation.html?utm_source=chatgpt.com)

ตรวจผล:

```bash
rg -n \
  'Reset released|ROM_SMOKE_PASS|ROM_SMOKE_FAIL|assertion' \
  "$LAB2_REPORT/rom-smoke.log"

cat "$LAB2_BUILD/uart_capture.txt"
```

ผลที่ต้องพบ:

```text
Reset released
...
ROM_SMOKE_PASS: expected UART marker received
```

ตัวอย่างนี้หยุด simulation เมื่อพบ marker จึงอาจเก็บเพียงส่วนต้นของ banner

**เกณฑ์ผ่านพร้อมกันทั้งหมด:**

1. Simulator exit code เป็นศูนย์
2. พบ `ROM_SMOKE_PASS`
3. ไม่มี assertion error/failure ก่อน PASS
4. Decoded output มี marker ที่กำหนดจาก bootloader จริง
5. Source/configuration/ROM image ตรงกับ baseline ที่บันทึกไว้

หากใช้ `--stop-time` เพิ่มเองและ simulation จบก่อน watchdog ต้องถือว่า **ยังไม่ทราบผล** แม้ exit code เป็นศูนย์ การถึง stop time ไม่ใช่ PASS

---

### 2.11 ขั้นตอนที่ 9 — เก็บ waveform สำหรับ startup

เริ่มจาก waveform ช่วงสั้นเพื่อดู reset และ initial fetch:

```bash
ghdl -r \
  --std=08 \
  --work=neorv32 \
  --workdir="$LAB2_BUILD/lib/neorv32" \
  tb_rom_smoke \
  -gCLK_HZ=50000000 \
  -gUART_BAUD=19200 \
  --assert-level=error \
  --stop-time=200us \
  --wave="$LAB2_REPORT/startup.ghw" \
  2>&1 | tee "$LAB2_REPORT/startup-wave.log"
```

เปิดด้วย:

```bash
gtkwave "$LAB2_REPORT/startup.ghw"
```

การรันช่วง 200 µs นี้ใช้ตรวจ startup เท่านั้น ไม่ถือว่า ROM smoke ผ่าน

เลือกสัญญาณตาม hierarchy ที่พบจริง:

| กลุ่ม | สิ่งที่ต้องดู |
|---|---|
| Testbench | `clk`, `rstn`, `rx`, `tx` |
| Reset sequencer | Internal reset assertion/deassertion |
| CPU | Fetch PC/address และ execution/control activity |
| Instruction path | Request, response valid/acknowledge และ data |
| Boot ROM | Selection, address และ returned instruction |
| UART | Configuration/write activity และ TX state |

ชื่อ internal signals อาจเปลี่ยนตาม revision ให้เลือกจาก hierarchy จริง ไม่ใช้ชื่อจากคู่มือเป็นข้อกำหนดตายตัว

#### ลำดับที่ควรเห็น

1. External reset เป็น low
2. Clock วิ่งต่อเนื่อง
3. External reset เปลี่ยนเป็น high
4. Internal reset ถูก release ตาม reset logic
5. CPU เริ่ม instruction request
6. Request ไปยัง boot memory ที่กำหนด
7. ROM ส่ง instruction กลับ
8. CPU มี execution activity
9. Firmware เริ่มตั้งค่าและเขียน UART

PC ไม่จำเป็นต้องเพิ่มทีละ 4 ตลอดเวลา เพราะมี branch, jump และอาจมี compressed instructions

หากต้องตรวจ UART waveform เต็ม character ให้เพิ่มระยะเวลาจำลองเท่าที่จำเป็น การเก็บทุก internal signal ตลอด 100 ms อาจทำให้ไฟล์ waveform ใหญ่มาก

---

### 2.12 ขั้นตอนที่ 10 — ทดสอบว่า checker ตรวจความผิดพลาดได้จริง

ทำ negative test โดยกำหนด marker ที่ไม่มีอยู่ใน output:

```bash
ghdl -r \
  --std=08 \
  --work=neorv32 \
  --workdir="$LAB2_BUILD/lib/neorv32" \
  tb_rom_smoke \
  -gCLK_HZ=50000000 \
  -gUART_BAUD=19200 \
  -gEXPECTED_MARKER=THIS_MARKER_MUST_NOT_EXIST \
  --assert-level=error \
  2>&1 | tee "$LAB2_REPORT/negative-marker.log"
```

ผลที่ต้องได้:

- ไม่มี `ROM_SMOKE_PASS`
- พบ `ROM_SMOKE_FAIL: startup marker timeout`
- Exit code ไม่เป็นศูนย์

การทดสอบนี้ยืนยันว่า testbench ไม่ประกาศผ่านเพียงเพราะ simulator จบหรือ UART มี activity

จากนั้นกลับมารัน marker ที่ถูกต้องเพื่อเก็บผล PASS ของ configuration หลักอีกครั้ง

---

### 2.13 ขั้นตอนที่ 11 — นำ smoke test ไปตรวจ ASIC wrapper

เมื่อ baseline ผ่าน ให้เปลี่ยนเฉพาะ DUT instance ใน testbenchให้เป็น ASIC wrapper ของโครงการ

ตรวจ mapping:

| Testbench connection | ASIC wrapper connection |
|---|---|
| `clk` | Main clock input |
| `rstn` | Reset input พร้อมตรวจ polarity |
| `rx` | UART0 RX |
| `tx` | UART0 TX |
| `gpio` | GPIO output bits ที่เลือก |

ทำตามลำดับ:

1. Import ASIC wrapper และ dependencies เพิ่ม
2. ตรวจ generic map เทียบกับ `soc-configuration.csv`
3. Analyze testbench ที่ instantiate ASIC wrapper
4. Elaborate
5. Run UART smoke เดิม
6. เปรียบเทียบ configuration และผลกับ baseline

ไม่ต้อง instantiate padframe ในขั้นตอนนี้ หากต้องการตรวจ VHDL SoC wrapper โดยตรง การทดสอบผ่าน signal pads เป็นอีกระดับหนึ่งของ integration verification

รายงานผลแยกกัน:

| Test | Configuration | Status |
|---|---|---|
| Upstream baseline | Upstream bootloader test setup | PASS/FAIL |
| ASIC wrapper smoke | Configuration สำหรับโครงการ | PASS/FAIL/NOT RUN |
| Full-chip pad simulation | รวม padframe และ pad models | NOT RUN ใน Lab นี้ เว้นแต่ทำเพิ่ม |

**เกณฑ์ผ่านของโครงการ:** หาก ASIC wrapper พร้อมใช้งานแล้ว ต้องผ่าน smoke test ของ wrapper ด้วย การผ่าน upstream baseline อย่างเดียวไม่ยืนยัน generic map และ tie-offs ของโครงการ

---

### 2.14 วิธีวิเคราะห์ปัญหาที่พบบ่อย

| อาการ | สิ่งที่ควรตรวจ | วิธีแก้ |
|---|---|---|
| หา `neorv32` library ไม่พบ | `--work`, `--workdir`, library paths | ใช้ settings เดียวกันทุก stage |
| หา package ไม่พบ | Source missing หรือ dependency | ตรวจ compile plan และ image packages |
| Generic ไม่มีใน entity | Wrapper กับ revision ไม่ตรง | แก้จาก declaration ของ source จริง |
| Component ไม่ bound | Entity/architecture/library | ตรวจ hierarchy และ bindings |
| Simulation ไม่มี UART activity | Reset, boot mode, UART enable, ROM path | เริ่มดู internal reset และ instruction fetch |
| มี TX edges แต่ framing error | Baud, clock metadata, sampling | ตรวจ `CLK_HZ`, DUT frequency และ ROM baud |
| UART decode ได้แต่ marker ไม่พบ | Marker, image revision, timeout | ตรวจ output และ startup string จริง |
| Timeout แต่ PC มี activity | Bootloader รอเหตุการณ์ หรือ configuration ไม่ตรง | ดู boot source และ peripheral accesses |
| Fetch response ไม่กลับ | Memory/bus integration | Trace request/response และ decode |
| PC เข้าสู่ trap ซ้ำ | ISA, ROM contents, access faults | ตรวจ trap cause และ boot image compatibility |
| Unknown warnings ที่เวลา 0 | Uninitialized signals | แยก startup transient จาก unknown ที่ใช้งานจริง |
| Negative test กลับผ่าน | Checker หรือ marker definition | ตรวจ PASS condition และ exit handling |

ไม่ควรเพิ่ม `-frelaxed-rules` หรือปิด assertions เพื่อทำให้ error หายโดยยังไม่ทราบสาเหตุ หากจำเป็นต้องใช้ compatibility option ต้องบันทึกเหตุผลและ diagnostics ที่เกี่ยวข้อง

---

### 2.15 ผลส่งของ Lab

ผู้เรียนต้องส่ง:

1. **Environment evidence:** GHDL version และ source revision
2. **Build evidence:** import, analysis และ elaboration logs
3. **Testbench:** `tb_rom_smoke.vhd`
4. **Boot expectation:** UART configuration, marker และ ROM image hash
5. **Positive test:** PASS log และ decoded UART capture
6. **Negative test:** timeout failure log
7. **Waveform evidence:** Reset release และ initial instruction fetch
8. **Wrapper result:** ผลของ ASIC wrapper หรือเหตุผลที่ยังไม่รัน

ตัวอย่างตารางสรุป:

| รายการ | ผล | หลักฐาน |
|---|---|---|
| Baseline analysis/elaboration | PASS/FAIL | `baseline-build.log` |
| Testbench analysis/elaboration | PASS/FAIL | `tb-analysis.log`, `tb-build.log` |
| UART framing | PASS/FAIL | `rom-smoke.log` |
| Expected marker | PASS/FAIL | UART capture |
| Timeout negative test | PASS/FAIL | `negative-marker.log` |
| Reset/fetch inspection | CHECKED/NOT CHECKED | Waveform และบันทึก |
| ASIC wrapper smoke | PASS/FAIL/NOT RUN | Wrapper smoke log |

### 2.16 เกณฑ์ผ่าน

Lab ผ่านเมื่อ:

- Configuration ที่เลือก analyze/elaborate ได้
- Boot ROM และ configuration มี provenance ที่ตรวจสอบได้
- CPU เริ่ม boot path หลัง reset
- UART decoder ได้ข้อความที่คาดหวังโดยไม่มี framing error
- Positive test มี PASS marker และ exit code ถูกต้อง
- Negative test ล้มเหลวตามที่ออกแบบไว้
- ไม่มี diagnostics สำคัญที่ยังไม่ได้อธิบาย
- ASIC wrapper ผ่านด้วย หากใช้ wrapper นั้นเป็น input ของ Lab ถัดไป

ROM smoke ยืนยันเส้นทาง startup ที่ทดสอบ ยังไม่ครอบคลุม instruction set ทั้งหมด, firmware upload, SRAM macro timing, ASIC power-up behavior หรือ physical sign-off

### 2.17 คำถามท้าย Lab

1. เพราะเหตุใด `ghdl -i` ผ่านจึงยังไม่เท่ากับ VHDL analysis ผ่าน?
2. Analysis ผ่านแล้ว แต่ elaboration ล้มเหลวได้จากสาเหตุใด?
3. ทำไมต้องใช้ `--std=08` และ library settings ให้ตรงกันทุก stage?
4. ถ้า UART มี activity แต่ decode ไม่ได้ ควรตรวจอะไรเป็นอันดับแรก?
5. เพราะเหตุใดการพบ startup marker จึงเป็นหลักฐานที่ดีกว่าการพบ TX edges?
6. Clock จริงกับ `CLOCK_FREQUENCY` ไม่ตรงกันจะกระทบ UART อย่างไร?
7. การถึง `--stop-time` โดยไม่มี assertion failure ถือว่าผ่านหรือไม่?
8. Negative test ช่วยตรวจความน่าเชื่อถือของ testbench อย่างไร?
9. Baseline ผ่านแต่ ASIC wrapper ไม่ผ่าน ควรเปรียบเทียบ configuration และ connections จุดใด?
10. ROM smoke ผ่านแล้ว เหตุใดจึงยังไม่ยืนยันว่า SRAM จะมีข้อมูลถูกต้องหลังเปิดชิปจริง?