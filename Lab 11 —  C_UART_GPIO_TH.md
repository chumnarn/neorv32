# Lab 11 — โปรแกรม C UART/GPIO

คู่มือ NEORV32 → LibreLane → IHP SG13G2 • บทขยายเชิงปฏิบัติ • 6 ตุลาคม 2026

## 11.1 เป้าหมายและขอบเขต

Lab นี้เชื่อมงาน software เข้ากับฮาร์ดแวร์ที่พัฒนาจาก Lab 1–10 ผู้เรียนจะตรวจ configuration, build โปรแกรม C ด้วย NEORV32 software framework, ตรวจ ELF และงบหน่วยความจำ, สร้างและติดตั้ง VHDL IMEM image, export Verilog ใหม่ และทดสอบ UART TX/RX กับ GPIO ด้วย testbench ที่มี timeout และตรวจข้อมูลจริง

เมื่อจบ Lab ผู้เรียนต้องอธิบายได้ว่า source C, executable, ROM image และ netlist มีความสัมพันธ์กันอย่างไร และทำไม firmware ใหม่จึงใช้กับ routed netlist เดิมไม่ได้ใน profile ROM นี้

การทดสอบแบ่งเป็น RTL simulation, simulation ของ exported Verilog, chip wiring/pad model simulation และ silicon bring-up แต่ละระดับต้องมีหลักฐานของตัวเอง ไม่ใช้ PASS จากระดับหนึ่งแทนอีกระดับ

สถานะบทนี้: ตรวจคำสั่ง, API และ configuration จาก source ใน NEORV32_IHP_SG13G2_Project.zip ที่มีอยู่จริงแล้ว ตัวอย่าง C และ testbench ต่อไปนี้เป็นเนื้อหาสำหรับผู้เรียนรัน ยังไม่ได้ cross-compile หรือ simulate ในสภาพแวดล้อมจัดทำบทนี้ เพราะไม่มี RISC-V GCC/GHDL ที่ตรวจพบ ไม่อ้างว่าขนาด executable หรือผล UART/GPIO ผ่านแล้ว

## 11.2 Baseline ที่ต้องรักษา

| รายการ | ค่าในโปรเจกต์ |
|---|---|
| NEORV32 snapshot | af905649fdbe650245af700690e435b58601ef7e |
| CPU ISA สำหรับ build | rv32i_zicsr_zifencei |
| ABI | ilp32 |
| System clock | 50,000,000 Hz; period 20 ns |
| BOOT_MODE_SELECT | 2: pre-initialized IMEM ROM |
| IMEM | base 0x00000000, 4 KiB |
| DMEM | base 0x80000000, 1 KiB |
| GPIO | 8 input และ 8 output |
| UART | UART0, 115200 baud สำหรับ application นี้ |
| UART0 base | 0xFFF50000 |
| GPIO base | 0xFFFC0000 |
| SYSINFO base | 0xFFFE0000 |
| UART RX/TX mapping | input_in[8] / output_out[8] |
| GPIO input/output mapping | input_in[7:0] / output_out[7:0] |

Address ของ peripheral ตรวจจาก vendor/neorv32/sw/lib/include/neorv32.h ของ snapshot นี้ ไม่คัดลอก address จากคู่มือ NEORV32 รุ่นเก่า GPIO input/output เป็นคนละเส้นทางใน wrapper ไม่ใช่ GPIO bidirectional pad ที่ต้องตั้ง direction register แบบ microcontroller ทุกชนิด

Boot mode 2 หมายถึง C image ถูกแปลงเป็นวงจร ROM ก่อนผลิต การส่งไฟล์ผ่าน UART หลังผลิตไม่ได้เปลี่ยน ROM หากต้องการโหลดโปรแกรมใหม่ ต้องพัฒนา writable IMEM/bootloader profile แยกพร้อมตรวจ memory และ flow ใหม่

## 11.3 ขั้นที่ 1 — เก็บ baseline และตรวจ source

รันคำสั่งจาก root ของโปรเจกต์ neorv32-ihp-sg13g2:

```bash
mkdir -p build/lab11
cp sw/main.c build/lab11/main-before.c
cp vendor/neorv32/rtl/core/neorv32_imem_image.vhd \
  build/lab11/imem-before.vhd
cp sw/Makefile build/lab11/sw-Makefile-before
sha256sum rtl/neorv32_asic.vhd rtl/chip_core.sv \
  sw/main.c sw/Makefile \
  vendor/neorv32/rtl/core/neorv32_imem_image.vhd \
  > build/lab11/baseline-sha256.txt
rg -n 'CLOCK_FREQUENCY|BOOT_MODE_SELECT|IMEM|DMEM|GPIO|UART' \
  rtl/neorv32_asic.vhd
cat rtl/chip_core.sv
cat sw/Makefile
rg -n 'UART0_BASE|GPIO_BASE|SYSINFO_BASE' \
  vendor/neorv32/sw/lib/include/neorv32.h
```

บันทึก mapping ต่อจาก Lab 6 ให้ถึง package pin จริง: software bit → SoC port → chip_core bus bit → pad instance → bondpad/package pin สำหรับการวัดบนชิป GPIO bit 0 ไม่จำเป็นต้องตรง package pin 0

ตรวจว่า UART RX ที่ไม่ใช้งานถูกขับเป็น logic 1 ใน testbench และไม่มี external input ลอย สำหรับ reset ใช้ rstn_i active low และปล่อย reset ตามข้อกำหนด clock/reset ของ Lab 7

ผลส่งขั้นนี้: configuration table, pin mapping, baseline hash และสำเนา ROM เดิม

## 11.4 ขั้นที่ 2 — ตรวจ cross compiler และ multilib

```bash
export RISCV_PREFIX=riscv64-unknown-elf-
"${RISCV_PREFIX}gcc" --version
"${RISCV_PREFIX}gcc" -print-multi-lib
"${RISCV_PREFIX}gcc" -march=rv32i_zicsr_zifencei -mabi=ilp32 \
  -print-libgcc-file-name
make -C sw info RISCV_PREFIX="$RISCV_PREFIX" \
  > build/lab11/build-info.txt
```

ชื่อ compiler ขึ้นต้น riscv64 ไม่ได้บอกว่าจะสร้าง RV64 เสมอ ต้องดู -march/-mabi และ ELF ที่สร้างจริง compiler ต้องมี libgcc ที่ใช้กับ RV32 ILP32 โดยเฉพาะ baseline ไม่มี hardware M extension การหารใน UART setup จึงอาจต้องใช้ software helper จาก libgcc

ถ้ามี toolchain riscv32 ให้เปลี่ยน prefix เป็น riscv32-unknown-elf- แล้วใช้ prefix เดียวกันกับ gcc/readelf/size/objdump ทุกขั้น ไม่ใช้ Linux userspace toolchain แทน bare-metal framework โดยไม่ได้ตรวจ runtime/linker

ห้ามเปิด M/C/F extension เพียงเพื่อแก้ปัญหา link เพราะ CPU RTL baseline ไม่ได้เปิด feature เหล่านั้น

## 11.5 ขั้นที่ 3 — ตรวจ Makefile และ linker contract

sw/Makefile เดิมมีสาระดังนี้:

```make
NEORV32_HOME := $(abspath ../vendor/neorv32)
RISCV_PREFIX ?= riscv64-unknown-elf-
MARCH = rv32i_zicsr_zifencei
MABI = ilp32
EFFORT = -Os
USER_FLAGS += -Wl,--defsym,__neorv32_rom_size=4k -Wl,--defsym,__neorv32_ram_size=1k
include $(NEORV32_HOME)/sw/common/common.mk
```

เพิ่มสองบรรทัดต่อไปนี้ก่อน include เพื่อเก็บ map และรายงาน stack ของแต่ละ function:

```make
USER_FLAGS += -Wl,-Map=main.map -fstack-usage
```

อย่ากำหนด USER_FLAGS ใหม่จาก command line จนทับ defsym 4k/1k ที่อยู่ในไฟล์ เพราะ linker default ของ framework ใหญ่กว่า hardware baseline แล้ว link อาจผ่านทั้งที่ชิปมี memory ไม่พอ

อ่าน vendor/neorv32/sw/common/neorv32.ld: ROM/RAM base, section placement และ __crt0_stack_top ซึ่ง baseline อยู่ที่ปลาย DMEM 0x80000400 startup framework เป็นผู้ตั้ง stack, เตรียม .data/.bss และเข้าสู่ main อย่าเอา C main ไปวางแทน smoke assembly โดยละเลย startup

## 11.6 ขั้นที่ 4 — โปรแกรม C สำหรับ directed UART/GPIO test

แทน sw/main.c ด้วยตัวอย่างนี้ ใช้ polling และ API ของ snapshot ไม่ต้องใช้ heap, printf ของ libc หรือ interrupt application

```c
#include <stdint.h>
#include <neorv32.h>

static uint32_t out_value;

static char hex_digit(uint32_t value) {
  value &= 15u;
  return (char)((value < 10u) ? ('0' + value) : ('A' + value - 10u));
}

static void report_input(void) {
  uint32_t value = neorv32_gpio_port_get() & 255u;
  neorv32_uart0_putc('I');
  neorv32_uart0_putc('=');
  neorv32_uart0_putc(hex_digit(value >> 4));
  neorv32_uart0_putc(hex_digit(value));
  neorv32_uart0_putc('\n');
}

int main(void) {
  /* Keep the framework trap handler for diagnosis; application uses polling. */
  neorv32_rte_setup();

  if (!neorv32_uart0_available()) {
    if (neorv32_gpio_available()) neorv32_gpio_port_set(0xeeu);
    for (;;) {}
  }
  neorv32_uart0_setup(115200, 0);
  if (!neorv32_gpio_available()) {
    neorv32_uart0_puts("ERR:GPIO\n");
    for (;;) {}
  }

  out_value = 0;
  neorv32_gpio_port_set(out_value);
  neorv32_uart0_puts("LAB11 READY\n");

  for (;;) {
    if (!neorv32_uart0_char_received()) continue;
    char command = neorv32_uart0_char_received_get();
    switch (command) {
      case 'a':
        out_value = 0xa5u;
        neorv32_gpio_port_set(out_value);
        neorv32_uart0_puts("O=A5\n");
        break;
      case 'b':
        out_value = 0x5au;
        neorv32_gpio_port_set(out_value);
        neorv32_uart0_puts("O=5A\n");
        break;
      case '0':
        out_value = 0;
        neorv32_gpio_port_set(out_value);
        neorv32_uart0_puts("O=00\n");
        break;
      case 't':
        out_value ^= 1u;
        neorv32_gpio_port_set(out_value);
        neorv32_uart0_puts("T\n");
        break;
      case 'i':
        report_input();
        break;
      default:
        neorv32_uart0_putc(command);
        break;
    }
  }
}
```

ตาราง protocol:

| Host ส่ง | ผล GPIO | UART ตอบ |
|---|---|---|
| reset | 00 | LAB11 READY ตามด้วย newline |
| a | A5 | O=A5 |
| b | 5A | O=5A |
| 0 | 00 | O=00 |
| t หลัง 00 | 01 | T |
| i เมื่อ input=3C | output เดิม | I=3C |
| ? | output เดิม | ? |

คำสั่งเป็น single byte ไม่ต้องส่ง Enter ในตัวอย่างนี้ CR/LF ที่ host เพิ่มมาจะเข้า default echo path ทำให้ log มีข้อมูลเพิ่มขึ้น ต้องกำหนด terminal/testbench ให้สอดคล้องกัน neorv32_uart0_puts และ putc ของ framework อาจแปลง newline เป็น CRLF ดังนั้น checker ด้านล่างยอมรับ CR แต่ตรวจ byte อื่นอย่างเคร่งครัด

การไม่ใช้ blocking getc ทำให้ loop มีที่สำหรับเพิ่มงานอื่น อย่างไรก็ตาม puts/putc อาจรอ TX FIFO จึงยังไม่ใช่ nonblocking application ทั้งหมด ให้ host รอ response ก่อนส่งคำสั่งถัดไปเพื่อไม่ให้ FIFO overflow

ตัวแปร out_value เป็น software shadow ของ output ไม่อ่าน input register เพื่อเดาค่า output เพราะ gpio_i และ gpio_o เป็นคนละเส้นทาง สำหรับ volatile ใช้กับ MMIO ผ่าน header/API ที่ framework จัดไว้ ไม่ใช้ volatile กับทุกตัวแปรและไม่ถือว่า volatile แก้ race/CDC

## 11.7 ขั้นที่ 5 — Build โดยยังไม่ติดตั้ง image

```bash
set -o pipefail
make -C sw clean_all
make -C sw elf asm image RISCV_PREFIX="$RISCV_PREFIX" \
  2>&1 | tee build/lab11/c-build.log
"${RISCV_PREFIX}readelf" -h sw/main.elf \
  > build/lab11/elf-header.txt
"${RISCV_PREFIX}readelf" -A sw/main.elf \
  > build/lab11/elf-attributes.txt
"${RISCV_PREFIX}readelf" -lW sw/main.elf \
  > build/lab11/elf-segments.txt
"${RISCV_PREFIX}size" -A sw/main.elf \
  > build/lab11/elf-size.txt
"${RISCV_PREFIX}nm" -n sw/main.elf \
  > build/lab11/elf-symbols.txt
cp sw/main.map build/lab11/main.map
```

ใช้ elf asm image เพื่อแยก review ออกจาก install ต่างจาก make firmware-c ของ root ซึ่งเรียก clean_all แล้ว all และ all ของ snapshot นี้รวม install อยู่ด้วย

ตรวจ ELFCLASS32, machine RISC-V, entry point, ISA attributes และ disassembly ต้องตรง RV32I/Zicsr/Zifencei การพบ software __divsi3/__udivsi3 ไม่เท่ากับมี hardware DIV instruction ให้ดู assembly จริง

### งบ ROM

ใช้ map/LOAD segments และขนาด sw/elf.bin ตรวจ code, read-only data, startup และ initializers ของ .data ที่เก็บใน ROM ช่วง load image ต้องอยู่ภายใน [0x00000000,0x00001000) อย่าใช้ขนาด main.elf ทั้งไฟล์เพราะรวม metadata และ symbol/debug information

### งบ RAM และ stack

.data + .bss + alignment + heap ที่ใช้งาน + worst-case stack ต้องไม่เกิน 1024 bytes โปรแกรมนี้ไม่ใช้ heap แต่ runtime/trap และ call chain ยังใช้ stack linker ผ่านไม่ได้รับรองว่า runtime stack ไม่ชน global data

อ่าน .su ที่ -fstack-usage สร้างไว้ใน build directory ของ sw รวม stack ตาม call chain ไม่ใช่ใช้เฉพาะ function ที่มี frame ใหญ่ที่สุด และเผื่อ trap handler ที่อาจเข้าระหว่างการทำงาน หากใช้ stack watermark ใน simulation ต้องทำหลัง startup และ fill เฉพาะพื้นที่ที่แน่ใจว่าไม่ทับ current stack/data ไม่เขียน pattern ทั่ว DMEM ขณะโปรแกรมกำลังใช้ stack

ถ้า ROM overflow ให้ลด banner/feature และวิเคราะห์ function contribution ถ้า RAM ไม่พอให้ลด buffer/global และ call depth ก่อน หากต้องเพิ่ม IMEM/DMEM ให้เปลี่ยน RTL/linker พร้อมกันและกลับ synthesis/PNR เพราะ memory baseline ถูก map เป็น standard cells การเพิ่ม memory ไม่ใช่การแก้ software อย่างเดียว

ผลส่ง: build log, ELF, assembly, map, size report และตาราง ROM/RAM/stack budget ไม่ใส่ตัวเลขคาดเดาว่าผ่านก่อนรัน compiler จริง

## 11.8 ขั้นที่ 6 — ติดตั้ง C image และตรวจ provenance

```bash
make -C sw install RISCV_PREFIX="$RISCV_PREFIX"
cmp sw/neorv32_imem_image.vhd \
  vendor/neorv32/rtl/core/neorv32_imem_image.vhd
sha256sum sw/main.c sw/Makefile sw/main.elf sw/elf.bin \
  sw/neorv32_imem_image.vhd \
  vendor/neorv32/rtl/core/neorv32_imem_image.vhd \
  > build/lab11/c-image-sha256.txt
make export
sha256sum build/neorv32_asic.v > build/lab11/export-sha256.txt
```

cmp ต้องคืนสถานะสำเร็จและ hashes ของ generated/installed image ต้องตรงกัน จากนั้น export ต้องวิเคราะห์ image ใหม่จริง อย่าคัดลอก exported Verilog จาก run smoke เดิมมาใช้

ห้ามรัน make firmware-smoke หรือ scripts/run_chip.sh ระหว่างการเตรียม C chip โดยไม่อ่าน dependency เพราะ firmware-smoke จะทับ ROM ด้วย smoke program scripts/run_chip.sh ของแพ็กเกจเป็น smoke-first workflow ส่วน make chip ของ root เรียก export แล้วรัน LibreLaneโดยตรง

make sim / make sim-verilog / make sim-chip-wiring เดิมมี checker ของ smoke A5→5A ไม่ใช่ checker ของ application นี้ ต้องใช้ testbench ต่อไปนี้แทน ไม่แก้ด้วยการลบ timeout หรือ assert

## 11.9 ขั้นที่ 7 — คำนวณ baud และตรวจ clock จริง

UART setup ของ snapshot อ่าน clock จาก SYSINFO และคำนวณ divider สำหรับ baud ต่ำที่ใช้ prescaler=1:

D = floor(50,000,000 / (2 × 115,200)) = 217

baud_actual = 50,000,000 / (2 × 217) ≈ 115,207.37 baud

bit_time_actual = 2 × 217 × 20 ns = 8680 ns

error ≈ +0.0064% เทียบเป้าหมาย 115200 bit time เป้าหมายคือ 8680.56 ns frame 8N1 มี 10 bits จึงใช้ประมาณ 86.8 µs ต่อ byte

สามค่าต้องสอดคล้องกัน: CLOCK_FREQUENCY=50000000 ใน RTL, clock period=20ns ใน SDC/testbench และ clock ที่ป้อนชิปจริง การเปลี่ยน CLOCK_PERIOD ใน SDC ไม่เปลี่ยนค่า SYSINFO และไม่เปลี่ยนความถี่ oscillator หากลด clock เพื่อ debug ให้แก้ configuration ที่เกี่ยวข้องด้วย

อย่าเปิด UART0_SIM_MODE ใน image ที่ต้องตรวจ serial pin หรือส่งผลิต เพราะ mode นี้ส่งข้อมูลเข้าช่อง simulation แทนการทดสอบ transmitter จริง

## 11.10 ขั้นที่ 8 — UART/GPIO testbench ของ exported Verilog

สร้าง sim/lab11_tb.sv ต่อไปนี้ ตรวจ UART ที่ pin จริงและรองรับ CRLF/ LF โดยตัดเฉพาะ CR ตัว checker ทำงานต่อเนื่องจึงไม่พลาด byte แรกระหว่างรอ GPIO

```systemverilog
`timescale 1ns/1ps
module lab11_tb;
  reg clk = 0;
  reg rst_n = 0;
  reg rx = 1;
  reg [7:0] gi = 8'h3c;
  wire [7:0] go;
  wire tx;
  localparam integer BIT_NS = 8681;
  byte received[$];

  neorv32_asic dut (
    .clk_i(clk), .rstn_i(rst_n),
    .gpio_i(gi), .gpio_o(go),
    .uart_rxd_i(rx), .uart_txd_o(tx)
  );
  always #10 clk = ~clk;

  task automatic send_byte(input byte value);
    @(negedge clk);
    rx = 0;
    #(BIT_NS);
    for (int k = 0; k < 8; k++) begin
      rx = value[k];
      #(BIT_NS);
    end
    rx = 1;
    #(BIT_NS);
  endtask

  task automatic expect_text(input string expected);
    byte value;
    for (int k = 0; k < expected.len(); k++) begin
      wait (received.size() != 0);
      value = received.pop_front();
      if (value !== expected[k])
        $fatal(1, "UART mismatch index=%0d got=%02x expected=%02x",
               k, value, expected[k]);
    end
  endtask

  task automatic expect_gpio(input logic [7:0] expected);
    repeat (10000) begin
      @(negedge clk);
      if (go === expected) return;
    end
    $fatal(1, "GPIO mismatch got=%02x expected=%02x", go, expected);
  endtask

  initial begin : monitor_tx
    byte value;
    wait (rst_n === 1'b1);
    forever begin
      @(negedge tx);
      #(BIT_NS/2);
      if (tx !== 1'b0) $fatal(1, "invalid UART start bit");
      for (int k = 0; k < 8; k++) begin
        #(BIT_NS);
        value[k] = tx;
      end
      #(BIT_NS);
      if (tx !== 1'b1) $fatal(1, "invalid UART stop bit");
      if (value != 8'h0d) received.push_back(value);
    end
  end

  initial begin : stimulus
    $dumpfile("build/lab11/lab11.vcd");
    $dumpvars(0, lab11_tb);
    repeat (20) @(negedge clk);
    rst_n = 1;
    expect_text("LAB11 READY\n");
    expect_gpio(8'h00);

    send_byte("a"); expect_text("O=A5\n"); expect_gpio(8'ha5);
    send_byte("b"); expect_text("O=5A\n"); expect_gpio(8'h5a);
    send_byte("0"); expect_text("O=00\n"); expect_gpio(8'h00);
    send_byte("t"); expect_text("T\n"); expect_gpio(8'h01);
    send_byte("t"); expect_text("T\n"); expect_gpio(8'h00);
    send_byte("i"); expect_text("I=3C\n"); expect_gpio(8'h00);
    @(negedge clk); gi = 8'ha6;
    repeat (20) @(negedge clk);
    send_byte("i"); expect_text("I=A6\n"); expect_gpio(8'h00);
    send_byte("?"); expect_text("?"); expect_gpio(8'h00);
    repeat (1000) @(negedge clk);
    if (received.size() != 0) $fatal(1, "unexpected UART bytes");
    $display("PASS: LAB11 serial TX/RX and GPIO directed tests");
    $finish;
  end

  initial begin
    #20000000;
    $fatal(1, "LAB11 timeout (20 ms)");
  end
endmodule
```

ใช้ simulator ที่รองรับ SystemVerilog queue/string/task return ตัวอย่างคำสั่ง Icarus:

```bash
set -o pipefail
iverilog -g2012 -s lab11_tb -o build/lab11/lab11.vvp \
  build/neorv32_asic.v sim/lab11_tb.sv
vvp build/lab11/lab11.vvp 2>&1 | tee build/lab11/serial-test.log
```

หาก simulator version ไม่รองรับ construct ให้ปรับ checker เป็น fixed buffer ตามข้อจำกัด simulator โดยรักษาการตรวจ byte/timeout ไม่ลบ test เพื่อให้ compile ผ่าน

20 ms เป็นเวลาใน simulation ไม่ใช่ระยะเวลาที่โปรแกรม simulator ต้องใช้บนเครื่อง host หากต้องเพิ่ม timeout ต้องอธิบาย boot/runtime ที่เพิ่มขึ้น ตรวจว่า image ถูกต้องก่อน

ตรวจ waveform ที่ reset, clk, rx, tx, gi, go ต้องเห็น TX idle high, start low, 8 bits LSB-first และ stop high การเห็น tx toggle อย่างเดียวไม่พอ ต้อง decode banner/response และตรวจ GPIO ตามคำสั่ง

ผล input A6 เป็น directed steady-input check ไม่รับรอง CDC/metastability หรือ switch debounce ให้ต่อยอด synchronizer/debounce ตามงานระบบจริงและตรวจ Lab 7

## 11.11 ขั้นที่ 9 — เพิ่ม coverage และ negative tests

1. GPIO walking-one: เพิ่มคำสั่งหรือ test program เขียน 01,02,04,08,10,20,40,80 ตรวจทุก bit ที่ core/wrapper และ pad output เพื่อจับ permutation ที่ pattern A5/5A อาจพลาด
2. GPIO input walking-one: เปลี่ยน gi ทีละ bit และอ่านกลับผ่านคำสั่ง i ต้องได้ค่าตรงกันทั้ง 8 bit
3. Reset ระหว่าง idle: assert reset low, ยกเลิก checker transaction เก่า, clear RX scoreboard, ปล่อย reset แล้วต้องได้ banner ใหม่และ output 00
4. Reset ระหว่าง UART frame: แยก negative testbench และกำหนด partial frame ที่ยอมให้ถูกตัด ไม่ reuse expected stream จากก่อน reset
5. Wrong baud: ส่งด้วย baud ต่างจาก specification ใน separate run ตรวจพฤติกรรม error และ recovery ไม่กำหนดว่าต้องผิดทุก byte เพราะบาง pattern อาจยังอ่านได้
6. RX flood: ส่งหลายคำสั่งติดกัน ตรวจ FIFO overrun/status ตาม source จริง งานนี้อยู่นอก stop-and-wait acceptance ต้องบันทึกจำนวน sent/received/lost bytes
7. Clock sweep: ถ้าเปลี่ยน clock จริงและคง SYSINFO เดิม ให้แสดง baud mismatch จาก waveform แล้วแก้ generic/testbench/SDC ให้ตรงกัน

แยก directed functional acceptance ออกจาก robustness experiments ไม่ใช้ negative-test FAIL ที่ตั้งใจให้เกิดมาปะปนกับ baseline PASS

## 11.12 ขั้นที่ 10 — ตรวจ VHDL และ chip wrapper

สำหรับ VHDL ให้สร้าง sim/lab11_tb.vhd ที่ทำ stimulus/decoder protocol เดียวกัน วิเคราะห์ตาม scripts/analyze.sh และ NEORV32 library ของโปรเจกต์ ไม่ใช้ smoke checker เดิมเป็น acceptance ของ C image เปรียบเทียบ UART byte stream และลำดับ GPIO ไม่บังคับ timestamps ต้องตรงทุก ps ระหว่าง simulator

สำหรับ chip wrapper ใช้ instantiation ของ chip_top ที่มี ports จริงใน rtl/chip_top.sv และ sim/pads_mock.v แล้วเปลี่ยน monitor/stimulus ไปที่ external input/output ของ chip ตรวจ input_in[8], output_out[8] และ GPIO bits ตาม Lab 6

Mock pad ตรวจ logical wiring เท่านั้น ไม่รับรอง delay, voltage, ESD หรือ electrical behavior ถ้าตรวจ mapped/pad simulation ต้องใช้ PDK functional models ที่ตรง snapshot และต่อ power/ground pins ตาม model ไม่เอา mocks เข้า synthesis source list

ถ้า gate-level test ใช้ netlist จาก Lab 10 ต้องเป็น netlistที่สังเคราะห์จาก C image นี้ ถ้า netlist เป็น smoke ROM เก่าให้ rebuild ก่อน การเปลี่ยน testbench ไม่ทำให้ firmware ใน gate netlist เปลี่ยน

## 11.13 ขั้นที่ 11 — Rebuild implementation สำหรับ C ROM

หลัง size และ functional checks ผ่าน:

```bash
make check-verilog
make synth RUN_TAG=NEORV32_C_LAB11_SYNTH
make chip RUN_TAG=NEORV32_C_LAB11_CHIP
```

ใช้ run tag ใหม่และอ่าน config จริงก่อนรัน เปรียบเทียบ cell area, ROM logic, congestion และ timing กับ smoke baseline C image ใหญ่ขึ้นอาจทำให้ timing/placement/routing เปลี่ยน แม้ SoC architecture และ memory window เท่าเดิม

ต้องรัน downstream checks จาก Lab 5–10 ใหม่ตาม methodology: synthesis, placement, CTS, routing, extraction, final STA และ required physical gates การ resume จาก routed state ที่ฝัง smoke ROM จะไม่ตรง firmware ใหม่

หลัง implementation บันทึก hashes ของ C source, installed image, exported Verilog, mapped/final netlist, configuration และ GDS พร้อม run tag ที่สัมพันธ์กัน ไม่อ้าง GDS เก่าเป็นผลของ image ใหม่

## 11.14 ขั้นที่ 12 — Silicon bring-up เมื่อมีชิปและบอร์ดจริง

1. ตรวจ supply rails, voltage domain, power sequence, clock และ reset ตาม chip/package specification จาก Lab 6–7
2. ตรวจ USB-UART adapter ว่า logic voltage เข้ากับ IOVDD/IO specification จริง ไม่สมมติว่า adapter 3.3V ต่อกับทุก pad ได้ ใช้ logic UART ไม่ต่อ RS-232 voltage ตรงกับ IO
3. ต่อ host TX → chip RX, host RX ← chip TX และ GND ร่วม วัด clock/reset และ TX idle ก่อนทดสอบ protocol
4. เปิด terminal 115200, 8 data bits, no parity, 1 stop bit, no flow control แล้ว reset ต้องเห็น LAB11 READY
5. ส่ง a/b/0/t/i แบบ single byte วัด GPIO ที่ pin/package และดู UART response ตามตาราง
6. ให้ GPIO inputs มีระดับ logic ที่กำหนด อ่าน I=xx เปรียบเทียบค่าที่ขับจริง ไม่ปล่อย input ลอยและไม่ขับชน output
7. บันทึก board revision, chip identifier, supply/clock ที่วัด, adapter voltage, terminal settings, serial log และ logic analyzer trace

หากยังไม่มี silicon ให้ระบุ NOT_RUN ใน silicon acceptance ไม่ใช้ simulation PASS เปลี่ยนเป็น hardware PASS

## 11.15 แนวทางวิเคราะห์ปัญหา

| อาการ | หลักฐานที่ตรวจ | การแก้ |
|---|---|---|
| command gcc ไม่พบ | command -v, PATH, prefix | เข้า environment ที่มี bare-metal toolchain |
| link incompatible libgcc | multilib, -march/-mabi, ELF | เลือก RV32 ILP32 library ที่เข้ากัน |
| ROM overflow | main.map, LOAD segments, elf.bin | ลด code หรือปรับ memory profile/RTL พร้อมกัน |
| รันแล้วค้างแม้ link ผ่าน | trap, PC/SP, .su, stack watermark | ตรวจ stack collision, startup และ unsupported instruction |
| UART ไม่มี banner | image hash, PC, reset, tx waveform | ตรวจ image ติดตั้ง/export ใหม่, UART enable และ clock |
| UART เป็นตัวอักษรผิด | bit-time, SYSINFO clock, host setting | ทำ clock/generic/SDC/baud ให้สอดคล้อง |
| TX มีแต่ simulation console | UART0_SIM_MODE build flags | ปิด simulation-only mode และ rebuild |
| TX ผ่าน แต่ RX ไม่ทำงาน | rx idle/start/stop, input_in[8] | ตรวจ RX pin mapping และ FIFO polling |
| GPIO output สลับ bit | walking-one ที่ core/wrapper/pad | แก้ permutation ที่ตำแหน่งต้นเหตุ |
| อ่าน input เหมือน output ไม่ได้ | gpio_i/gpio_o mapping | ใช้ input ที่ขับจริงหรือ loopback ที่ออกแบบไว้ |
| smoke test timeout หลังเปลี่ยน C | installed image, checker protocol | ใช้ Lab 11 checker ไม่ปิด assert |
| PNR ยังเหมือน smoke เดิม | netlist/image hashes, run state | กลับ export/synthesis ไม่ resume routed state เก่า |

## 11.16 ผลส่งและ acceptance matrix

| Gate | หลักฐาน | เงื่อนไขผ่าน |
|---|---|---|
| configuration | generic/ISA/pin table | clock, memory, ports ตรง baseline |
| C build | compiler log, ELF attributes | exit status สำเร็จ, RV32/ILP32 และ ISA ตรง RTL |
| ROM budget | map, segments, raw image | load image อยู่ใน 4 KiB window |
| RAM/stack budget | sections, call-chain/.su, runtime evidence | data/stack/heapไม่ชนและรวมไม่เกิน 1 KiB |
| image install | cmp และ hashes | generated/installed image เหมือนกัน |
| exported serial test | serial-test.log, waveform | banner/commands/response/GPIO ตรง, ไม่มี framing error/timeout |
| VHDL test | log/waveform | protocol เดียวกันผ่านเมื่อเป็น required gate |
| chip wiring | wrapper/pad test evidence | UART/GPIO map ตรงทุก bit ตาม scope |
| physical rebuild | new run tag และ reports | C ROM netlist ผ่าน required Lab 5–10 checks |
| silicon test | measured clock, serial/pin traces | hardware protocol ผ่าน; ไม่มีชิปให้ NOT_RUN |

ใช้ PASS / FAIL / NOT_RUN / WAIVED ตาม methodology และแนบ evidence path ไม่มี compiler, ไม่มีชิป หรือยังไม่ได้รัน check ไม่ใช่ PASS

## 11.17 คำถามท้าย Lab

1. ทำไม riscv64-unknown-elf-gcc สามารถสร้างโปรแกรมให้ RV32 ได้?
2. ทำไม ELF size ทั้งไฟล์ไม่ใช่ ROM usage?
3. ทำไม .data ใช้ทั้ง ROM load space และ RAM runtime space?
4. ทำไม linker ผ่านยังไม่รับรอง stack safety?
5. ทำไม software division helper จึงไม่เท่ากับเปิด RISC-V M extension?
6. ทำไมเปลี่ยน CLOCK_PERIOD เพียงอย่างเดียวจึงทำให้ UART baud ผิดได้?
7. ทำไม GPIO input register ไม่ใช่ output readback ใน wrapper นี้?
8. ทำไม UART simulation mode ไม่รับรอง serial pin behavior?
9. ทำไม C firmware ใหม่ต้องทำ synthesis ใหม่ใน boot mode 2?
10. ถ้าต้องการ UART upload หลังผลิต ต้องเปลี่ยน memory/boot architecture อะไรบ้าง?

## 11.18 แหล่งอ้างอิงและขอบเขตเวอร์ชัน

อ้างอิงหลักคือ source ที่ตรวจจากแพ็กเกจ NEORV32_IHP_SG13G2_Project.zip และคู่มือ NEORV32_Chip_Implementation_TH.md โดยเฉพาะ Makefile, sw/Makefile, sw/common/common.mk, sw/common/neorv32.ld, neorv32_uart.h/.c, neorv32_gpio.h และ rtl/chip_core.sv

เอกสารออนไลน์ใช้ประกอบภาพรวม ต้องยืนยัน API กับ vendor snapshot ก่อนใช้:

- https://github.com/stnolting/neorv32
- https://stnolting.github.io/neorv32/
- https://stnolting.github.io/neorv32/sw/neorv32__uart_8h.html
- https://stnolting.github.io/neorv32/sw/neorv32__gpio_8h.html

ไม่เปลี่ยน vendor ไปเป็น main โดยอัตโนมัติเพียงเพราะ online documentation ใหม่กว่า
