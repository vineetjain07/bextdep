
Reference Hardware Implementations of Bit Extract/Deposit Instructions
======================================================================

The BEXT (bit extract, aka parallel extract, PEXT) instruction takes as
arguments a value and a bit mask. The bits in the mask indicate which bits in
the value should be extracted. All extracted bits are then packed into the LSB
end of the result:

    uint32_t bext32(uint32_t v, uint32_t mask)
    {
        uint32_t c = 0, m = 1;
        while (mask) {
            uint32_t b = mask & -mask;
            if (v & b)
                c |= m;
            mask -= b;
            m <<= 1;
        }
        return c;
    }

The BDEP (bit deposit, aka parallel deposit, PDEP) instruction performs the
inverse operation. It takes the LSB bits of the value and places them in the
positions marked in the mask argument:

    uint32_t bdep32(uint32_t v, uint32_t mask)
    {
        uint32_t c = 0, m = 1;
        while (mask) {
            uint32_t b = mask & -mask;
            if (v & m)
                c |= b;
            mask -= b;
            m <<= 1;
        }
        return c;
    }

Simple 16-bit example:

    value 0011_0110_0000_0101
    mask  0011_1100_0000_0000
    bext  0000_0000_0000_1101
    bdep  0001_0100_0000_0000

This instructions may be valuable features in bit-manipulation ISA extensions.
However, they are not trivial to implement in an efficient manner. This
repository contains permissively licensed Verilog reference implementations for
32-bit and 64-bit implementations of those instructions using different
architectures:

- Pipelined prallel extract/deposit
- Single-stage parallel extract/deposit
- Sequential in chunks with fixed cycle count
- Sequential with one cycle per set mask bit

Additionally, some of the cores do also implement the generalised bit reversal
(GREV) instruction, since the GREV instruction can in some cases share
resources with BEXT/BDEP. GREV can be used to reverse the bit order in a word,
or swap the two halfs of a word, or any in-between operation (like endianess
convertion), or combinations of them:

    uint32_t grev32(uint32_t x, int k)
    {
        if (k &  1) x = ((x & 0x55555555) <<  1) | ((x & 0xAAAAAAAA) >>  1);
        if (k &  2) x = ((x & 0x33333333) <<  2) | ((x & 0xCCCCCCCC) >>  2);
        if (k &  4) x = ((x & 0x0F0F0F0F) <<  4) | ((x & 0xF0F0F0F0) >>  4);
        if (k &  8) x = ((x & 0x00FF00FF) <<  8) | ((x & 0xFF00FF00) >>  8);
        if (k & 16) x = ((x & 0x0000FFFF) << 16) | ((x & 0xFFFF0000) >> 16);
        return x;
    }

The Verilog code for the cores can be found in `bextdep.v`. The Verilog file
`bextdep_pps.v` is also needed. It is generated by `ppsmaker.py` (`make
bextdep_pps.v`) and contains prefix-sum cores that are used by most of the
cores in `bextdep.v`.

Evaluation
----------

This are the 64 bit Verilog cores:

| Name          |  Gates |    Rel |  Rel+G | Description                               |
|:------------- | ------:| ------:| ------:|:----------------------------------------- |
| `bextdep64g3` |   9670 |   1.98 |   1.98 | 3-stage pipeline with GREV                |
| `bextdep64p3` |   8837 |   1.81 |   2.14 | 3-stage pipeline                          |
| `bextdep64g2` |   8101 |   1.66 |   1.66 | 2-stage pipeline with GREV                |
| `bextdep64p2` |   7178 |   1.47 |   1.80 | 2-stage pipeline                          |
| `bextdep64g1` |   6282 |   1.29 |   1.29 | single-stage with GREV support            |
| `bextdep64p1` |   5282 |   1.08 |   1.42 | single-stage implementation               |
| `bextdep64sh` |   4289 |   0.88 |   1.21 | 4-cycles sequential implementation        |
| `bextdep64sb` |   3481 |   0.71 |   1.05 | 8-cycles sequential implementation        |
| `bextdep64sn` |   3182 |   0.65 |   0.99 | 16-cycles sequential implementation       |
| `bextdep64sx` |   3314 |   0.68 |   1.01 | up-to-64-cycles sequential implementation |
| `bextdep64go` |   1635 |   0.33 |   0.33 | single-stage GREV-only core               |
| `vscalealu64` |   4884 |   1.00 |   1.33 | ALU from V-Scale CPU (for comparison)     |

And this are the 32 bit Verilog cores:

| Name          |  Gates |    Rel |  Rel+G | Description                               |
|:------------- | ------:| ------:| ------:|:----------------------------------------- |
| `bextdep32g3` |   4191 |   1.88 |   1.88 | 3-stage pipeline with GREV                |
| `bextdep32p3` |   3837 |   1.72 |   2.04 | 3-stage pipeline                          |
| `bextdep32g2` |   3432 |   1.54 |   1.54 | 2-stage pipeline with GREV                |
| `bextdep32p2` |   3040 |   1.36 |   1.69 | 2-stage pipeline                          |
| `bextdep32g1` |   2598 |   1.16 |   1.16 | single-stage with GREV support            |
| `bextdep32p1` |   2162 |   0.97 |   1.29 | single-stage implementation               |
| `bextdep32sb` |   1949 |   0.87 |   1.20 | 4-cycles sequential implementation        |
| `bextdep32sn` |   1693 |   0.76 |   1.08 | 8-cycles sequential implementation        |
| `bextdep32sx` |   1543 |   0.69 |   1.02 | up-to-32-cycles sequential implementation |
| `bextdep32go` |    723 |   0.32 |   0.32 | single-stage GREV-only core               |
| `vscalealu32` |   2231 |   1.00 |   1.32 | ALU from V-Scale CPU (for comparison)     |

The gate counts in the tables above are for the cores mapped to NAND, NOR, and
NOT gates, and DFFs, with NOT counting as 0.5 gates and DFFs counting as 4
gates. (See `Makefile` for synthesis scripts.)

The "Rel" column lists the size relative to the the ALU of a V-Scale RISC-V
processor. (The 64-bit version is just the 32 bit ALU with extended bit-width,
it does not support the RV64 `*W` opcodes. See `vscalealu.v`.) The "Rel+G"
column lists the relative size when the size of a GREV core is added to the
cores that don't implement GREV.

The sequential cores require an additional cycle for initialization. So the
initiation interval (II) of the `bextdep32sb` core is 5 and the II of the
`bextdep64sx` core is up to 65 (when all mask bits in the input are set).

Here are the sizes and timings for the cores when mapped to a Xilinx UltraScale
Kintex device (speed grade -3) with Vivado 2017.3. This evaluation also includes
the Rocket "TinyConfig" core for size comparison:

| Name          |   LUTs |    Rel |  Rel+G | Max Freq. |
|:------------- | ------:| ------:| ------:| ---------:|
| `bextdep64g3` |    774 |   0.80 |   0.80 |   451 MHz |
| `bextdep64p3` |    675 |   0.70 |   0.90 |   462 MHz |
| `bextdep64g2` |    762 |   0.79 |   0.79 |   430 MHz |
| `bextdep64p2` |    678 |   0.70 |   0.91 |   406 MHz |
| `bextdep64g1` |    854 |   0.89 |   0.89 |   277 MHz |
| `bextdep64p1` |    733 |   0.76 |   0.96 |   285 MHz |
| `bextdep64sh` |    553 |   0.57 |   0.78 |   351 MHz |
| `bextdep64sb` |    437 |   0.45 |   0.66 |   543 MHz |
| `bextdep64sn` |    314 |   0.33 |   0.53 |   668 MHz |
| `bextdep64sx` |    309 |   0.32 |   0.52 |   421 MHz |
| `bextdep64go` |    195 |   0.20 |   0.20 |  1189 MHz |
| `vscalealu64` |    964 |   1.00 |   1.00 |   713 MHz |

| Name          |   LUTs |    Rel |  Rel+G | Max Freq. |
|:------------- | ------:| ------:| ------:| ---------:|
| `bextdep32g3` |    322 |   0.75 |   0.75 |   521 MHz |
| `bextdep32p3` |    279 |   0.65 |   0.84 |   537 MHz |
| `bextdep32g2` |    318 |   0.74 |   0.74 |   497 MHz |
| `bextdep32p2` |    276 |   0.64 |   0.83 |   497 MHz |
| `bextdep32g1` |    370 |   0.86 |   0.86 |   350 MHz |
| `bextdep32p1` |    283 |   0.66 |   0.85 |   352 MHz |
| `bextdep32sb` |    263 |   0.61 |   0.80 |   545 MHz |
| `bextdep32sn` |    168 |   0.39 |   0.58 |   686 MHz |
| `bextdep32sx` |    158 |   0.37 |   0.56 |   427 MHz |
| `bextdep32go` |     83 |   0.19 |   0.19 |  1116 MHz |
| `vscalealu32` |    431 |   1.00 |   1.00 |   819 MHz |
| `tinyrocket`  |   4280 |   9.93 |  ----- |   211 MHz |


Interfaces
----------

All cores implement the same interface (XLEN=32 or XLEN=64):

    input clock, reset;

    input             din_valid;
    output            din_ready;
    input  [     1:0] din_mode;
    input  [XLEN-1:0] din_value;
    input  [XLEN-1:0] din_mask;

    output            dout_valid;
    input             dout_ready;
    output [XLEN-1:0] dout_result;

The `din*` signals are a valid/ready-style input stream. Data is transferred
in to the core in cycles where valid and ready are high. The core will raise
ready when it is ready to receive data without waiting for an external valid
signal, and ready is guaranteed to stay high until a transfer happens (or the
core is reset).

The `dout*` signals are a valid/ready-style output stream. Data is transferred
out of the core in cycles where valid and ready are high. The core will raise
valid when it is ready to transmit data without waiting for an external ready
signal, and valid is guaranteed to stay high until a transfer happens (or the
core is reset).

The assigned values for `din_mode` are (GREV mode is only valid for cores with
GREV support):

|  Mode | Description |
|:-----:|:----------- |
| 2'b00 | BEXT        |
| 2'b01 | BDEP        |
| 2'b10 | GREV        |
| 2'b11 | reserved    |

All cores have FFs near the `dout_result` output, but may borrow from the
timing path of the incomming `din_mode`, `din_value`, and `din_mask` signals.

References
----------

- http://www.felixcloutier.com/x86/PEXT.html
- http://www.felixcloutier.com/x86/PDEP.html
- http://palms.ee.princeton.edu/PALMSopen/hilewitz04comparing_with_cit.pdf
- http://palms.ee.princeton.edu/PALMSopen/hilewitz06FastBitCompression.pdf
- https://en.wikipedia.org/wiki/Bit_Manipulation_Instruction_Sets#Parallel_bit_deposit_and_extract
- http://programming.sirrida.de/bit_perm.html#bmi2

