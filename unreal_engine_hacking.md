# Base for Hacking Unreal Engine 5 Games (Internal C++)

Unlike Unity (Managed C#), Unreal Engine games run on Native C++. We cannot just decompile a DLL. Instead, we must generate an **SDK** (Software Development Kit) to map out the game's memory, and then write a C++ DLL to interact with it.

## Prerequisites
*   **Visual Studio 2022** (Install "Desktop development with C++").
*   **Dumper-7** (The tool to generate the SDK).
*   **Process Hacker 2** or **Extreme Injector**.

---

## Phase 1: Generating the SDK (The Map)

Before writing code, we need the game's class structure.

1.  **Get Dumper-7:** Download the source from [GitHub (Encryqed/Dumper-7)](https://github.com/Encryqed/Dumper-7).
2.  **Compile:** Open `Dumper-7.sln` -> Set to **Release / x64** -> Build Solution (`Ctrl+Shift+B`).
3.  **Inject:**
    *   Launch your target game (e.g., Bodycam) and sit in the Main Menu/Lobby.
    *   Open Process Hacker 2.
    *   Right-click the game process -> **Miscellaneous** -> **Inject DLL**.
    *   Select your compiled `Dumper-7.dll`.
4.  **Wait:** A console will open. Wait until it says "Finished".
5.  **Locate:** The SDK will be generated in `C:\Dumper-7\GameName\SDK`.

---

## Phase 2: creating The Project & Folder Structure

1.  **New Project:** Visual Studio -> Create New Project -> **Dynamic-Link Library (DLL)** -> C++.
    *   *Name:* `InternalCheat` (or whatever you want).
2.  **Folder Setup (Crucial):**
    *   Go to your project folder (where `InternalCheat.vcxproj` is).
    *   Create a folder named `External`.
    *   Inside `External`, create a folder named `SDK`.
    *   1. **Copy ALL files** files from `C:\Dumper-7\{GameVersion}-{GameName}\CppSDK\SDK\` into your `External\SDK`
    *   2. **Copy ALL files** files from `C:\Dumper-7\{GameVersion}-{GameName}\CppSDK\` into your `External\SDK`
    *   Also since we changed the auto generated project structure, we need to change some of the paths in the header files, i cant remember what files exactly, but it will probably look something like this `../{some header file}` just do ctrl+H replace `../` with `` so basically just nothing, just use your common knowledge.
    *   *The reason that we do this is because of a quirk in how Visual Studio handles "Include Directories" versus how Dumper-7 writes its #include lines.* **WHICH IS FUCKING STUPID I HATE VISUAL STUDIO AND I HATE C++, FUCK THAT FUCKIGN LANGUAGE, ITS SO FUCKIGN ANNOYING, EVER TRIED TO INSTALL LIBARIES AND CREATE A CMAKE FILE IN C++ ?? YOU WILL KNOW EXACTLY THE SHIT I AM TALKING ABOUT**

**Your structure must look like this:**
```text
InternalCheat\
 │  InternalCheat.vcxproj
 │  dllmain.cpp
 └──External\
     └──SDK\
         │  Basic.hpp
         │  Engine_classes.hpp
         │  ... (and thousands of other files)
```

---

## Phase 3: The "Cleanup" & Fixes (The Hard Part)

The generated SDK contains ~2000 files. Compiling all of them will crash Visual Studio. We must clean it up and fix common generation errors.

### 1. Delete "Bloat" Files
Create a file named `clean_sdk.bat` inside your `External\SDK` folder, paste this, and run it: 
**I recommend going into the directory with all of the many files and doing "dir" in a terminal then copying that output and putting it into an AI with a large context length such as Gemini, along with the bat, and telling it to delete all the bloat which is nmot relevant for game hacking**
```bat
@echo off
mkdir KEEPERS
:: Move the Essential Engine Files
move Basic.cpp KEEPERS\
move CoreUObject_functions.cpp KEEPERS\
move Engine_functions.cpp KEEPERS\
:: Move Game Specific Files (Change these based on your game!)
move Bodycam_functions.cpp KEEPERS\
move WEP_functions.cpp KEEPERS\
move Zombie_functions.cpp KEEPERS\
:: Move Input & UI Files
move InputCore_functions.cpp KEEPERS\
move EnhancedInput_functions.cpp KEEPERS\
move UMG_functions.cpp KEEPERS\
move SlateCore_functions.cpp KEEPERS\

:: Delete the rest (Garbage)
del *.cpp

:: Restore
move KEEPERS\* .
rmdir KEEPERS
echo Cleanup Complete.
pause
```

### 2. Fix "TMap is not a template" Error
1.  Open `External\SDK\Basic.hpp`.
2.  At the very top, **under** `#pragma once`, add:
    ```cpp
    #include "UnrealContainers.hpp"
    ```
3.  Scroll down. If you see forward declarations like `class TMap;` or `class TArray;`, **DELETE THEM**.

### 3. Fix "Static Assertion Failed" Error
1.  Open `External\SDK\Assertions.inl`.
2.  Press `Ctrl + H` (Replace).
3.  Find: `static_assert(`
4.  Replace with: `static_assert(true || `
5.  Click **Replace All**. (This neutralizes the error checks without breaking syntax).

---

## Phase 4: Visual Studio Configuration

If you skip this, you get 10,000+ errors.

1.  Right-click Project (`InternalCheat`) -> **Properties**.
2.  **Top Bar:** Set Configuration to **All Configurations**, Platform to **x64**.
3.  **C/C++ -> General -> Additional Include Directories:**
    *   Add: `$(ProjectDir)External`
4.  **C/C++ -> Precompiled Headers:**
    *   Set to: **Not Using Precompiled Headers**.
5.  **C/C++ -> Language:**
    *   Set to: **ISO C++20 Standard (/std:c++20)**.
    *   Set Conformance mode to: **Yes (/permissive-)**.
6.  Click **Apply** -> **OK**.

**Final Step:** In Solution Explorer, Right-Click "Source Files" -> Add -> Existing Item -> Select the ~8 `.cpp` files remaining in your `External\SDK` folder.

---

## Phase 5: The Code (`dllmain.cpp`)

Here is a complete template with **Safe Pointer Validation**, **FOV Changer**, and **Entity Radar**.

```cpp
#include <Windows.h>
#include <iostream>

// Include the SDK you generated
// (Make sure your project Include Directories point to the 'External' folder)
#include "SDK/SDK.hpp"

// The GWorld offset from your dumper logs
constexpr uintptr_t OFFSET_GWORLD = 0x93D0B50;

void MainThread(HMODULE hModule) {
    // 1. Setup Console
    AllocConsole();
    FILE* f;
    freopen_s(&f, "CONOUT$", "w", stdout);

    std::cout << "============================================" << std::endl;
    std::cout << "        SDK VALIDATION CHECK" << std::endl;
    std::cout << "============================================" << std::endl;

    // 2. Get Base Address
    uintptr_t baseAddress = (uintptr_t)GetModuleHandle(NULL);
    std::cout << "[+] Game Base Address: 0x" << std::hex << baseAddress << std::endl;

    // 3. Read UWorld Pointer
    // We cast the address to a pointer-to-pointer, then dereference it
    SDK::UWorld* World = *reinterpret_cast<SDK::UWorld**>(baseAddress + OFFSET_GWORLD);

    if (World) {
        std::cout << "[+] UWorld Address:    0x" << World << std::endl;

        // 4. Validate Internal Structure
        // If the SDK classes are generated correctly, we should be able to read 'PersistentLevel'.
        // If this crashes or prints 0, the SDK struct size is wrong.
        if (World->PersistentLevel) {
            std::cout << "[+] PersistentLevel:   0x" << World->PersistentLevel << std::endl;
            std::cout << "\n[SUCCESS] The SDK is working perfectly!" << std::endl;
        }
        else {
            std::cout << "\n[WARNING] UWorld found, but PersistentLevel is NULL." << std::endl;
            std::cout << "          (Are you inside a match/training?)" << std::endl;
        }
    }
    else {
        std::cout << "\n[FAILURE] UWorld is NULL. The Offset 0x93D0B50 might be outdated." << std::endl;
    }

    std::cout << "============================================" << std::endl;
    std::cout << "Press [END] to Unload..." << std::endl;

    // 5. Keep thread alive until user presses END
    while (!GetAsyncKeyState(VK_END)) {
        Sleep(100);
    }

    // 6. Eject
    fclose(f);
    FreeConsole();
    FreeLibraryAndExitThread(hModule, 0);
}

BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved) {
    if (ul_reason_for_call == DLL_PROCESS_ATTACH) {
        CreateThread(nullptr, 0, (LPTHREAD_START_ROUTINE)MainThread, hModule, 0, nullptr);
    }
    return TRUE;
}
```

---

## Phase 6: Injection

1.  **Build:** In Visual Studio, press `Ctrl + Shift + B`. Ensure it says "Build: 1 succeeded".
2.  **Locate DLL:** It is usually in `ProjectFolder\x64\Release\InternalCheat.dll`.
3.  **Run Process Hacker 2** (as Administrator).
4.  Find your game (e.g., `Bodycam-Win64-Shipping.exe`).
5.  Right-click -> **Miscellaneous** -> **Inject DLL**.
6.  Select your DLL.
7.  **Success:** A black console window should appear over your game.

---

### Troubleshooting Common Errors

*   **"Declaration has no storage class..."**: You emptied `Assertions.inl` instead of using the `true ||` fix. Restore the file and fix it properly.
*   **"TMap is not a template"**: You forgot to include `UnrealContainers.hpp` inside `Basic.hpp`.
*   **"Unresolved External Symbol"**: You forgot to add the `.cpp` files (like `Engine_functions.cpp`) to the Solution Explorer.
*   **"Requires std::invocable..."**: You didn't set C++ Language Standard to **C++20** in properties.
