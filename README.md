
# Intro
So for many years people have been hooking Direct X by overlays provided by 3rd party companys to avoid hooking the native Direct X. In this ill show u can obtain these dynamically rather then staticly revsering DiscordHook64.dll or overlay64.dll for their functions that may change update to update. 

(Note: this will focus on DX12)

# Setup
In this tutorial, I will be using a modified D3D12-Hook to fit my coding style as a base for my DX12 hook. For my hooking library, I rather differ from minhook to safetyhook as it gives me more control over the stack and registers

D3D12-Hook
- https://github.com/DrNseven/D3D12-Hook-ImGui
Hooking library
- https://github.com/cursey/safetyhook

# The Initial Idea for getting Present
Rather than copying and pasting code ill show u the thought process.

## Understanding how overlays work

- DiscordHook64.dll - discords overlay
- overlay64.dll - Ubisoft's overlay

These are both overlays that display information over the game process. In order to do this, they must interact with the game. SO HOW? they load [dynamic link library's](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-libraries). This gives them the context to run in the game's memory and interact with its rendering system in order to render a fast overlay optimally.

Now in order for them to intercept the render queue, they hook the [Present function in dx12](https://learn.microsoft.com/en-us/windows/win32/api/dxgi/nf-dxgi-idxgiswapchain-present)
. This is achieved through using [microsoft detours libary](https://github.com/microsoft/Detours) that internally uses inline hooks (aka trampoline hooks) meaning 

Example:
Let’s say the target function starts like this:
```c++
0x1000: mov eax, ecx
0x1002: add eax, 1
```
Detours will overwrite those bytes with a jump:
```c++
0x1000: jmp 0xDEADBEEF   ; jumps to your hook
0x1002: add eax, 1
```

To understand or reverse this, we need to decode the jump instruction. A typical jmp used by Detours looks like:
```c++
E9 <rel32>
```

- E9 is the opcode for a relative jump.
- <rel32> is a signed 4-byte offset from the next instruction.
- Total size = 5 bytes.

With this knowledge, we can follow the jmp in C++
```c++
    // Skips the first byte (E9) to read the next 4 bytes.
    int32_t relOffset = *reinterpret_cast<int32_t*>(instructionAddress + 1);

    // calculate the absolute target address
    uintptr_t nextInstruction = reinterpret_cast<uintptr_t>(instructionAddress + 5);
    uintptr_t targetAddress = nextInstruction + relOffset;
```

This works because the jump offset is relative to the instruction after the jmp. By adding the offset to the address immediately following the instruction, you get the actual destination of the jump. This is the same way the cpu would do it.

# Putting the idea into action
Now, by using the Kiero library, we can dynamically get the offset for DX12's Present already and log it out

```c++
//add this to d3d12hook.h
inline void* getMethodByIndex(uint16_t Index) {
	void* target = (void*)MethodsTable[Index];
	return target;
}

//[140] Present
void* _140 = getMethodByIndex(140);
printf("[140] Present: %p \n", _140);


//[54] Present
void* _54 = getMethodByIndex(54);
printf("[54] ExecuteCommandLists: %p \n", _54);
```

We should get something like this when we run the code
```c++
[-] _140: 00007FFFB2472A20
[-] _54: 00007FFEA08B6AC0
```
Open the address of _140 in Cheat Engine and right-click "disassemble this memory region" We should see the E9 as expected

We can see that Cheat Engine helpfully shows us what module this leads to being overlay64.dll. Now right-click it and press follow 2 times, and u end up in the overlay64.dll module. This is the hook Ubisoft uses for an overlay.

The first jump u may have noticed, u was taken to a padding region. These use a different calling convention with the op code FF 25

That instruction format:

```c++
FF 25 xx xx xx xx   ; jmp qword ptr [rip+rel32]
```
- The 4-byte value after is a relative offset from the end of the instruction
- You must compute the target address by reading the memory address stored at [rip + offset]

```c++
uintptr_t resolveIndirectJmp(uintptr_t address) {
    uint32_t ripOffset = *reinterpret_cast<uint32_t*>(address + 2);
    uintptr_t rip = address + 6; // instruction is 6 bytes long
    uintptr_t targetPtr = rip + ripOffset;
    return *reinterpret_cast<uintptr_t*>(targetPtr); // dereference the pointer
}
```

Now we can make the functions to loop through all the jumps and get a valid one by checking that we don't go into a padding region.

```c++
uintptr_t resolveJump(uintptr_t address) {
    const int instructionSize = 5;

    int32_t relOffset = *reinterpret_cast<int32_t*>(address + 1);
    uintptr_t nextInstruction = address + instructionSize;
    return nextInstruction + relOffset;
}

bool isIndirectJmp(uintptr_t address) {
    // check for FF 25
    uint8_t* bytes = reinterpret_cast<uint8_t*>(address);
    return bytes[0] == 0xFF && bytes[1] == 0x25;
}

uintptr_t resolveIndirectJmp(uintptr_t address) {
    uint32_t ripOffset = *reinterpret_cast<uint32_t*>(address + 2);
    uintptr_t rip = address + 6; // instruction is 6 bytes long
    uintptr_t targetPtr = rip + ripOffset;
    return *reinterpret_cast<uintptr_t*>(targetPtr); // dereference the pointer
}

uintptr_t resolveJumpIfValid(uintptr_t address, int loop = 99) {
    uintptr_t currentAddress = address;
    uintptr_t prevValidTarget = address;

    while (loop-- > 0) {
        uint8_t* bytes = reinterpret_cast<uint8_t*>(currentAddress);
        uint8_t opcode = bytes[0];
        logger::print("[~] Current opcode: 0x%02X at 0x%p\n", opcode, (void*)currentAddress);

        uintptr_t targetAddress = 0x0;
        if (opcode == 0xE9) {
            targetAddress = resolveJump(currentAddress);
        }
        else if (bytes[0] == 0xFF && bytes[1] == 0x25) {
            targetAddress = resolveIndirectJmp(currentAddress);
        }
        else {
            break; // not a known jump
        }

        uintptr_t targetAddress = resolveJump(currentAddress);

        uint8_t targetOpcode = *reinterpret_cast<uint8_t*>(targetAddress);
        logger::print("[~] Target opcode: 0x%02X at 0x%p\n", targetOpcode, (void*)targetAddress);

        if (targetOpcode != 0x00)
            prevValidTarget = targetAddress;

        if (targetAddress == currentAddress) {
            logger::print("[~] JMP loops to itself at 0x%p\n", (void*)currentAddress);
            break;
        }

        logger::print("[~] Following JMP: 0x%p -> 0x%p\n", (void*)currentAddress, (void*)targetAddress);
        currentAddress = targetAddress;
    }

    logger::print("[~] Final resolved JMP: 0x%p\n", (void*)prevValidTarget);
    return prevValidTarget;
}
```

# Result

When combined with a basic hook on ExecuteCommandLists for now 

![cheat engine](https://github.com/TheRealJoelmatic/dx12-discord-hook/blob/main/img/Screenshot%202025-08-06%20013554.png?raw=true)

Boom, we have hooked Ubisoft's overlay with 0 sig or offsets that could change, and this will work with any other overlay. (Some overlays will be signed by anticheats, so look in IDA Pro for functions that the found overlay calls with a vaild SwapChain)

Now you're wondering whether there is a better way to get the CommandQueue from the ExecuteCommandLists without such an easily detectable hook on ExecuteCommandLists?
- yes

One of the best ways is to simply find the current address for ID3D12CommandQueue* queue and scan for all locations referencing it. In the case of r6, there are spots in the .data section that can be sig scanned for and read to achieve this.
