---
title: "CHIP-8"
date: 2022-10-08T10:08:00+01:00
draft: false
---

In this post we will look at an implementation of a CHIP-8 emulator. CHIP-8 is an interpreted programming language that was initially used on the COSMAC VIP and Telmac 1800 8-bit microcomputers in the mid-1970s. The programming language has been developed by Joseph Weisbecker. Because of its long history and its simplicity, it is one of the best projects to start with if one is interested in writing emulators.

## Computing systems 101

One could argue that the most basic computer you can make consists of a CPU and a fixed size of memory. Of course you would also want to have a display and some I/O, but let's keep it simple for now.

I'd also like to state that the explanation that is going to come is an over simplification of what really is going on, but we have to start somewhere.

So how does a computer work?

### Memory

Let's start by looking at the memory. Memory is a contiguous set of bytes that can store data. Each of these bytes have an _address_, which allows the CPU to access the byte. The bytes are stored in binary numbers, but when we look at it we display it as a hexadecimal number. To illustrate; the byte `00001111` can be represented as `0f` in hexadecimal.

Now what do we store in this memory? One thing we can store is _data_, such as the pixel data of an image. Another thing we can store are _programs_, which is what we are interested in. To show how a program is stored in memory, we first need to take a look at how we can represent a program in binary. Before we can do that, we actually need a program first!

### Assembly

The low level language of Assembly, the—human readable—language of the CPU, seems confusing at first, but it actually is surprisingly simple. It consists of a lot of _very_ elementary operations. Let's look at a few of these operations:

 * `add R0, 1`: add the value of `1` to the content of the `R0` register. We will talk about registers in a second.
 * `mov R0, 0`: set the content of the `R0` register to `0`.
 * `cmp R0, 10`: check if the content of the `R0` register is equal to `10`. It will set the zero-flag to `1` if it is equal.
 * `jz $0010`: jump to memory address `$0010` (hex) if the zero-flag is `0`, e.g. jump if the comparison resulted in not equal.

With these four operations we can already create a very simple program. A program that counts to 10 and then stops.

{{< highlight asm >}}
mov R0, 0           ; initialize R0 to 0
loop:               ; define a label to jump to
add R0, 1           ; add 1 to R0
cmp R0, 10          ; test if R0 == 10
jz loop             ; goto loop if R0 != 10
halt
{{< /highlight >}}

We can visualize the program with the following state diagram:

{{< mermaid class="text-center" >}}
stateDiagram-v2

    mov: initialize R0 to 0
    add: add 1 to R0
    cmp: test if R0 == 10
    jz: goto loop if R0 != 10

    [*] --> mov
    mov --> add
    add --> cmp
    cmp --> jz
    jz --> add : zero-flag is 0
    jz --> [*] : zero-flag is 1
{{< /mermaid >}}

A label such a `loop:` is used in the assembler instead of a hard-coded memory address. After running this program, the content of the `R0` register is `10`. Now that we have a program, how do we translate this into the bytes that need to go in memory?

We can do this by assigning a byte-value to every instruction. However, the `add`, `mov`, and `cmp` instructions have two arguments, while `jz` only has one. Because of this each instruction needs a different number of bytes. This is important for later. If we create a table with the encoding:

| Opcode           | Data                 | Bytes needed |
| ---------------- | -------------------- | ------------ |
| `mov reg, const` | `0x01` `reg` `const` | 3 bytes      |
| `add reg, const` | `0x02` `reg` `const` | 3 bytes      |
| `cmp reg, const` | `0x03` `reg` `const` | 3 bytes      |
| `jz const`       | `0x04` `const`       | 2 bytes      |

For demonstrative purposes, I assigned increasing hexadecimal values to each instruction (skipping `0x00`). Finally, the `R0` register is encoded as `0x00` and all the of the constants can be encoded by converting the from decimal to hexadecimal. With this encoding in place, we can convert the program into a set of bytes.

```
mov R0, 0           0x01 0x00 0x00
loop:               this is pointing to the 4th byte (0x03)
add R0, 1           0x02 0x00 0x01
cmp R0, 10          0x03 0x00 0x0a
jz loop             0x04 0x03
```

Which gives us the complete program as a hexadecimal string. If we store this as bytes in a file, it is known as a _binary_.

```
01 00 00 02 00 01 03 00 0a 04 03
```

Note that we came up with an encoding ourselves, but in a real scenario you would use the data sheet that comes with a CPU to see which hexadecimal values are assigned to each opcode. We also assume that each operation is 8-bit, which is not true for a processor where 32-bit or 64-bit is used. We simplified it for demonstrative purposes.

### The CPU

Now that we have a program that we can load into memory, it is time to look at the CPU, which is going to _execute_ the program.

A CPU has a set of registers that it uses to store data, a byte in this case. Various operations such as `add` operate on the data that is in those registers. Another very important thing a CPU needs is a _program counter_. A program counter is an additional register which holds a memory address that contains the instruction it has to execute. Now to execute a program, the CPU uses a fetch-decode-execute loop. This loop does the following things:

 1. **Fetch** the instruction that the program counter is pointing to.
 2. Increment the program counter by the number of bytes that is needed for the instruction.
 3. **Decode** the instruction so that it can be executed. Electronically, the instruction is decoded such that the correct wiring is activated, but for an emulator this step becomes pretty simple.
 4. **Execute** the instruction.
 5. Go back to (1) where we fetch the instruction that the updated program counter is pointing at.

Taking a step back, one can see how the CPU is reading all the instruction from memory and executing them. The CPU itself is actually pretty stupid, because it is only executing whatever we feed into it by pointing the program counter to the memory address of whatever we want to execute. With this basic understanding it is time to define the CPU.

The CPU that can execute our program is going to be very simple. It consists of the following components:

 * A single register `R0` that is used to store data and perform operations on.
 * A program counter `PC` that points to the memory address that we are currently executing.
 * A zero-flag `ZF` that is set by the `cmp` instruction and used by the `jz` instruction.

We have now defined enough information to actually emulate this CPU with some code.

### Executing the program

To do so, we create a program in C#. We start with a `CPU` class. To it, we will add some constants that represent the opcodes, because that is much more readable than hexadecimals. We will also load in the program by defining an array of bytes. The size of the memory is 11 bytes in this example, which is fine for this demonstration.

{{< highlight csharp >}}
class CPU
{
    const byte OPCODE_MOV = 0x01; const byte OPCODE_ADD = 0x02;
    const byte OPCODE_CMP = 0x03; const byte OPCODE_JZ  = 0x04;

    byte[] Memory = new byte[] 
    { 
        0x01, 0x00, 0x00, 
        0x02, 0x00, 0x01, 
        0x03, 0x00, 0x0a, 
        0x04, 0x03 
    };

    // ...
}
{{< /highlight >}}

Next we define the register `R0`, the program counter, zero flag, and an array that contains how many bytes each instruction needs.

{{< highlight csharp >}}
byte[] Registers = new byte[1];
byte ProgramCounter;
bool ZeroFlag;
byte[] BytesPerInstruction = new byte[] { 0, 3, 3, 3, 2 };
{{< /highlight >}}

With that in place we can start with the fetch-decode-execute loop. First we define the fetch method which will return the first byte that the program counter is pointing to, which is the opcode. At this point, we are also incrementing the program counter with the number of bytes that are needed for the instruction that we have fetched. The number of bytes can be found in the `BytesPerInstruction` array we have defined earlier.

{{< highlight csharp >}}
byte Fetch()
{
    byte opcode = Memory[ProgramCounter];
    ProgramCounter += (byte)BytesPerInstruction[opcode];
    return opcode;
}
{{< /highlight >}}

It is also possible to update the program counter after executing each instruction, but in my opinion, this makes it more error prone because we have to update it after every instruction.

Now it is time to decode the instruction. In this context—unlike the electronic case—it means that we need to read the arguments that the instruction requires from memory.

{{< highlight csharp >}}
byte[] Decode(byte opcode)
{
    int instructionSize = BytesPerInstruction[opcode];
    byte[] args = new byte[instructionSize - 1];

    for (int i = 0; i < instructionSize - 1; i++)
    {
        args[i] = Memory[ProgramCounter - instructionSize + i + 1];
    }

    return args;
}
{{< /highlight >}}

Finally, it is time to execute the instruction. Based on each opcode, we will update the registers, program counter, and zero flag.

{{< highlight csharp >}}
void Execute(byte opcode, byte[] args)
{
    switch (opcode)
    {
        case OPCODE_MOV: 
            Registers[args[0]] = args[1];                
            break;
        case OPCODE_ADD: 
            Registers[args[0]] += args[1];               
            break;
        case OPCODE_CMP: 
            ZeroFlag = Registers[args[0]] == args[1];    
            break;
        case OPCODE_JZ: 
            if (!ZeroFlag)
                ProgramCounter = args[0];      
            break;
    }
}
{{< /highlight >}}

And to tie it all together we will create the fetch-decode-execute loop.

{{< highlight csharp >}}
public void Run()
{
    while (ProgramCounter < Memory.Length)
    {
        byte opcode = Fetch();
        byte[] args = Decode(opcode);
        Execute(opcode, args);
    }
}
{{< /highlight >}}

Combining all of the above results in the code below. The code is self-sufficient and will execute the program that we have defined earlier. 

{{< highlight csharp >}}
class CPU
{
    const byte OPCODE_MOV = 0x01; const byte OPCODE_ADD = 0x02;
    const byte OPCODE_CMP = 0x03; const byte OPCODE_JZ  = 0x04;

    byte[] Memory = new byte[] 
    { 
        0x01, 0x00, 0x00, 
        0x02, 0x00, 0x01, 
        0x03, 0x00, 0x0a, 
        0x04, 0x03 
    };
    
    byte[] Registers = new byte[1];
    byte ProgramCounter;
    bool ZeroFlag;
    byte[] BytesPerInstruction = new byte[] { 0, 3, 3, 3, 2 };

    public void Run()
    {
        while (ProgramCounter < Memory.Length)
        {
            byte opcode = Fetch();
            byte[] args = Decode(opcode);
            Execute(opcode, args);
        }
    }

    byte Fetch()
    {
        byte opcode = Memory[ProgramCounter];
        ProgramCounter += (byte)BytesPerInstruction[opcode];
        return opcode;
    }

    byte[] Decode(byte opcode)
    {
        int instructionSize = BytesPerInstruction[opcode];
        byte[] args = new byte[instructionSize - 1];

        for (int i = 0; i < instructionSize - 1; i++)
        {
            args[i] = Memory[ProgramCounter - instructionSize + i + 1];
        }

        return args;
    }

    void Execute(byte opcode, byte[] args)
    {
        switch (opcode)
        {
            case OPCODE_MOV: 
                Registers[args[0]] = args[1];                
                break;
            case OPCODE_ADD: 
                Registers[args[0]] += args[1];               
                break;
            case OPCODE_CMP: 
                ZeroFlag = Registers[args[0]] == args[1];    
                break;
            case OPCODE_JZ: 
                if (!ZeroFlag)
                    ProgramCounter = args[0];      
                break;
        }
    }
}
{{< /highlight >}}

This example contains the core ideas that you will see in most computing systems. After running this program you can see that `R0` contains the value `10`. If you don't believe it, or see it, try to check it with the debugger.

With the basic understanding of how a computer system works, we will take a look at the implementation of CHIP-8.

## CHIP-8

To emulate CHIP-8 we are going to use a console application in C#. We could opt for a graphics library to visualize the screen and handle the keyboard and audio, but to keep it more simple I decided to stick to the console. I've done other projects where I have implemented a graphics library which made me decide that I didn't want to focus on those aspects. Due to this the implementation will lack support for the keyboard and audio. Perhaps I'll add this later, but I think I won't.

With the disclaimer out of the way, lets look at the components that we need to emulate CHIP-8.

### CPU

The CPU will have a total of 16 registers that are a byte each. Because each register is only one byte, each register can hold a value between 0 and 255 (inclusive). The registers are called `V0`, `V1`, `V2`, up to `VF`. In the instructions you will see them referenced as `VX` or `VY`.

Additionally, the CPU will also have a stack. Usually the stack is stored somewhere in memory, but because we are emulating it, we can store it in an additional array. Each element on the stack is 16-bit. The reason for this is because the stack is used to store memory addresses, which is used to call subroutines and return from them. The total addressable memory is only 12-bit though, which gives a total of 4Kb (4096 bytes). The size of the stack is 16 elements, which is what almost all CHIP-8 programs are programmed for. The stack also comes with a stack pointer `SP`, which tells us what the current element is on the stack.

Finally, there is a program counter `PC` which is 16-bit, because it needs to hold the 12-bit memory addresses.

### Memory

As stated above, there is a total of 4Kb of memory, which is 4096 bytes. Conveniently this is addressable by 12-bits. Most CHIP-8 programs stay within this memory limit, but you could use the full 16-bit address space if you'd like.

Programs are loaded starting at memory address `0x200`, or at 512 in decimal. Programs expect this! The reason for this is that historically the interpreter was loaded at the start of the memory (`0x00`—`0xff`), but we are emulating it ourselves, so we are going to let it be empty. Note though, at address `0x50`, there is some sprite data for all hexadecimal digits, but this can also be placed elsewhere in memory. I sticked with `0x50` out of convention.

### Display

The display is a whopping 64 by 32 pixels. I stored the screen buffer in an additional 1D array that is $64 \times 32 = 2048$ bytes. Although the only values that are allowed are `0` and `1`, I opted for a byte anyway because perhaps we want add colors or write character data instead.

Updating the display is actually done with a seperate instruction, as well as clearing the display buffer. If this instruction is executed, we write the screen buffer to the display, which is the console. The following code changes the dimensions of the console, which is needed to display the screen buffer correctly. 

{{< highlight csharp >}}
#pragma warning disable CA1416
Console.SetWindowSize(Chip8.ScreenWidth, Chip8.ScreenHeight + 1);
Console.BufferWidth = Chip8.ScreenWidth;
Console.BufferHeight = Chip8.ScreenHeight + 1;
#pragma warning restore CA1416
{{< /highlight >}}

Originally the display is updated at 60 Hz, but we are only going to update the display if the screen buffer has been changed.

### Timing

There are also two different timers that run independently from the fetch-decode-execute loop. The first timer is a delay timer. This timer runs at 60Hz and decreases the timer by one on each tick, so every 1/60th of a second. The timer doesn't do anything on itself, but it is used by the program. The program can set the timer and read the value that is left in the timer.

The second timer is an audio timer. This timer runs on the same interval as the delay timer. This timer makes the computer beep as long as the timer is above zero. However, because we are not using audio, this has not been implemented.

{{< highlight csharp >}}
    Timer InitializeTimer()
    {
        var timer = new Timer();
        timer.Interval = 1000 / DelayTimerHz;
        timer.AutoReset = true;
        timer.Elapsed += OnTick;
        timer.Start();
        return timer;
    }

    void OnTick(object? sender, ElapsedEventArgs args)
    {
        if (DelayTimer > 0) DelayTimer--;
    }
}
{{< /highlight >}}

## Instructions

Each instruction for CHIP-8 is two bytes. If we fetch the instruction, we are going to fetch two bytes and increment the program counter by 2. There are a few instructions that will increase the program counter by an additional two, such as the if instructions.

An instruction is 16-bit, and each of the nibbles (4-bits) is used to encode data. The first nibble is the most important once since that determines which instruction to execute. The interpretation of the other three nibbles depends on the instruction that is being executed.

If we fetch the two bytes needed for an instruction, we have to bit shift the first byte by 8 to the left, and _AND_ it with the other byte to get the 2 byte instruction.

{{< highlight csharp >}}
(ushort)(Ram[PC++] << 8 | Ram[PC++])
{{< /highlight >}}

The instruction is then decoded into the followings parts: `X`, `Y`, `N`, `NN`, and `NNN`. The table belows shows how the nibbles should be decoded into the seperate parts.

| Nibble 1 | Nibble 2 | Nibble 3 | Nibble 4 |
| -------- | -------- | -------- | -------- |
| `Kind`   | `X`      | `Y`      | `N`      |
|          |          | `N`      | `N`      |
|          | `N`      | `N`      | `N`      |

The `Kind` is used to select the instruction. The `X` and `Y` part refer to a register. The last nibble `N` is a 4-bit constant, the last two nibbles `NN` is an 8-bit constant, and the last three nibbles `NNN` is a 12-bit memory address. Note that all of them fit in a `byte`, but the `NNN` part does not! You should use a `ushort` for that. Yes, I screwed up there.

When decoding the instruction, it is useful to decompose it into those parts, since you'll need different parts of it during the execution. We will define a C# record for that to hold the decoded instruction:

{{< highlight csharp >}}
record Instruction(byte Kind, byte X, byte Y, byte N, byte NN, ushort NNN);
{{< /highlight >}}

If you take a look at the opcodes, `7XNN` for example, the naming convention should become clear using the decoding table from above. Having looked at multiple instructions sets, the clever encoding of all this data into two bytes is something I find really cool about CHIP-8.

## Font

CHIP-8 has a default font built into the memory that is used to display hexadecimal characters. A character has a width of 4 pixels and a height of 5 pixels. Each digit in the font consists of 5 bytes of data, where only the first nibble is used. This nibble contains the four pixels that have to be on for the character. Originally this font is loaded in starting at memory adress `0x50`. This example below shows how the character `A` is constructed.

```
0xF0    11110000
0x90    10010000
0xF0    11110000
0x90    10010000
0x90    10010000
```

The following method loads this font into memory at a given offset.

{{< highlight csharp >}}
void LoadFontAtAddress(ushort offset)
{
    byte[] font =
    {
        0xF0, 0x90, 0x90, 0x90, 0xF0, // 0
        0x20, 0x60, 0x20, 0x20, 0x70, // 1
        0xF0, 0x10, 0xF0, 0x80, 0xF0, // 2
        0xF0, 0x10, 0xF0, 0x10, 0xF0, // 3
        0x90, 0x90, 0xF0, 0x10, 0x10, // 4
        0xF0, 0x80, 0xF0, 0x10, 0xF0, // 5
        0xF0, 0x80, 0xF0, 0x90, 0xF0, // 6
        0xF0, 0x10, 0x20, 0x40, 0x40, // 7
        0xF0, 0x90, 0xF0, 0x90, 0xF0, // 8
        0xF0, 0x90, 0xF0, 0x10, 0xF0, // 9
        0xF0, 0x90, 0xF0, 0x90, 0x90, // A
        0xE0, 0x90, 0xE0, 0x90, 0xE0, // B
        0xF0, 0x80, 0x80, 0x80, 0xF0, // C
        0xE0, 0x90, 0x90, 0x90, 0xE0, // D
        0xF0, 0x80, 0xF0, 0x80, 0xF0, // E
        0xF0, 0x80, 0xF0, 0x80, 0x80  // F
    };

    Array.Copy(font, 0, Ram, offset, font.Length);
}
{{< /highlight >}}

CHIP-8 has an instruction that stores the binary digit representation `FX33` which extracts the three seperate digits of a byte (0-255) to a location in memory. This is pretty useful to display the contents of a register.

## Fetch, decode, execute

{{< highlight csharp >}}
using System.Timers;
using System.Diagnostics;
using Timer = System.Timers.Timer;


class Chip8Mini
{
    public const int ScreenWidth = 64;
    public const int ScreenHeight = 32;
    public const uint MemorySize = 4096;
    public const uint StackSize = 16;
    public const uint NumRegisters = 16;
    public const uint DelayTimerHz = 60;
    public const ushort ProgramStart = 0x200;
    public const ushort FontOffset = 0x50;
    public const byte FilledPixel = (byte)219;
    public const byte EmptyPixel = (byte)0;
    public const byte Carry = 0xf;

    byte[] V, Ram, Screen;
    ushort PC, SP, I;
    ushort[] Stack;
    byte DelayTimer;
    Timer Timer;
    Display Display;
    Random Rng;

    public Chip8Mini(Stream stdOut)
    {
        Ram = new byte[MemorySize];
        Screen = new byte[ScreenWidth * ScreenHeight];
        Stack = new ushort[StackSize];
        V = new byte[NumRegisters];
        Display = new Display(stdOut);
        Rng = new Random();
        Timer = InitializeTimer();
        LoadFontAtAddress(FontOffset);
    }

    // InitializeTimer, OnTick

    public void Start()
    {
        PC = ProgramStart;
        Run();
    }

    void Run()
    {
        for (;;)
        {
            ushort opcode = Fetch();
            Instruction instruction = Decode(opcode);
            Execute(instruction);
        }
    }

    // ...
}
{{< /highlight >}}

{{< highlight csharp >}}
ushort Fetch()
{
    return (ushort)(Ram[PC++] << 8 | Ram[PC++]);
}
{{< /highlight >}}

{{< highlight csharp >}}
public static Instruction Decode(ushort opcode)
{
    return new Instruction(
        (byte)  ((opcode & 0xF000) >> 12),
        (byte)  ((opcode & 0x0F00) >>  8),
        (byte)  ((opcode & 0x00F0) >>  4),
        (byte)  ( opcode & 0x000F),
        (byte)  ( opcode & 0x00FF),
        (ushort)( opcode & 0x0FFF));
}
{{< /highlight >}}

{{< highlight csharp >}}
void Execute(Instruction instruction)
{
    var (kind, x, y, n, nn, nnn) = instruction;

    switch (kind, x, y, n)
    {
        // ...
    }
}
{{< /highlight >}}

## Loading ROMs

Now that we have implemented the minimum of CHIP-8, we need to load in a program. A program is stored as a ROM image (read-only memory). This is a file that contains a stream of bytes, which can be viewed as the hexadecimal string from the _Assembly_ section.

To do that we will use the following utility method to read the ROM from the disk into memory.

{{< highlight csharp >}}
class ROM
{
    public byte[] Program;
    public string Filepath = string.Empty;

    public ROM(long size)
    {
        Program = new byte[size];
    }

    public static ROM FromFile(string filepath)
    {
        using var stream = new FileStream(filepath, FileMode.Open);
        var rom = new ROM(stream.Length);
        stream.Read(rom.Program, 0, (int)stream.Length);
        rom.Filepath = filepath;
        return rom;
    }
}
{{< /highlight >}}

The following method will copy the data of the ROM into the memory of CHIP-8.

{{< highlight csharp >}}
public void LoadROM(ROM rom)
{
    Array.Copy(rom.Program, 0, Ram, ProgramStart, rom.Program.Length);
}
{{< /highlight >}}

We will use this to load `test_opcode.ch8` into memory, which I will explain in the next section. 
You can download the rom here.

<a href="/assets/chip-8/roms/test_opcode.ch8" class="book-btn" download>test_opcode.ch8</a>

## Passing the test

![test opcode screenshot](/chip-8-test-rom.png)



## Opcodes

{{< highlight csharp >}}
void Execute(Instruction instruction)
{
    var (kind, x, y, n, nn, nnn) = instruction;

    switch (kind, x, y, n)
    {
        case (0x0, 0x0, 0x0, 0x0): Console.ReadKey();                           break;
        case (0x0, 0x0, 0xe, 0x0): Array.Fill<byte>(Screen, 0);                       
                                   __c8_display();                              break;
        case (0x0, 0x0, 0xe, 0xe): PC = Stack[--SP];                            break;
        case (0x1,   _,   _,   _): PC = nnn;                                    break;
        case (0x2,   _,   _,   _): Stack[SP++] = PC;                                  
                                   PC = nnn;                                    break;
        case (0x3,   _,   _,   _): if (V[x] == nn) PC += 2;                     break;
        case (0x4,   _,   _,   _): if (V[x] != nn) PC += 2;                     break;
        case (0x5,   _,   _, 0x0): if (V[x] == V[y]) PC += 2;                   break;
        case (0x6,   _,   _,   _): V[x] = nn;                                   break;
        case (0x7,   _,   _,   _): V[x] += nn;                                  break;
        case (0x8,   _,   _, 0x0): V[x] = V[y];                                 break;
        case (0x8,   _,   _, 0x1): V[x] |= V[y];                                break;
        case (0x8,   _,   _, 0x2): V[x] &= V[y];                                break;
        case (0x8,   _,   _, 0x3): V[x] ^= V[y];                                break;
        case (0x8,   _,   _, 0x4): __c8_add(x, y);                              break;
        case (0x8,   _,   _, 0x5): __c8_sub(x, y);                              break;
        case (0x8,   _,   _, 0x6): __c8_shr(x);                                 break;
        case (0x8,   _,   _, 0x7): __c8_sub(y, x);                              break;
        case (0x8,   _,   _, 0xe): __c8_shl(x);                                 break;
        case (0x9,   _,   _, 0x0): if (V[x] != V[y]) PC += 2;                   break;
        case (0xa,   _,   _,   _): I = nnn;                                     break;
        case (0xb,   _,   _,   _): PC = (ushort)(nnn + V[0]);                   break;
        case (0xc,   _,   _,   _): V[x] = (byte)(Rng.Next(0, 0xFF) & nn);       break;
        case (0xd,   _,   _,   _): __c8_sprite(V[x], V[y], n);                        
                                   __c8_display();                              break;
        case (0xe,   _, 0x9, 0xe): /* KeyDown(x) */                             break;
        case (0xe,   _, 0xa, 0x1): /* KeyUp(x)   */ PC += 2;                    break;
        case (0xf,   _, 0x9, 0x7): V[x] = DelayTimer;                           break;
        case (0xf,   _, 0x0, 0xa): __c8_read_key(x);                            break;
        case (0xf,   _, 0x1, 0x5): DelayTimer = V[
            x];                           break;
        case (0xf,   _, 0x1, 0x8): SoundTimer = V[x];                           break;
        case (0xf,   _, 0x1, 0xe): I += V[x];                                   break;
        case (0xf,   _, 0x2, 0x9): I = (byte)(5 * V[x] + FontOffset);           break;
        case (0xf,   _, 0x3, 0x3): __c8_bcd(x);                                 break;
        case (0xf,   _, 0x5, 0x5): for (int i = 0; i <= x; i++)                       
                                       Ram[I + i] = V[i];                       break;
        case (0xf,   _, 0x6, 0x5): for (int i = 0; i <= x; i++)                       
                                       V[i] = Ram[I + i];                       break;
    }
}
{{< /highlight >}}

### 0000 — Debug break

I have added this instruction myself. It stops the execution when it encounter two zero bytes, and waits for a key press.

{{< highlight csharp >}}
case (0x0, 0x0, 0x0, 0x0): Console.ReadKey();
{{< /highlight >}}

### 00E0 — Clear display

{{< highlight csharp >}}
case (0x0, 0x0, 0xe, 0x0): Array.Fill<byte>(Screen, 0);
                           __c8_display();
{{< /highlight >}}

{{< highlight csharp >}}
void __c8_display()
{
    Display.ResetCursor();
    Display.Write(Screen);
}
{{< /highlight >}}

### 00EE — Return from subroutine

{{< highlight csharp >}}
case (0x0, 0x0, 0xe, 0xe): PC = Stack[--SP];
{{< /highlight >}}

### 1NNN — Jump to NNN

{{< highlight csharp >}}
case (0x1,   _,   _,   _): PC = nnn;
{{< /highlight >}}

### 2NNN — Call subroutine at NNN 

{{< highlight csharp >}}
case (0x2,   _,   _,   _): Stack[SP++] = PC;
                           PC = nnn;
{{< /highlight >}}

### 3XNN — Skip next if VX == NN

{{< highlight csharp >}}
case (0x3,   _,   _,   _): if (V[x] == nn) PC += 2;
{{< /highlight >}}

### 4XNN — Skip next if VX != NN

{{< highlight csharp >}}
case (0x4,   _,   _,   _): if (V[x] != nn) PC += 2;
{{< /highlight >}}

### 5XY0 — Skip next if VX == VY

{{< highlight csharp >}}
case (0x5,   _,   _, 0x0): if (V[x] == V[y]) PC += 2;
{{< /highlight >}}

### 6XNN — Set VX = NN

{{< highlight csharp >}}
case (0x6,   _,   _,   _): V[x] = nn;
{{< /highlight >}}

### 7XNN — Add NN to VX

{{< highlight csharp >}}
case (0x7,   _,   _,   _): V[x] += nn;
{{< /highlight >}}

### 8XY0 — VX = VY 

{{< highlight csharp >}}
case (0x8,   _,   _, 0x0): V[x] = V[y];
{{< /highlight >}}

### 8XY1 — VX = VX OR VY

{{< highlight csharp >}}
case (0x8,   _,   _, 0x1): V[x] |= V[y];
{{< /highlight >}}

### 8XY2 — VX = VX AND VY

{{< highlight csharp >}}
case (0x8,   _,   _, 0x2): V[x] &= V[y];
{{< /highlight >}}

### 8XY3 — VX = VX XOR VY

{{< highlight csharp >}}
case (0x8,   _,   _, 0x3): V[x] ^= V[y];
{{< /highlight >}}

### 8XY4 — VX = VX + VY

{{< highlight csharp >}}
case (0x8,   _,   _, 0x4): __c8_add(x, y);
{{< /highlight >}}

{{< highlight csharp >}}
void __c8_add(int x, int y)
{
    V[Carry] = (byte)((V[x] + V[x] > 0xFF) ? 1 : 0);
    V[x] = (byte)((V[x] + V[y]) % 0x100); 
}
{{< /highlight >}}

### 8XY5 — VX = VX - VY

{{< highlight csharp >}}
case (0x8,   _,   _, 0x5): __c8_sub(x, y);
{{< /highlight >}}

{{< highlight csharp >}}
void __c8_sub(int x, int y)
{
    V[Carry] = 1;
    if (V[y] <= V[x]) V[x] -= V[y];
    else
    {
        V[Carry] = 0;
        V[x] = (byte)(V[x] - V[y] + 0x100);
    }
}
{{< /highlight >}}

### 8XY6 — VX = VX SHR 1 

{{< highlight csharp >}}
case (0x8,   _,   _, 0x6): __c8_shr(x);
{{< /highlight >}}

{{< highlight csharp >}}
void __c8_shr(int x)
{
    V[Carry] = (byte)(V[x] & 1);
    V[x] >>= 1;
}
{{< /highlight >}}

### 8XY7 — VX = VY - VX

{{< highlight csharp >}}
case (0x8,   _,   _, 0x7): __c8_sub(y, x);
{{< /highlight >}}

The implementation of `__c8_sub` can be found in `8XY5`.

### 8XYE — VX = VX SHL 1

{{< highlight csharp >}}
case (0x8,   _,   _, 0xe): __c8_shl(x);
{{< /highlight >}}

{{< highlight csharp >}}
void __c8_shl(int x)
{
    V[Carry] = (byte)((V[x] & (1 << 3)) >> 3);
    V[x] <<= 1;
}
{{< /highlight >}}

### 9XY0 — Skip next if VX != VY

{{< highlight csharp >}}
case (0x9,   _,   _, 0x0): if (V[x] != V[y]) PC += 2;
{{< /highlight >}}

### ANNN — Set i = NNNN

{{< highlight csharp >}}
case (0xa,   _,   _,   _): I = nnn;
{{< /highlight >}}

### BNNN — Jump to NNN + V0

{{< highlight csharp >}}
case (0xb,   _,   _,   _): PC = (ushort)(nnn + V[0]);
{{< /highlight >}}

### CXNN — Random number

{{< highlight csharp >}}
case (0xc,   _,   _,   _): V[x] = (byte)(Rng.Next(0, 0xFF) & nn);
{{< /highlight >}}

### DXYN — Draws a sprite at (VX,VY)

{{< highlight csharp >}}
case (0xd,   _,   _,   _): __c8_sprite(V[x], V[y], n);
                           __c8_display();
{{< /highlight >}}

{{< highlight csharp >}}
void __c8_sprite(int vx, int vy, int n)
{
    Debug.Assert(n > 0); // Case not implemented.
    V[Carry] = 0;
    int x = vx & (ScreenWidth - 1);
    int y = vy & (ScreenHeight - 1);
    for (int i = 0; i < n && y < ScreenHeight; i++, y++, x = vx & (ScreenWidth - 1))
        for (int bit = 7; bit >= 0 && x < ScreenWidth; bit--, x++)
            if ((Ram[I + i] & (1 << bit)) > 0 && __c8_flip_pixel(x, y)) 
                V[Carry] = 1;
}
{{< /highlight >}}


The implementation of `__c8_display` can be found in `00E0`.

### EXA1 — Skip if key VX not pressed

Because we do not have a keyboard, it is not possible to press a key, so all keys will always be not pressed. Because of this we can just increment the program counter by 2. It doesn't really implement the instruction, but it covers the case when a key a not been pressed.

{{< highlight csharp >}}
case (0xe,   _, 0xa, 0x1): PC += 2;
{{< /highlight >}}

### FX07 — VX = Delay timer

{{< highlight csharp >}}
case (0xf,   _, 0x9, 0x7): V[x] = DelayTimer;
{{< /highlight >}}

### FX0A — Waits a keypress

{{< highlight csharp >}}
case (0xf,   _, 0x0, 0xa): __c8_read_key(x);
{{< /highlight >}}

{{< highlight csharp >}}
void __c8_read_key(int x)
{
    V[x] = Console.ReadKey().KeyChar switch
    {
        '1' => 0x1, '2' => 0x2, '3' => 0x3, '4' => 0xc,
        'q' => 0x4, 'w' => 0x5, 'e' => 0x6, 'r' => 0xd,
        'a' => 0x7, 's' => 0x8, 'd' => 0x9, 'f' => 0xe,
        'z' => 0xa, 'x' => 0x0, 'c' => 0xb, 'v' => 0xf,
        _   => 0x0
    };    
}
{{< /highlight >}}

### FX15 — Delay timer = VX

{{< highlight csharp >}}
case (0xf,   _, 0x1, 0x5): DelayTimer = V[x];
{{< /highlight >}}

### FX1E — I = I + VX

{{< highlight csharp >}}
case (0xf,   _, 0x1, 0xe): I += V[x];
{{< /highlight >}}

### FX29 — I points to font sprite

{{< highlight csharp >}}
case (0xf,   _, 0x2, 0x9): I = (byte)(5 * V[x] + FontOffset);
{{< /highlight >}}

### FX33 — Store BCD representation

{{< highlight csharp >}}
case (0xf,   _, 0x3, 0x3): __c8_bcd(x);
{{< /highlight >}}

{{< highlight csharp >}}
void __c8_bcd(int x)
{
    Ram[I    ] = (byte)( V[x] / 100);
    Ram[I + 1] = (byte)((V[x] / 10 ) % 10);
    Ram[I + 2] = (byte)((V[x] % 100) % 10);          
}
{{< /highlight >}}

### FX55 — Save V0...VX in memory

{{< highlight csharp >}}
case (0xf,   _, 0x5, 0x5): for (int i = 0; i <= x; i++)
                               Ram[I + i] = V[i]; 
{{< /highlight >}}

### FX65 — Load V0...VX from memory

{{< highlight csharp >}}
case (0xf,   _, 0x6, 0x5): for (int i = 0; i <= x; i++)
                               V[i] = Ram[I + i];
{{< /highlight >}}


## Disassembly

Now that we implemented all the instructions, we can also disassemble a binary back into something readable. Doing so is rather straightforward. We make a copy of the switch statement in the execute method, and replace the operation by the returning of a string. 

I have included the code below that can be used to disassemble a binary. It disassembles back to something similar to Octo, which is a language that can be used to program CHIP-8, which we will look at in the end of the post.

{{< highlight csharp >}}
switch (kind, x, y, n)
{
    /* 0000 */ case (0x0, 0x0, 0x0, 0x0): return "halt";                           
    /* 00E0 */ case (0x0, 0x0, 0xe, 0x0): return "clear";                          
    /* 00EE */ case (0x0, 0x0, 0xe, 0xe): return "return";                         
    /* 1NNN */ case (0x1,   _,   _,   _): return $"jump {nnn}";                    
    /* 2NNN */ case (0x2,   _,   _,   _): return $"call {nnn}";                    
    /* 3XNN */ case (0x3,   _,   _,   _): return $"if R[{x}] != {nn} then";        
    /* 4XNN */ case (0x4,   _,   _,   _): return $"if R[{x}] == {nn} then";        
    /* 5XY0 */ case (0x5,   _,   _, 0x0): return $"if R[{x}] != R[{y}] then";      
    /* 6XNN */ case (0x6,   _,   _,   _): return $"R[{x}] = {nn}";                 
    /* 7XNN */ case (0x7,   _,   _,   _): return $"R[{x}] += {nn}";                
    /* 8XY0 */ case (0x8,   _,   _, 0x0): return $"R[{x}] = R[{y}]";               
    /* 8XY1 */ case (0x8,   _,   _, 0x1): return $"R[{x}] |= R[{y}]";              
    /* 8XY2 */ case (0x8,   _,   _, 0x2): return $"R[{x}] &= R[{y}]";              
    /* 8XY3 */ case (0x8,   _,   _, 0x3): return $"R[{x}] ^= R[{y}]";              
    /* 8XY4 */ case (0x8,   _,   _, 0x4): return $"R[{x}] += R[{y}]";              
    /* 8XY5 */ case (0x8,   _,   _, 0x5): return $"R[{x}] -= R[{y}]";              
    /* 8XY6 */ case (0x8,   _,   _, 0x6): return $"R[{x}] SHR 1";                  
    /* 8XY7 */ case (0x8,   _,   _, 0x7): return $"R[{y}] -= R[{x}]";              
    /* 8XYE */ case (0x8,   _,   _, 0xe): return $"R[{x}] SHL 1";                  
    /* 9XY0 */ case (0x9,   _,   _, 0x0): return $"if R[{x}] == R[{y}] then";      
    /* ANNN */ case (0xa,   _,   _,   _): return $"I = {nnn}";                     
    /* BNNN */ case (0xb,   _,   _,   _): return $"jump {nnn} + R[0]";             
    /* CXKK */ case (0xc,   _,   _,   _): return $"V[{x}] = rand AND {nn}";        
    /* DXYN */ case (0xd,   _,   _,   _): return $"sprite R{x}, R{y}, {n}";        
    /* EX9E */ case (0xe,   _, 0x9, 0xe): return $"if keyup R[{x}] then";          
    /* EXA1 */ case (0xe,   _, 0xa, 0x1): return $"if keydown R[{x}] then";        
    /* FX07 */ case (0xf,   _, 0x9, 0x7): return $"R[{x}] = delay_timer";          
    /* FX0A */ case (0xf,   _, 0x0, 0xa): return $"readkey in R[{x}]";             
    /* FX15 */ case (0xf,   _, 0x1, 0x5): return $"delaytimer = R[{x}]";           
    /* FX18 */ case (0xf,   _, 0x1, 0x8): return $"soundtimer = R[{x}]";           
    /* FX1E */ case (0xf,   _, 0x1, 0xe): return $"I += R[{x}]";                   
    /* FX29 */ case (0xf,   _, 0x2, 0x9): return $"I = font_sprite[R[{x}]]";       
    /* FX33 */ case (0xf,   _, 0x3, 0x3): return $"bcd R[{x}]";                    
    /* FX55 */ case (0xf,   _, 0x5, 0x5): return $"saveflags {x}";                 
    /* FX65 */ case (0xf,   _, 0x6, 0x5): return $"loadflags {x}";                 
}
{{< /highlight >}}

Using this to disassemble the test rom, we will get the excerpt below. I am not displaying all of the code for breveity.

{{< highlight csharp >}}
...
   613	D8B4	sprite R8, R11, 4
   614	A236	I = 566
   615	D9B4	sprite R9, R11, 4
   616	A202	I = 514
   617	DAB4	sprite R10, R11, 4
   618	6B06	R[11] = 6
   619	A22A	I = 554
   620	D8B4	sprite R8, R11, 4
   621	A20A	I = 522
   622	D9B4	sprite R9, R11, 4
   623	A206	I = 518
   624	8750	R[7] = R[5]
   625	472A	if R[7] == 42 then
   626	A202	I = 514
   627	DAB4	sprite R10, R11, 4
   628	6B0B	R[11] = 11
   629	A22A	I = 554
   630	D8B4	sprite R8, R11, 4
   631	A20E	I = 526
   632	D9B4	sprite R9, R11, 4
   633	A206	I = 518
   634	672A	R[7] = 42
   635	87B1	R[7] |= R[11]
   636	472B	if R[7] == 43 then
   637	A202	I = 514
   638	DAB4	sprite R10, R11, 4
   639	6B10	R[11] = 16
   640	A22A	I = 554
   641	D8B4	sprite R8, R11, 4
   642	A212	I = 530
   643	D9B4	sprite R9, R11, 4
   644	A206	I = 518
   645	6678	R[6] = 120
   646	671F	R[7] = 31
   647	8762	R[7] &= R[6]
   648	4718	if R[7] == 24 then
   649	A202	I = 514
   650	DAB4	sprite R10, R11, 4
   651	6B15	R[11] = 21
   652	A22A	I = 554
   653	D8B4	sprite R8, R11, 4
   654	A216	I = 534
   655	D9B4	sprite R9, R11, 4
   656	A206	I = 518
...
{{< /highlight >}}

## Sierpinski

## Octo

### What is Octo?

### An example program

### Compiling a rom
