# KnockoutKraken

This is the repository with all the code you need to run KnockoutKraken. KnockoutKraken brings FPGA-accelerated simulation to the QFlex family. We are working towards simulating instrumented code on instrumented hardware at 100 MIPS/core!

KnockoutKraken is composed of three main components: a modified version of QEMU, an instrumented ARM softcore (ARMFlex), and a driver that handles the communication between QEMU and ARMFlex. The vast majority of developers will work on QEMU and/or ARMFlex. QEMU is written in C and can be developed on most Linux machines. ARMFlex is written in Chisel, and while basic testing can be done on most Linux machines, fully simulating and synthesizing the softcore requires an extensive toolchain.

- Go to the `qemu` directory and build QEMU with support for FPGA acceleration ([README](https://github.com/parsa-epfl/qemu/blob/knockoutkraken/README.md)).

- Go to the `image` directory and run `unpack.sh` to unpack the image. You can use your newly built QEMU to try it out ([README](https://github.com/parsa-epfl/images/blob/matmul-knockoutkraken/README.md)).

- Go to the `knockoutkraken` directory and start simulating the FPGA soft core right away with `sbt`. Afterwards, you can synthesize and run your image on an Amazon AWS F1 node. ([README](https://github.com/parsa-epfl/KnockoutKraken/blob/master/README.md)).
