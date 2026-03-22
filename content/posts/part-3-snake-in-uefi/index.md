+++
title = "(Part 3) Snake in... UEFI?"
description = "Turning our idea into reality: debugging, graphics, and finally playing our UEFI Snake game!"
date = "2026-03-22T01:58:58.000+01:00"
toc = true
tags = ["edk2", "uefi", "firmware"]
+++

![Badge](https://hitscounter.dev/api/hit?url=https%3A%2F%2Fhexaliker.fr%2Fposts%2Fpart-3-snake-in-uefi%2F&label=views+%28day+%2F+all+time%29&icon=eye-fill&color=%2362a0ea&message=&style=for-the-badge&tz=UTC)

With our high-level UEFI component overview, and our mandatory Hello World application written, developing SnakePkg will be a piece of cake!

## Recap from Part 2

In part 2, we covered how `.inf` and `.dsc` files are structured, and their role in the EDK II build system. We also wrote the mandatory UEFI Hello World application (as any programmer would've done)

## Our model

To make any type of maintainable project, no matter its size, one should always think about its model; I define the word "model" in this context as:

- The project's main goal
- Its dependencies
- Potential algorithms/techniques to implement

You might think that its awkward to introduce it now, however I'd like to think it makes sense to introduce it now: that way, we can first learn the basics as done in **parts 1 and 2**, to then have the most optimal thinking model, therefore aiming to make optimal decisions regarding the project's architecture.

Now is the right time to define our model:

### Visual and gameplay-wise:

- We must have a grid the snake can move in:
    - One that's reasonably sized so that the game feels enjoyable to play
- We must have a green-colored snake (duh) which will move in the aforementioned grid
- We can display an image in the background to "enlive" to the application

### Internals-wise

- We must make middleware(s) that handles such errors cleanly
    - UEFI APIs tend to depend a lot on returning `EFI_STATUS` values
    - Writing several `if (EFI_ERROR(Status))` error-handling clauses will make the code feel clunky and messy
- We must avoid using the heap whenever possible (especially given our environment)
- We must easily debug our code (_hint hint EDK2Code and LLDB_)

With our model in mind, we're ready to start!

## Debugging our app

In the previous two parts, I've teased setting up LLDB as well as using EDK2Code to set up clangd and to give us a bunch of QoL features.

Now that we're actually developing something serious, it's time to reveal how those are installed!

### EDK2Code

We're starting with the easiest and most important one.

According to Intel, [EDK2Code](https://github.com/intel/Edk2Code/) is an extension "designed to improve the development experience for engineers working with the EDK2 (UEFI) codebase"

Indeed, the extension is exactly as advertised, the steps to set it up are trivial:

1. Download the latest VSIX file from [the releases tab](https://github.com/intel/Edk2Code/releases/)
2. Install it in any VSCode-based IDE (such as the awesome VSCodium)
3. **Build your project**
4. Hit `Ctrl + Shift + P` and type `EDK2: Rebuild index database`
5. Select the EDK II `Build` folder, and in the next menu, your project

This automatically sets up clangd, and a bunch of other useful things helpful for development (like "Go-tos" in DSC or INF files, and syntax highlighting in the latter).

### LLDB

Now this is where things get technical; I couldn't find much resources about setting the former for **UEFI development** (It's a niche domain so it makes sense).

Therefore, with the help of the [LLDB Python API](https://lldb.llvm.org/python_api.html), I managed to set up the following Python script:


<details>
  <summary>Click to expand</summary>

```py
#!/usr/bin/env python3

# You won't believe how much time this took me

import lldb
import threading
import sys
import re
import os
import time

def parse_image_base_from_log(log_file):
    if not log_file:
        return None

    with open(log_file, 'r', errors='ignore') as f:
        content = f.read()
        pattern = re.compile(r'LoadedImage\-\>ImageBase[:\s]+0x([0-9A-Fa-f]+)+\n', re.IGNORECASE)
        match = pattern.search(content)
        if match:
            return f"0x{match.group(1)}"

    return None

def load_symbols_at_base(debugger, image_base):
    target = debugger.GetSelectedTarget()

    if not target.IsValid():
        print("No valid target")
        return False

    base_addr = int(image_base, 16)
    error = lldb.SBError()
    module = target.AddModule("Snake.debug", None, None)

    if not module.IsValid():
        print(f"Failed to add module")
        return False

    print(f"Loading symbols at ImageBase {image_base}")

    for section in module.section_iter():
        section_name = section.GetName()
        section_addr = section.GetFileAddress()

        if section_addr != lldb.LLDB_INVALID_ADDRESS:
            load_addr = base_addr + section_addr
            target.SetSectionLoadAddress(section, load_addr)

    return True

def wait_and_load_symbols(debugger, log_file, timeout):
    print(f"Waiting for ImageBase in {log_file}")

    start_time = time.time()

    while time.time() - start_time < timeout:
        image_base = parse_image_base_from_log(log_file)
        if image_base:
            print(f"Detected ImageBase: {image_base}")
            return load_symbols_at_base(debugger, image_base)

    print("Missed ImageHandle log")
    return False

def auto_load_symbols(debugger, command, result, internal_dict):
    args = command.split()
    log_file = args[0] if len(args) > 0 else None
    timeout = int(args[1]) if len(args) > 1 else 10

    if not log_file:
        print("Cannot find debug.log")
        print("Usage: auto_load_symbols <log_file> [timeout]")
        return

    print("Running watcher in separate thread...")
    threading.Thread(target=wait_and_load_symbols, args=(debugger, log_file, timeout)).start()

def __lldb_init_module(debugger, internal_dict):
    debugger.HandleCommand(
        'command script add -f lldb_uefi_helper.auto_load_symbols auto_load_symbols'
    )
```
</details>

This script essentially registers a command `auto_load_symbols` that takes in 2 arguments:

- `<log_file>`: the output of the VM piped into a file (where we'll grab the application's image base from)
- `[timeout]`: self-explainatory 

The script then periodically performs a regex search with the pattern `LoadedImage\-\>ImageBase[:\s]+0x([0-9A-Fa-f]+)+\n` to grab the image base log our app will output after it gets loaded.

Funnily enough, this is the biggest roadblock when debugging a UEFI application; the debugger simply can't guess the image base!

To automate this even further, we'll make a `.vscode` folder with two files:

`.vscode/launch.json`

<details>
  <summary>Click to expand</summary>

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "64-bit UEFI Debug App",
            "type": "lldb-dap",
            "request": "attach",
            "program": "${workspaceFolder}/../Build/SnakePkg/DEBUG_GCC5/X64/SnakePkg/Snake/Snake/DEBUG/Snake.dll",
            "preLaunchTask": "Launch QEMU with Debug",
            "postDebugTask": "Kill QEMU",
            "attachCommands": [
                "gdb-remote localhost:1234"
            ],
            "preRunCommands": [
                "command script import \"${workspaceFolder}/lldb_uefi_helper.py\"",
                "continue",
            ],
            "postRunCommands": [
                "auto_load_symbols ${workspaceFolder}/../Build/SnakePkg/DEBUG_GCC5/X64/debug.log 10"
            ]
        }
    ]
}
```
</details>


and `.vscode/tasks.json`

<details>
  <summary>Click to expand</summary>

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Launch QEMU with Debug",
            "type": "shell",
            "command": "${workspaceFolder}/Scripts/run.sh",
            "args": ["X64", "DEBUG"],
            "isBackground": true,
            "options": {
                "cwd": "${workspaceFolder}/Scripts"
            },
            "problemMatcher": {
                "pattern": {
                    "regexp": "^$"
                },
                "background": {
                    "activeOnStart": true,
                    "beginsPattern": ".",
                    "endsPattern": "Starting QEMU"
                }
            }
        },
        {
            "label": "Kill QEMU",
            "type": "shell",
            "command": "pkill",
            "args": ["-9", "qemu-system"],
            "problemMatcher": [],
            "presentation": {
                "echo": true,
                "reveal": "silent",
                "focus": false,
                "panel": "shared",
                "showReuseMessage": false,
                "clear": false
            }
        },
    ]
}
```
</details>

In essence, when we press the fancy debug button in the VS Code debugging pane, it will automatically build the SnakePkg package, and launch QEMU, and automatically attach LLDB to the latter, loading the symbols using `auto_load_symbols`, as soon as our app outputs its base address in `ConOut`, with a timeout of 10 seconds.

Here's how our app would print its base address:

```c
EFI_STATUS                    Status;
EFI_LOADED_IMAGE_PROTOCOL     *LoadedImage;

Status = gBS->OpenProtocol(
  ImageHandle,
  &gEfiLoadedImageProtocolGuid,
  (VOID **)&LoadedImage,
  ImageHandle,
  NULL,
  EFI_OPEN_PROTOCOL_GET_PROTOCOL
);
ASSERT_EFI_ERROR(Status);

Print(L"LoadedImage->ImageBase: 0x%p\n", LoadedImage->ImageBase);

Status = gBS->CloseProtocol(
  ImageHandle,
  &gEfiLoadedImageProtocolGuid,
  ImageHandle,
  NULL
);
ASSERT_EFI_ERROR(Status);

```

We "open" a protocol with `EFI_BOOT_SERVICES->OpenProtocol` for our own application, which is **DIFFERENT** than "locating" a protocol using `EFI_BOOT_SERVICES->LocateProtocol`, as the latter will use another component's version of that protocol, which is NOT linked to our `EFI_HANDLE`.

![A screenshot of a the UEFI 2.11 specification regarding `OpenProtocol` and `LocateProtocol`](./uefi_spec_gbs_protocols.png)

_Source: https://uefi.org/specs/UEFI/2.11/07_Services_Boot_Services.html#protocol-handler-services_

## Graphics in all of this

With that out of the way, what really remains is making Snake, which is a game. A game requires graphics, obviously. The UEFI specification allows us to use the [Graphics Output Protocol](https://uefi.org/specs/UEFI/2.11/12_Protocols_Console_Support.html#efi-graphics-output-protocol):

```c
typedef struct EFI_GRAPHICS_OUTPUT_PROTCOL {
 EFI_GRAPHICS_OUTPUT_PROTOCOL_QUERY_MODE     QueryMode;
 EFI_GRAPHICS_OUTPUT_PROTOCOL_SET_MODE       SetMode;
 EFI_GRAPHICS_OUTPUT_PROTOCOL_BLT            Blt;
 EFI_GRAPHICS_OUTPUT_PROTOCOL_MODE           *Mode;
} EFI_GRAPHICS_OUTPUT_PROTOCOL;
```

We're interested in `EFI_GRAPHICS_OUTPUT_PROTCOL->Blt`, which allows efficient block transfering to the framebuffer (`EFI_GRAPHICS_OUTPUT_PROTOCOL_MODE->FrameBufferBase`), and the pointer `EFI_GRAPHICS_OUTPUT_PROTCOL->Mode` which allows us to get information about the current mode GOP is in.

We can access the latter with `mInfo = mGop->Mode->Info`, giving us these properties:

```c
typedef struct {
 UINT32                    Version;
 UINT32                    HorizontalResolution;
 UINT32                    VerticalResolution;
 EFI_GRAPHICS_PIXEL_FORMAT PixelFormat;
 EFI_PIXEL_BITMASK         PixelInformation;
 UINT32                    PixelsPerScanLine;
} EFI_GRAPHICS_OUTPUT_MODE_INFORMATION;
```

We only care about `HorizontalResolution` and `VerticalResolution` for properly scaling the grid to screens of various sizes.

We can write the following:

In `Graphics.h`

```c
#ifndef __GRAPHICS_H__
#define __GRAPHICS_H__

...

extern EFI_GRAPHICS_OUTPUT_PROTOCOL           *gGop;
extern EFI_GRAPHICS_OUTPUT_MODE_INFORMATION   *gGopInfo;
extern EFI_HII_IMAGE_PROTOCOL                 *gHiiImage;
extern EFI_HII_HANDLE                         gHiiHandle;
extern UINTN                                  gMiddleScreenX;
extern UINTN                                  gMiddleScreenY;

...


BOOLEAN
InitGfx(
  VOID
);
```

and in `Graphics.c`:

```c
EFI_GRAPHICS_OUTPUT_PROTOCOL            *gGop;
EFI_GRAPHICS_OUTPUT_MODE_INFORMATION    *gGopInfo;
STATIC EFI_GRAPHICS_OUTPUT_BLT_PIXEL    *mBackBuffer;
EFI_HII_IMAGE_PROTOCOL                  *gHiiImage;
EFI_HII_HANDLE                          gHiiHandle;
STATIC EFI_HII_DATABASE_PROTOCOL        *mHiiDatabase;
STATIC UINTN                            mBackBufferLen;
UINTN                                   gMiddleScreenX;
UINTN                                   gMiddleScreenY;

BOOLEAN
InitGfx(
  VOID
)
{
  EFI_STATUS Status;
  EFI_HII_PACKAGE_LIST_HEADER   *PackageListHeader;

  Status = gBS->OpenProtocol (
    gImageHandle,
    &gEfiHiiPackageListProtocolGuid,
    (VOID **)&PackageListHeader,
    gImageHandle,
    NULL,
    EFI_OPEN_PROTOCOL_GET_PROTOCOL
  );
  ASSERT_EFI_ERROR(Status);

  Status = gBS->LocateProtocol(
    &gEfiHiiImageProtocolGuid,
    NULL,
    (VOID **)&gHiiImage
  );
  ASSERT_EFI_ERROR(Status);

  Status = gBS->LocateProtocol(
    &gEfiHiiDatabaseProtocolGuid,
    NULL,
    (VOID **)&mHiiDatabase
  );
  ASSERT_EFI_ERROR(Status);

  Status = gBS->LocateProtocol(
    &gEfiGraphicsOutputProtocolGuid,
    NULL,
    (VOID **)&gGop
  );
  ASSERT_EFI_ERROR(Status);

  Status = mHiiDatabase->NewPackageList(
    mHiiDatabase,
    PackageListHeader,
    NULL,
    &gHiiHandle
  );
  ASSERT_EFI_ERROR(Status);

  gGopInfo = gGop->Mode->Info;
  mBackBufferLen = gGopInfo->HorizontalResolution * gGopInfo->VerticalResolution * sizeof(EFI_GRAPHICS_OUTPUT_BLT_PIXEL);
  mBackBuffer = AllocateZeroPool(mBackBufferLen);
  ASSERT(mBackBuffer);

  gMiddleScreenX = (gGopInfo->HorizontalResolution / 2) - 1;
  gMiddleScreenY = (gGopInfo->VerticalResolution / 2) - 1;

  return TRUE;
}
```

Essentially, we allocate a backbuffer which we will use to avoid screen tearing (GOP doesn't really help much), and we calculate the middle coordinates of the screen `gMiddleScreenX` and `gMiddleScreenY`. Simple and straightforward!

Now that the graphics are set up, it's time to bring the Snake game to life. The core mechanics live in [`GameLogic.c`](https://github.com/AstonishedLiker/SnakePkg/blob/main/Snake/GameLogic.c):

- The game grid is just a simple 1D array of type [`GRID_MATRIX`](https://github.com/AstonishedLiker/SnakePkg/blob/main/Snake/GameLogic.h#L18), where each [`GRID_CELL`](https://github.com/AstonishedLiker/SnakePkg/blob/main/Snake/GameLogic.h#L17) can be either empty (cell is `0`), filled with a snake segment [(cell is `1`)](https://github.com/AstonishedLiker/SnakePkg/blob/main/Snake/GameLogic.h#L14), or hold an apple [(cell is `2`)](https://github.com/AstonishedLiker/SnakePkg/blob/main/Snake/GameLogic.h#L15).

- The snake itself is stored as a list of cell indices, which makes movement and collision checks straightforward. Apple placement is handled by a fixed-seed (`13371337`) XorShift32 PRNG, so they spawn in the same spots every time, deterministically random, if you will. While generating a random seed is possible, I just didn't bother in this case.

- Input is captured by [`QueryLastKeystroke`](https://github.com/AstonishedLiker/SnakePkg/blob/main/Snake/GameLogic.c#L32), which reads arrow key presses from the UEFI console using a double-read loop to make sure the moves register instantly (and not 1 cycle late!!).

- For visuals, [`DrawGrid`](https://github.com/AstonishedLiker/SnakePkg/blob/main/Snake/GameLogic.c#L98) handles rendering the grid, snake, and apple, using the display's resolution provided by GOP as previously mentioned (with the constant [`CELL_PERCENTAGE_SCREEN_OCCUPANCY`](https://github.com/AstonishedLiker/SnakePkg/blob/main/Snake/GameLogic.h#L5) set to 3%), while [`DrawScene`](https://github.com/AstonishedLiker/SnakePkg/blob/main/Snake/GameLogic.c#L177) layers on the [TianoCore bitmap that appears at boot on OVMF builds](https://raw.githubusercontent.com/AstonishedLiker/SnakePkg/refs/heads/main/Snake/Assets/Logo.bmp), as an easter egg that references EDK-II, what we're using to make SnakePkg!

- [`UpdateSnake`](https://github.com/AstonishedLiker/SnakePkg/blob/main/Snake/GameLogic.c#L263) moves the snake forward based on the current direction, checks for collisions with the grid edges or the snake's own body, and handles apple consumption by growing the snake and increasing the score.

In the end, everything comes together in the [`RunGameLogic`](https://github.com/AstonishedLiker/SnakePkg/blob/main/Snake/GameLogic.c#L333) main loop, which initializes the snake and the first apple, processes input to update direction, updates the game state, and renders the scene. The game also ramps up the speed as your score climbs! Though I'll admit, that part's a bit messy, floating point arithmetic isn't exactly UEFI's strong suit (Floating Point units are disabled, sadly).

I would've linked the code here, but it's hundreds of lines long, and would ruin the reading experience.

Feel free to check the repository though: https://github.com/AstonishedLiker/SnakePkg

## The end product

We can successfully debug SnakePkg within VS Code:

<video controls>
  <source src="ovmf_snake_debugging.mp4" type="video/mp4">
  Your browser doesn't seem to support embedded videos!
</video>

We're also able to play the game normally!

<video controls>
  <source src="ovmf_snake_gameplay.mp4" type="video/mp4">
  Your browser doesn't seem to support embedded videos!
</video>

...on real hardware!!

<video controls>
  <source src="snakepkg_on_thinkpad.mp4" type="video/mp4">
  Your browser doesn't seem to support embedded videos!
</video>

## Conclusion

**Conclusion**

With SnakePkg now running on both QEMU and real hardware, we proved that UEFI development is not just about theory. The project started with basic EDK II concepts, moved through impossible-at-first debugging with LLDB and EDK2Code, and ended with a working game using the Graphics Output Protocol!

This was just the beginning. The skills and tools we built here apply to bigger challenges, like security research or custom firmware projects... The code is open, the process is documented, and the door is wide open for anyone willing to dive in.

Until we meet again!
