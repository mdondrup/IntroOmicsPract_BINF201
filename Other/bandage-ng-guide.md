# Alternative Analysis & Bandage-NG Compilation Guide
## Visualizing Dense Genome and Metagenome Assembly Graphs on Modern Operating Systems

This repository contains a comprehensive overview of alternative tools evaluated for visualizing heavy genome and metagenome graphs (specifically targeting networks exceeding **20,000 nodes**), along with verified, platform-agnostic source-compilation guides for **Bandage-NG** on **macOS (including Apple Silicon M4)** and **Windows 11**.

---

## Part 1: Visualizer Tool Evaluation Summary

Visualizing heavy, complex structures like plant metagenome graphs presents unique technical bottlenecks. Many modern tools offload computing tasks to distinct architecture layers or internet frameworks, which often fail when handling vast datasets or modern hardware restrictions. In addition, many open-source developers do not have Apple developer certificates to allow safe installation. 

### Evaluation matrix

| Tool Evaluated | Core Engine Engine | Setup / Runtime Failure Point |  How it worked out for Heavy Datasets (>20k Nodes) |
| :--- | :--- | :--- | :--- |
| **Assembly Graph Browser (AGB)** | Desktop Layout Engine | Automated Subgraph Slicing | **Failed Requirement:** It clips the graph structural details to preserve performance; does not depict the full topology. |
| **[BandageJS](https://jbrowse.org/demos/bandagejs/)** | WebAssembly (WASM) | Browser Memory Sandboxing | **Scale:** Throws memory allocation warnings and experiences severe lag due to web browser memory limits. It worked with the Stool sample. This is the best no-install option. |
| **GfaViz** | Desktop (Qt5/OGDF) | Missing Subdirectory Target Indexes | **Failed Environment Setup:** The software package is deprecated or unindexed on modern ARM64 package channels. |
| **gfaestus** | GPU (Vulkan / Rust) | Hardcoded Linker Paths & Missing Portability Flags | **Failed Handshake:** The application lacks the explicit portability extension bit (`VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR`) required to interact with Apple Silicon GPUs via MoltenVK. |
| **Bandage-NG** | Desktop (Qt6 / CMake) | Manual Path Configuration | **SUCCESS:** Bypasses missing package indexes and OS security quarantine locks when compiled directly from source. |

---

## Part 2: The Definitive Guide to Installing Bandage-NG

**Bandage-NG** is the modernized, community-driven fork of the original Bandage application, updated to compile natively with **Qt6, C++17, and CMake**. Compiling directly from source ensures absolute hardware compatibility and maximum rendering efficiency.

### 🍏 Compilation Guide for macOS (Intel & Apple Silicon (M1–M5))

Modern macOS versions (including the latest **macOS 15+ / Golden Gate**) isolate third-party developer frameworks to specific paths depending on your chip architecture. Passing an explicit layout path variable to CMake forces a successful compile configuration.

#### 1. Install Toolsets via Homebrew
Make sure you have [Homebrew](https://brew.sh/) installed.
Open your standard Mac terminal and pull down the complete native compiler toolsets:
```bash
brew install qt6 cmake git
```

#### 2. Clone and Setup the Source Code
```bash
git clone https://github.com/asl/BandageNG.git
cd BandageNG
mkdir build && cd build
```

#### 3. Run the Native Compilation Stack
Pass the explicit Homebrew Qt6 prefix. Compiling with `-j4` utilizes multiple CPU cores to build the binary in seconds:

*   **For Apple Silicon (M1/M2/M3/M4) Macs:**
    ```bash
    cmake .. -DCMAKE_PREFIX_PATH=/opt/homebrew/opt/qt6
    make -j4
    ```
*   **For Intel-Based Macs:**
    ```bash
    cmake .. -DCMAKE_PREFIX_PATH=/usr/local/opt/qt6
    make -j4
    ```

#### 4. Launching the Application
```bash
open BandageNG.app
```
> 💡 **Tip:** Drag and drop `BandageNG.app` from your current build folder directly into your Mac's primary `/Applications` directory. Because it was compiled locally  on your machine, it bypasses Apple's Gatekeeper quarantine; no unverified developer certificate prompts or security blockades will occur.

---

### 🪟 Compilation Guide for Windows 11

Windows environments require setting up Microsoft Visual Studio compilers alongside the official Qt online software distribution framework.

#### 1. Install Prerequisites
1. Download and install **[Visual Studio Community Edition](https://visualstudio.microsoft.com/vs/community/)**. Ensure the **"Desktop development with C++"** workload checkbox is selected during setup.
2. Download and install **[CMake for Windows](https://cmake.org/download/)**. Make sure to select the option to **"Add CMake to the system PATH"** during the wizard.
3. Download and install **[Git for Windows](https://gitforwindows.org/)**.

#### 2. Install Qt6 Framework
1. Download the **[Qt Online Installer](https://www.qt.io/download-open-source)**.
2. Complete the free registration to create a Qt open-source developer account.
3. In the component installation tree layout, expand the modern Qt6 directory (e.g., **Qt 6.x**) and verify that the box for **MSVC 2022 (64-bit)** is checked. Proceed with the download.

#### 3. Build Using the Developer Command Prompt
Do not use standard Windows PowerShell or standard Command Prompt. Instead, search your Start Menu for **"Developer Command Prompt for VS 2022"** to guarantee compiler binaries are loaded in your terminal path, then execute:

```cmd
git clone https://github.com/asl/BandageNG.git
cd BandageNG
mkdir build
cd build

:: Generate configuration layout files pointing to where Qt6 is installed
cmake .. -DCMAKE_PREFIX_PATH="C:\Qt\6.x.x\msvc2022_64"

:: Compile a highly optimized release executable binary
cmake --build . --config Release
```
*(Make sure to change `6.x.x` in the directory string to match the exact minor version number of Qt downloaded to your local C drive).*

#### 4. Launching the Application
Your executable file will be generated cleanly inside the build subdirectory layout. Navigate into the folder to open your program interface:
```cmd
cd Release
BandageNG.exe
```
