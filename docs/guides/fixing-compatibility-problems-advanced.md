# Fixing Missing AOBs (Advanced & In-Depth)

When UE4SS fails to properly launch due to missing AOBs (Array of Bytes signatures), you can provide custom AOBs and callback functions using Lua. Doing so, however, requires a level of reverse engineering knowledge and tooling setup that may feel complex at first. This guide expands upon the original instructions, providing even more detail, context, and recommended best practices.

## Prerequisites

- **Knowledge of Basic Reverse Engineering Concepts:**  
  You should have a general idea of what a signature (AOB) is, how to use a debugger, and how to navigate memory in x64dbg.
  
- **Understanding ‘root directory’ and ‘working directory’:**  
  - **Root directory:** The directory containing `ue4ss.dll`.
  - **Working directory:** This can be either the root directory **or** a game-specific directory such as `<root directory>/SatisfactoryEarlyAccess`.

- **Familiarity with UE4SS Setup & Directories:**  
  Make sure you know where `UE4SS_Signatures` folder should be created (it should be next to `ue4ss.dll` or in a game-specific working directory as described above).

- **Preparation and Tools Installed:**
  - **Epic Games Launcher & Unreal Engine:** For creating a “blank shipped game” environment with the correct engine version.
  - **x64dbg:** A debugger tool for Windows (https://x64dbg.com/).
  - **(Optional) Baymax Tools:** A plugin to help generate signatures easily.
  - **(Optional) Swiss Army Knife (by Nukem9):** For extracting signatures.

## Why Create Your Own AOBs?

When UE4SS updates or when you attempt to mod a game with a newer or unusual engine build, official AOB signatures inside UE4SS may fail. In these cases, providing your own AOBs and callbacks allows UE4SS to locate critical engine functions and global variables in memory and resume normal operation.

## High-Level Overview

1. **Identify Which Signatures Are Missing:** Determine which functions or variables UE4SS cannot find (e.g., GUObjectArray, GMalloc, FName or FText constructors).
2. **Set Up a Reference Environment:** Create a blank Unreal Engine shipped game with debug files (PDBs) that matches your game’s Unreal Engine version. This environment helps you identify function signatures cleanly.
3. **Reverse Engineer and Extract AOBs:** Using x64dbg (and optional plugins), open the shipped game and locate the desired function in memory. Copy out the unique bytes that form a reliable signature.
4. **Apply Your Signatures to the Actual Game:** Attach x64dbg to the target game, find the matching bytes, and confirm that the signature you extracted matches code in the game you want to mod.
5. **Create a Lua Script:** Write a Lua file in `UE4SS_Signatures` to tell UE4SS what AOB pattern to search for (through `Register`) and what final address to return (through `OnMatchFound`).

## Finding AOBs: A More Detailed Explanation

> [!CAUTION]
> Reverse engineering these signatures isn’t trivial. You may need to step outside the scope of this guide, read reverse-engineering tutorials, or ask for community support. The steps below are a starting point, not a complete primer on reverse engineering.

### Step 1: Determine Your Game’s Unreal Engine Version

UE4SS tries to detect the engine version automatically. If you need to verify:

- Right-click on the game’s `.exe` file (often in `Binaries` folder).
- Select **Properties** -> **Details** tab.
- Look for the “File Version” or “Product Version” field, which often correlates to the Unreal Engine version.

For example: If it says `5.3.2.0`, it likely corresponds to UE 5.3.2.

### Step 2: Installing the Matching Unreal Engine Version

- Create an [Epic Games](https://www.epicgames.com/) account and install the **Epic Games Launcher**.
- In the launcher, go to **Unreal Engine** -> **Library** tab and install the engine version matching your game’s version (e.g., UE 5.3.2).

### Step 3: Creating a Blank Shipped Game with PDBs

1. Launch the installed Unreal Engine version.  
2. In the New Project window, select the **Games** tab -> **Blank** template.  
   - Uncheck “Starter Content” if you prefer a minimal project.
   - Name your project and specify a directory.
3. Once created, open **Platforms** -> **Packaging Settings**, and enable “Include Debug Files in Shipping Builds”.  
4. From **Platforms** -> **Windows**, select “Shipping” configuration (or whichever build matches your target game’s build type).  
5. **Package Project** -> Choose a folder.  
6. After packaging completes, verify that the output folder’s `Binaries` directory contains both a `.exe` and a `.pdb`. The `.pdb` file provides symbolic information for reverse engineering.

### Step 4: Using x64dbg to Analyze the Blank Project

1. Install x64dbg from [https://x64dbg.com/](https://x64dbg.com/).
2. Run the `.exe` of your newly packaged blank project from its root directory.
3. Open x64dbg.  
   - Go to **File** -> **Attach** -> Select the blank project `.exe`. 
   - Ensure you’re attaching to the shipped `.exe` located in `Binaries` or root (whichever works).

### Step 5: Identifying the Function of Interest

You need to know which function or variable you’re trying to match in your target game. For example, if UE4SS fails on `GMalloc`, you must find `FMemory::Free` as a reference to locate `GMalloc`.

**Optional Steps:**
- Connect your Epic Games account with GitHub to access the Unreal Engine source code.
  - **Epic Games Website** -> **Manage Account** -> **Apps and Accounts** -> **GitHub**
  - Accept invitation via email.
- Browse the Unreal Engine source for the function you need. For `FMemory::Free` in UE5.3.2, you might look at:  
  [FMemory::Free in UE source](https://github.com/EpicGames/UnrealEngine/blob/5.3.2-release/Engine/Source/Runtime/Core/Public/HAL/FMemory.inl#L142)

### Step 6: Locating the Function in x64dbg

1. In x64dbg, switch to the **Symbols** tab.
2. In the left pane, select the `.exe`.
3. In the right pane, search for the function name (e.g., `FMemory::Free`).
4. Double-click the found function to navigate back to the CPU view, positioning the instruction pointer at the start of the function in memory.

### Step 7: Extracting a Signature

Once you’ve identified the start of the function, you need to copy a unique sequence of bytes:

1. Consider installing [Baymax Tools](https://github.com/sicaril/BaymaxTools) for x64dbg to ease signature extraction.
2. Highlight a set of instructions at the start of the function. 
   - Right-click -> **Copy** -> **Selection (Bytes only)** to get a raw byte sequence.
   - With Baymax Tools: Right-click -> **Baymax Tools** -> **Copy Signature** for a ready-made signature pattern.
3. Save these bytes or patterns for later comparison. You may want to store them in a file to easily refer back.

### Understanding Terminology

- **Signature:** A carefully chosen sequence of bytes that uniquely identifies a function or code snippet.
- **Block of Bytes:** A simple, possibly unstructured, segment of raw data without inherent uniqueness.
- **RIP (Instruction Pointer):** The CPU register that holds the address of the next instruction to execute.

### Step 8: Searching in the Actual Target Game

Now that you have a reference signature, you need to find it in your target game:

1. Launch the target game `.exe`.
2. Attach x64dbg as before: **File** -> **Attach** -> Select the game’s `.exe`.
3. In x64dbg, search memory for the signature you extracted from the blank project.
   - If direct search fails, try partial sequences of bytes.
   - Try patterns generated by Baymax Tools.
   - Compare and contrast instructions between the blank project and the actual game to locate a similar code region.

If you find a match, you’ve identified the address that corresponds to the target function or variable in the actual game. If you can’t find it, you may need to refine your signature, pick a different part of the function, or ask for community help (UE4SS Discord or GitHub Issues).

### Step 9: Applying the Signature in UE4SS

Once you have a working AOB:

1. Create the `UE4SS_Signatures` directory in your working directory (if it doesn’t already exist).
2. Make a `.lua` file corresponding to the missing AOB (e.g., `GMalloc.lua` if you’re fixing GMalloc).
3. Inside this `.lua` file, define the `Register` and `OnMatchFound` functions.

**Register function:** Returns your AOB signature as a string (spaces optional), e.g.:

```lua
function Register()
    return "48 8B D9 48 8B 0D ?? ?? ?? ?? 48 85 C9 75 0C E8 ?? ?? ?? ??"
end
```

**OnMatchFound function:** Receives the match address and must return the exact memory address of the target function or variable. Use `DerefToInt32` if needed to resolve relative addresses.

```lua
function OnMatchFound(MatchAddress)
    local MovInstr = MatchAddress + 0x03
    local Offset = DerefToInt32(MovInstr + 0x3)
    local RIP = MovInstr + 0x7
    local GMallocAddress = RIP + Offset
    return GMallocAddress
end
```

### Verifying Your Work

- Run the game with UE4SS again. If successful, UE4SS now uses your custom script to find the previously missing address.
- If it fails silently, confirm:
  - That the Lua script is in the correct directory.
  - That your AOB is correct and unique.
  - That `OnMatchFound` returns the correct final address.
  
If still stuck, consider posting detailed steps, logs, and code snippets to the UE4SS community channels. The more detail you provide, the more likely someone can guide you to a solution.

## What ‘OnMatchFound’ Should Return

Recap of the required returns for each known signature type:

- **GUObjectArray:** Return the exact address of the `GUObjectArray` global variable.
- **FName_ToString:** Return the start address of the `FName::ToString` function.
- **FName_Constructor:** Return the start address of `FName::FName`. Multiple versions might exist (e.g., for `char*` and `wchar_t*`), but UE4SS validates the correct one internally.
- **FText_Constructor:** Return the start address of `FText::FText`.
- **StaticConstructObject:** Return the start address of the `StaticConstructObject_Internal` global function.
- **GMalloc:** Return the address of the global `GMalloc` variable. Typically found by scanning around `FMemory::Free` and resolving a MOV instruction.

## Example Scripts

### Direct Scan Example

```lua
function Register()
    return "48 8B C4 57 48 83 EC 70 80 3D ?? ?? ?? ?? ?? 48 89"
end

function OnMatchFound(MatchAddress)
    return MatchAddress
end
```

### Indirect Scan Example

```lua
function Register()
    return "41 B8 01 00 00 00 48 8D 15 ?? ?? ?? ?? 48 8D 0D ?? ?? ?? ?? E9"
end

function OnMatchFound(MatchAddress)
    local InstrSize = 0x05
    local JmpInstr = MatchAddress + 0x14
    local Offset = DerefToInt32(JmpInstr + 0x1)
    local Destination = JmpInstr + Offset + InstrSize
    return Destination
end
```

## Tips, Tricks, and Troubleshooting

- **Patience & Iteration:** Extracting and refining AOBs can be trial-and-error. If a signature doesn’t work, try a different sequence of bytes or look elsewhere in the function.
- **Partial Signatures:** If the full function signature isn’t found, try unique parts of it.
- **Community Help:** If stuck, show your steps, scripts, and logs on the UE4SS Discord or GitHub Issues.
- **Check Offsets Carefully:** Off-by-one or incorrect indexing is a common issue. Double-check your calculations.
- **Manual Verification:** Sometimes running the blank project again in x64dbg and comparing with the target game’s memory can highlight discrepancies.

By following these expanded steps and leveraging the provided tools, you’ll have a more comprehensive understanding of how to fix missing AOBs with UE4SS. Although still complex, this extended guide should help clarify the process and offer practical insights for both beginners and experienced modders venturing into reverse engineering territory.