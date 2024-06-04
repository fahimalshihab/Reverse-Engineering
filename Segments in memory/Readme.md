![image](https://github.com/fahimalshihab/Reverse-Engineering/assets/97816146/06160313-93aa-4ab9-8c19-15f992987c26)

![A-high-level-representation-of-a-typical-Harvardbased-MCU-memory-segments](https://github.com/fahimalshihab/Reverse-Engineering/assets/97816146/f0db7f7f-2452-4fbb-888f-804aeedbadd6)
![image](https://github.com/fahimalshihab/Reverse-Engineering/assets/97816146/ca075c9b-d7be-47f7-8432-185a442ae325)

# Program memory
A computer program memory can be largely categorized into two sections: read-only and read/write. This distinction grew from early systems holding their main program in read-only memory such as Mask ROM, EPROM, PROM or EEPROM. As systems became more complex and programs were loaded from other media into RAM instead of executing from ROM, the idea that some portions of the program's memory should not be modified was retained. These became the .text and .rodata segments of the program, and the remainder which could be written to divided into a number of other segments for specific tasks.

## Code
Main article: Code segment
The code segment, also known as text segment, contains executable code and is generally read-only and fixed size.

## Data
The data segment contains initialized static variables, i.e. global variables and local static variables which have a defined value and can be modified. Examples in C include:
```
int i = 3;
char a[] = "Hello World";
static int b = 2023;    // Initialized static global variable
void foo (void) {
  static int c = 2023; // Initialized static local variable
}
```
## BSS
Main article: BSS segment
The BSS segment contains uninitialized static data, both variables and constants, i.e. global variables and local static variables that are initialized to zero or do not have explicit initialization in source code. Examples in C include:
```
static int i;
static char a[12];
```

## .rodata
Contains static constants rather than variables (strings)

## Heap
Contains dynamically allocated memory

## Stack
Contains the call stack,which is a sequence of stack frames 
