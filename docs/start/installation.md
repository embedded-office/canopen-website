---
title: Overview of the CANopen Ecosystem
description: The free CANopen Stack includes multiple repositories for important aspects of embedded software development.
---

# Installation

With version 4.4 the CANopen Stack project introduces an ecosystem, which supports you in project management. This is realized by using multiple repositories with independent version management for important aspects of an embedded software project setup:

- **[canopen-stack][]** - this repository represents the platform independent CANopen stack component.
- **[cmake-scripts][]** - this repository is responsible for the embedded toolchains and the component package management.

!!! info "Target Quickstart Projects"

    === "STM32F4 Series"

        - **[canopen-stm32f4xx][]** - this repository contains a complete Quickstart example setup for the device STM32F446. The adaption to other devices out of the STM32F4xx series are small.
        - **[STM32CubeF4][]** - this fork of the ST Microelectronics HAL package is integrated into the CMake build system and packaged with minimal required source files to get the ST HAL/LL drivers working (No middleware and documentation).

    === "STM32F7 Series"

        - **[canopen-stm32f7xx][]** - this repository contains a complete Quickstart example setup for the device STM32F769. The adaption to other devices out of the STM32F7xx series are small.
        - **[STM32CubeF7][]** - this fork of the ST Microelectronics HAL package is integrated into the CMake build system and packaged with minimal required source files to get the ST HAL/LL drivers working (No middleware and documentation).

See the following illustration which shows the relations between these repositories:

<figure markdown>
  ![The CANopen Stack ecosystem][ecosystem]{ width=700px }
</figure>

## Add Component in CMake (recommended)

The build system is realized with [CMake][] and the [CPM.cmake][] package management. See [cmake-scripts][] for details. Including the CANopen Stack into the project is done during the configuration phase of the build environment. During this phase, the CANopen Stack is fetched in the defined version and is available for usage.

```cmake
set(CO_TARGET   "canopen-stack")
set(CO_PROJECT  "embedded-office/canopen-stack")
set(CO_VERSION  "4.4.0")
CPMAddPackage(
  NAME    ${CO_TARGET}
  URL     https://github.com/${CO_PROJECT}/releases/download/v${CO_VERSION}/${CO_TARGET}-src.zip
  VERSION ${CO_VERSION}
)
```

## Getting the Source

When you just want to use the source code of the platform independent CANopen Stack, all you need to do is getting the repository [canopen-stack][] and use the files in your prefered build system. There are multiple ways in getting the source code:

### Clone Repository

Clone the [canopen-stack][] repository from github. This is good if you want to review and merge future updates into your project:

```
git clone https://github.com/embedded-office/canopen-stack.git
```

### Download ZIP File

Download and unzip the file from the [canopen-stack][] repository. This is good if you want to work on hosts without internet access.

- Click on `Releases`
- Download the package `canopen-stack-src.zip`
- Unzip the archive to your local drive

!!! Note

    This is exactly what the CMake command `CPMAddPackage()` does during project configuration.

### Fork Repository

Fork the github repository [canopen-stack][] into your GitHub account. This is good if you want to collaborate enhancements or bugfixes to the project. Any improvement of the project is highly welcome.

## Cross-compile for Target

Add the source files and include paths to a library project (see [repository structure][]). Create a library with your cross-compiler and share this library together with the include files with your embedded software project.

!!! Note

    This is exactly what the CMake command `CPMAddPackage()` prepares during project configuration.


[ecosystem]: ../assets/images/illustrations/canopen-ecosystem.svg
    "The CANopen Stack ecosystem"
[canopen-stack]: https://github.com/embedded-office/canopen-stack
    "Repository: canopen-stack"
[canopen-stm32f7xx]: https://github.com/embedded-office/canopen-stm32f7xx
    "Repository: canopen-stm32f7xx"
[canopen-stm32f4xx]: https://github.com/embedded-office/canopen-stm32f4xx
    "Repository: canopen-stm32f4xx"
[stm32cubef7]: https://github.com/embedded-office/STM32CubeF7
    "Repository: STM32CubeF7"
[stm32cubef4]: https://github.com/embedded-office/STM32CubeF4
    "Repository: STM32CubeF4"
[repository structure]: ../structure
    "Structure of canopen-stack repository"
[cmake-scripts]: https://github.com/embedded-office/cmake-scripts
    "Repository: cmake-scripts"
[cpm.cmake]: https://github.com/cpm-cmake/CPM.cmake
    "Repository: CPM.cmake"
[cmake]: https://cmake.org/
    "Website: CMake"
