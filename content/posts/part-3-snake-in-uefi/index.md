+++
title = "(Part 3) Snake in... UEFI?"
description = "With an overview of the UEFI boot process complete, we can finally start practicing!"
date = "2026-02-14T14:07:42.000+01:00"
toc = true
tags = ["edk2", "uefi", "firmware"]
+++

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

- `<log_file>`: the serial output of the VM (where we'll grab the applicaiton's image base from)
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

In essence, 
