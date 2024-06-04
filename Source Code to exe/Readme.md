# Source Code to exe

![image](https://github.com/fahimalshihab/Reverse-Engineering/assets/97816146/8b289b0b-aa62-4352-b66f-da0687265d2f)

Source code hello.cpp
```
#include <iostream>

using namespace std;

int sum(int a, int b) {
  return a + b; // Directly return the sum for efficiency
}

int main() {
  sum(5, 6);
  return 0;
}

```

``` g++ hello.cpp -o  hello.exe ```
To make .o file and .exe file

[hello.o](https://github.com/fahimalshihab/Reverse-Engineering/blob/main/Source%20Code%20to%20exe/hello.o)

[hello.exe](https://github.com/fahimalshihab/Reverse-Engineering/blob/main/Source%20Code%20to%20exe/hello.exe)
```
iftx@iftx-Modern-15-A5M:~/Desktop$ objdump  hello.o -s

hello.o:     file format elf64-x86-64

Contents of section .text:
 0000 f30f1efa 554889e5 897dfc89 75f88b55  ....UH...}..u..U
 0010 fc8b45f8 01d05dc3 f30f1efa 554889e5  ..E...].....UH..
 0020 be060000 00bf0500 0000e800 000000b8  ................
 0030 00000000 5dc3f30f 1efa5548 89e54883  ....].....UH..H.
 0040 ec10897d fc8975f8 837dfc01 753b817d  ...}..u..}..u;.}
 0050 f8ffff00 00753248 8d050000 00004889  .....u2H......H.
 0060 c7e80000 0000488d 05000000 004889c2  ......H......H..
 0070 488d0500 00000048 89c6488b 05000000  H......H..H.....
 0080 004889c7 e8000000 0090c9c3 f30f1efa  .H..............
 0090 554889e5 beffff00 00bf0100 0000e893  UH..............
 00a0 ffffff5d c3                          ...].           
Contents of section .init_array:
 0000 00000000 00000000                    ........        
Contents of section .comment:
 0000 00474343 3a202855 62756e74 75203131  .GCC: (Ubuntu 11
 0010 2e342e30 2d317562 756e7475 317e3232  .4.0-1ubuntu1~22
 0020 2e303429 2031312e 342e3000           .04) 11.4.0.    
Contents of section .note.gnu.property:
 0000 04000000 10000000 05000000 474e5500  ............GNU.
 0010 020000c0 04000000 03000000 00000000  ................
Contents of section .eh_frame:
 0000 14000000 00000000 017a5200 01781001  .........zR..x..
 0010 1b0c0708 90010000 1c000000 1c000000  ................
 0020 00000000 18000000 00450e10 8602430d  .........E....C.
 0030 064f0c07 08000000 1c000000 3c000000  .O..........<...
 0040 00000000 1e000000 00450e10 8602430d  .........E....C.
 0050 06550c07 08000000 1c000000 5c000000  .U..........\...
 0060 00000000 56000000 00450e10 8602430d  ....V....E....C.
 0070 06024d0c 07080000 1c000000 7c000000  ..M.........|...
 0080 00000000 19000000 00450e10 8602430d  .........E....C.
 0090 06500c07 08000000                    .P......
```

<pre>

  The output of objdump shows the contents of the hello.o object file, including:

The file format (ELF64)
The machine code in the .text section
Empty .init_array section
A comment string in the .comment section indicating the compiler version
Notes about the object file in the .note.gnu.property section
Exception handling frame information in the .eh_frame section
Each section is displayed in hexadecimal format, with each line showing 16 bytes of data.
</pre>


- Source Code: This is the starting point, written in a high-level language like C.
- Preprocessor (D): This stage removes comments and directives (lines starting with #) from the code.
- Compiler: In modern compilers, a single step often combines compilation and assembly. The compiler translates the preprocessed code directly into machine code or a very low-level assembly language that's nearly
- machine code. The output is typically an object file (.o file).
- Assembly/Machine Code (.o file): This file contains the machine code instructions the CPU can understand. There might be a minimal level of assembly involved for specific processor architectures, but most of the code is directly executable machine code.
- Linker: The linker combines the object code from the compiler with any necessary library code (prewritten functions).
- Executable (.exe file): The linker generates the final executable file, which contains all the machine code needed to run the program.
- Loader: The loader places the executable file into memory, preparing it for execution by the CPU.
- Memory (Ready to Run): Once loaded into memory, the CPU can fetch and execute the instructions in the executable file, running the program.
