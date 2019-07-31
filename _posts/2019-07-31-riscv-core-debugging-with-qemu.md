---
layout: post
title: simple risc-v core in verilog (PART1) - debugging with qemu
categories: [debugging, risc-v, verilog]
tags: [risc-v, debugging, qemu, verilog]
fullview: true
---

In this blog series I want to document my experience of writing my own RV32I core in verilog.
Altough there are already a lot of risc-v cores out there I wanted to write my own as I got curious to learn 
verilog and the risc-v instruction set. 

The goal for this series is to get a very simple RV32I core which is synthesizable with the open source FPGA toolchain 
[Yosys](https://github.com/YosysHQ/yosys) and can also be loaded into an iCE40 FPGA.

***

Before proceeding with this tutorial, check that these tools are installed on your system:
- riscv32-*-elf-gcc
- qemu-system-riscv32

***

Before we can even think of writing the first verilog code we need to prepare some things first.  
**Most importantly: We need to find a way to verify our core's correct function.**  
  
Here are our options:
 - [risc-v formal](https://github.com/SymbioticEDA/riscv-formal)
 - [risc-v isa test](https://github.com/riscv/riscv-tests)

risc-v formal is a Formal Verification Framework for risc-v cores by Clifford Wolf. As I am a very beginner in Verilog and don't know much about
formal verification I decided to choose the official risc-v isa test suite.  

Unfortunately we need to do some changes in the test-environment, as the default one has a lot of pre-assumptions we do not have on our core in the first place.  

**Here are the steps:**  

- Prepare your working environment:
{% highlight shell %}
mkdir riscv_test && cd riscv_test
{% endhighlight %}

- Checkout the risc-v isa test repo:
{% highlight shell %}
git clone --recursive https://github.com/riscv/riscv-tests.git  
{% endhighlight %}  

- Now create a file called riscv_test.h and add the following:  

{% highlight c %}
#ifndef _ENV_PHYSICAL_SINGLE_CORE_H
#define _ENV_PHYSICAL_SINGLE_CORE_H

#include "encoding.h"

#define TESTNUM gp

#define RVTEST_RV32U                   \
  .macro init;                         \
  .endm

#define RVTEST_CODE_BEGIN \
	.globl _start;          \
_start:                   \
  call tests_begin_here;  \
                          \
tests_begin_here:         \
  nop;

#define RVTEST_FAIL \
  call _start;

#define RVTEST_PASS       \
  nop;                    \

#define RVTEST_CODE_END   \
  nop;

//-----------------------------------------------------------------------
// Data Section Macro
//-----------------------------------------------------------------------

#define EXTRA_DATA

#define RVTEST_DATA_BEGIN                                               \
        EXTRA_DATA                                                      \
        .pushsection .tohost,"aw",@progbits;                            \
        .align 6; .global tohost; tohost: .dword 0;                     \
        .align 6; .global fromhost; fromhost: .dword 0;                 \
        .popsection;                                                    \
        .align 4; .global begin_signature; begin_signature:

#define RVTEST_DATA_END .align 4; .global end_signature; end_signature:

#endif
{% endhighlight %}

Those macros modify the test behavior as such that if the isa test fails, it will stuck in an endless loop, otherwise the tests finish successfully. So 
the only instructions for this to work is JAL and ADDI (nop).  

- Now prepare a buildscript like this:

{% highlight shell %}
#!/bin/bash

TESTS_DIR="riscv-tests/isa/rv32ui"
ENV_DIR="riscv-tests/env"

if [ "$#" -ne 4 ]; then
	echo "Usage: $0 <test_name> <linker_script> <output_filename> <output_dir>" >&2
	echo "Example (leiwandrv32): ./build_test.sh addi link_leiwandrv32.ld leiwandrv32 test_output"
	echo "Example (qemu): ./build_test.sh addi link_qemu.ld qemu test_output"
	exit 1
fi

TEST_NAME="$1"
LINKER_SCRIPT="$2"
OUTPUT_FILE="$3"
OUTPUT_DIR="$4"

mkdir -p $OUTPUT_DIR

riscv32-none-elf-gcc -march=rv32i \
	-I. -I${TESTS_DIR}/../macros/scalar/ -I${ENV_DIR}/ -Wl,-T,$LINKER_SCRIPT,-Bstatic -ffreestanding -nostdlib \
	-o ${OUTPUT_DIR}/${TEST_NAME}_${OUTPUT_FILE}.elf ${TESTS_DIR}/${TEST_NAME}.S
riscv32-none-elf-objcopy -O binary ${OUTPUT_DIR}/${TEST_NAME}_${OUTPUT_FILE}.elf ${OUTPUT_DIR}/${TEST_NAME}_${OUTPUT_FILE}.bin
{% endhighlight %}

- The last missing part before building is an appropriate linker script. Prepare one like this (link_qemu.ld):  
{% highlight shell %}
OUTPUT_ARCH( "riscv" )
ENTRY(_start)

SECTIONS
{
  . = 0x20400000;
  .text : { *(.text) }
  . = ALIGN(0x400);
  .data : { *(.data) }
  .bss : { *(.bss) }
  _end = .;
}
{% endhighlight %}

- Now you can build the isa tests like this:

{% highlight shell %}
./build_test.sh addi link_qemu.ld qemu test_output
{% endhighlight %}

---

OK, first part of the first part is over. Now let's get down to the real business. As I initially stated we need to find
a way to verify our own risc-v core. To achieve this we will simply run all the isa tests in qemu-system-riscv32 which has
an official port of the SiFive HiFive1 microcontroller. Our assumption is, that the official implementation will deliver us 
the correct CPU core states when running our modified isa tests.  

**So how do we get all the CPU core states from qemu?**  

Answer: We will issue qemu and gdb with some scripting magic to get all CPU core states we want. 

- First prepare a gdb script (step_mult.gdb) like this:

{% highlight gdb %}
# file: step_mult.gdb

define step_mult
    set pagination off

    if $argc == 0
        printf "Please specify end position!!!!!!!!!!!!!!!!\n"
        quit
    else
        set $end_pos = $arg0
    end

    printf "end pos %d\n", $end_pos

    while ($pc < $end_pos)
        stepi
    end

    # One more for the nops at the end
    stepi

    quit
end
{% endhighlight %}

- This can be called like this in combination with qemu:  

{% highlight bash %}
SUCCESS_PC="$(riscv32-none-elf-objdump -S qemu_compiled_files/<test>_qemu.elf | grep "<pass>:" | awk '{print $1}')"
END_PC=$(echo $((16#$SUCCESS_PC)))

qemu-system-riscv32 -nographic -machine sifive_e -kernel qemu_compiled_files/<test>_qemu.elf -d in_asm,cpu -s -S 2> <test>_trace.txt &
riscv32-none-elf-gdb -ex "target remote localhost:1234" -ex "source step_mult.gdb" -ex "step_mult $END_PC" qemu_compiled_files/<test>_qemu.elf
{% endhighlight %}

- Just let it run and kill it with fire after a few seconds:  
{% highlight bash %}
kill -9 $(pidof qemu-system-riscv32)
{% endhighlight %}

The _trace.txt file should now contain all CPU register states of the test run.  


- I've prepared a script which does all the above things automatically for all selected tests:  

{% highlight bash %}
#!/bin/bash


tests=(
lui
auipc
jal
jalr
beq
bne
blt
bge
bltu
bgeu
lb
lh
lw
lbu
lhu
#sb
#sh
#sw
addi
slti
sltiu
xori
ori
andi
slli
srli
srai
add
sub
sll
slt
sltu
xor
srl
sra
or
and
#fence
#fence_i
#ecall
#ebreak
#csrrw
#csrrs
#csrrc
#csrrwi
#csrrsi
#csrrci
)

mkdir -p qemu_compiled_files
mkdir -p qemu_traces
mkdir -p qemu_register_states

for i in "${tests[@]}"
do
    ./build_test.sh $i link_qemu.ld qemu qemu_compiled_files
done

for i in "${tests[@]}"
do
    SUCCESS_PC="$(riscv32-none-elf-objdump -S qemu_compiled_files/${i}_qemu.elf | grep "<pass>:" | awk '{print $1}')"
    echo $SUCCESS_PC
    END_PC=$(echo $((16#$SUCCESS_PC)))
    echo $END_PC

    qemu-system-riscv32 -nographic -machine sifive_e -kernel qemu_compiled_files/${i}_qemu.elf -d in_asm,cpu -s -S 2> qemu_traces/${i}_trace.txt &
    riscv32-none-elf-gdb -ex "target remote localhost:1234" -ex "source step_mult.gdb" -ex "step_mult $END_PC" qemu_compiled_files/${i}_qemu.elf

    kill -9 $(pidof qemu-system-riscv32)
done

for i in "${tests[@]}"
do
    SUCCESS_PC="$(riscv32-none-elf-objdump -S qemu_compiled_files/${i}_qemu.elf | grep "<pass>:" | awk '{print $1}')"

    ./convert_qemu_output.sh qemu_traces/${i}_trace.txt > qemu_register_states/${i}_states.txt
    # delete first two lines as this is some jump in qemu which is not in the elf
    sed -i -e '1,2d' qemu_register_states/${i}_states.txt

    # now delete all lines after the success symbol, as qemu runs freely after we quit gdb
    last_line_number=$(cat qemu_register_states/${i}_states.txt | awk '{print $1}' | grep -n $SUCCESS_PC | cut -d : -f 1)
    last_line_number=$((last_line_number+1))
    sed -i ${last_line_number}',$d' qemu_register_states/${i}_states.txt
done
{% endhighlight %}

OK, fine first part is finished. In the next part we will start developing the actual risc-v core and setup a test-environment which compares the register 
states of our own core with the ones from qemu. The workflow is comparable to a typical test-driven-development flow ;)  
