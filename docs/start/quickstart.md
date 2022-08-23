---
title: The CANopen Quickstart Project
description: The quickstart example describes in detail the steps to a running CANopen node with the free CANopen Stack.
---

# Quickstart

The quickstart example describes in detail the steps to build a CANopen node. The source files are included in the example repository [canopen-stm32f7xx][]. To get most out of this article, you should clone this repository and follow in the real files during reading this article.

For setting up the hardware and software environment, please follow the `README.md` in the root of the example repository [canopen-stm32f7xx][].

## Functional Specification

In this quickstart example, we will create a *CANopen Clock*. This clock is not a serious application, the example just illustrates the key principles of creating a device using the CANopen Stack.

- The CANopen clock shall measure the time while the device is switched to `OPERATIONAL` mode.
- In `OPERATIONAL` mode, the node will transmit a `PDO` every second.
- The `SDO` server allows access to the device information at any time.

### Architectural Overview

The following figure shows the layered architecture of the CANopen Clock device and the related directories.

<figure markdown>
  ![architectural overview][]{ width=700px }
</figure>


The following descriptions explains in detail the important parts of the directory `src/app/...`. All other directories and files are described within the source files.

The directory `src/app/...` contains three modules:

- `clock_spec.c/h` - this module configures the CANopen Stack layer
- `clock_hw.c/h` - this module connects the CANopen Stack layer to the hardware
- `clock_app.c/h` -  this module includes the CANopen application

## Clock Spec

The main settings of the node are configured inside the `CO_NODE_SPEC` struct. This struct is not used or modified after initialization is finished. This allows you to declared this structure as a constant.

Due to the fact, that the central element of a CANopen node is the object dictionary, we start with the description of the specification of our object dictionary.

You find the specification in the file `src/app/clock_spec.c`.

### Object Dictionary

#### Dictionary Object Array

##### Description

To keep the software as simple as possible, we will use a static object dictionary. In this case, the object dictionary is an array of object entries, declared as a constant array of object entries of type `CO_OBJ`. 

##### Implementation

```c
  :
#define APP_OBJ_N         128u                /* Object dictionary max size  */
  :
/* define the static object dictionary */
const CO_OBJ ClockOD[APP_OBJ_N] = {
    :
  /* here is your array of object entries */
    :
  CO_OBJ_DICT_ENDMARK  /* mark end of used objects */
};
  :
```

#### Mandatory Object Entries

##### Description

When we want to achieve compliance with the CiA301 specification, the object dictionary must hold some mandatory object entries:

| Index:sub | Type       | Access     | Value         | Description        |
| --------- | ---------- | ---------- | ------------- | ------------------ |
| 1000h:00  | UNSIGNED32 | Const      | 0             | Device Type        |
| 1001h:00  | UNSIGNED8  | Read-only  | 0             | Error Register     |
| 1014h:00  | UNSIGNED32 | Const      | 80h + node ID | COB-ID EMCY        |
| 1017h:00  | UNSIGNED16 | Read-Write | 0             | Heartbeat Producer |
| 1018h:00  | UNSIGNED8  | Const      | 4             | *Identity Object*  |
| 1018h:01  | UNSIGNED32 | Const      | 0             | - Vendor ID        |
| 1018h:02  | UNSIGNED32 | Const      | 0             | - Product code     |
| 1018h:03  | UNSIGNED32 | Const      | 0             | - Revision number  |
| 1018h:04  | UNSIGNED32 | Const      | 0             | - Serial number    |

##### Implementation

The configuration of each object entry is a single configuration line within the object dictionary `CO_OBJ` array. The following figure shows how a single line is constructed:

<figure markdown>
  ![Object Entry Definition][]{ width=700px }
</figure>

The `CO_KEY` macro creates out of "**index, subindex**" and the "**property flags**" the unique object entry key, which is used for addressing an object entry. The property flags are a collection of type settings (more details, see in configuration of [Property Flags][]). A letter indicates `ON`, while an underscore (`_`) at the same place indicates `OFF`:

- `RW` - The access mode flags (read, write by network)
- `NAP` - The object type flags (node-id considered, asynchronous trigger, PDO mappable)
- `D` - The direct storage flag (used to store the value directly in the object data field)

The **object type** references the object type functions for this object entry. The **object data** is a pointer to the object entry data (or in some cases a object data structure). What kind of data the object type requires, is listed in configuration of [Object Type Interface][].

With this knowledge, the mandatory entries are added with the following lines of code:

```c
  :
{CO_KEY(0x1000, 0, CO_OBJ_____R_), CO_TUNSIGNED32, (CO_DATA)(&Obj1000_00_20)},
{CO_KEY(0x1001, 0, CO_OBJ_____R_), CO_TUNSIGNED8,  (CO_DATA)(&Obj1001_00_08)},
{CO_KEY(0x1014, 0, CO_OBJ__N__R_), CO_TEMCY_ID,    (CO_DATA)(&Obj1014_00_20)},
{CO_KEY(0x1017, 0, CO_OBJ_____RW), CO_THB_PROD,    (CO_DATA)(&Obj1017_00_10)},

{CO_KEY(0x1018, 0, CO_OBJ_D___R_), CO_TUNSIGNED8,  (CO_DATA)(4)             },
{CO_KEY(0x1018, 1, CO_OBJ_____R_), CO_TUNSIGNED32, (CO_DATA)(&Obj1018_01_20)},
{CO_KEY(0x1018, 2, CO_OBJ_____R_), CO_TUNSIGNED32, (CO_DATA)(&Obj1018_02_20)},
{CO_KEY(0x1018, 3, CO_OBJ_____R_), CO_TUNSIGNED32, (CO_DATA)(&Obj1018_03_20)},
{CO_KEY(0x1018, 4, CO_OBJ_____R_), CO_TUNSIGNED32, (CO_DATA)(&Obj1018_04_20)},
  :
```

!!! warning "Important"

    The CANopen Stack relies on a binary search algorithm to ensure that object dictionary entries are found quickly. Because of this, you must keep the index / subindex of all entries in the object dictionary sorted in ascending order.

Most of these entries are constant. We can place their values in read-only FLASH memory. This is not the case for the error register object `1001h` and the heartbeat producer `1017h`. These entries may change during runtime. Therefore, we need to declare global variables in RAM memory to hold the runtime value of these entries:

```c
/* allocate variables for dynamic runtime value in RAM */
uint8_t  Obj1001_00_08 = 0;
uint16_t Obj1017_00_08 = 0;

/* allocate variables for constant values in FLASH */
const  uint32_t Obj1000_00_20 = 0x00000000L;
const  uint32_t Obj1014_00_20 = 0x00000080L;
const  uint32_t Obj1018_01_20 = 0x00000000L;
const  uint32_t Obj1018_02_20 = 0x00000000L;
const  uint32_t Obj1018_03_20 = 0x00000000L;
const  uint32_t Obj1018_04_20 = 0x00000000L;
```

A pointer to the variables and constants are stored in the corresponding object dictionary entry. The subindex 0 of the object record `1018h` holds a single byte which is constant. We can store the value directly in the object data field (so we need no constant variable) and set the `D` flag in the properties (compare: `CO_OBJ_D___R_` vs. `CO_OBJ_____R_` - and be careful when specify your object entries).

!!! warning "Important"
    
    When using architectures with pointer types lower than 32bit (e.g. 16bit microcontrollers), you can store only values up to the pointer width directly in the object dictionary. For larger values declare a constant variable and place a pointer to this constant into the object dictionary!
    ```

#### SDO Server Communication

##### Description

The settings for the SDO server are defined in CiA301 and must contain the following object dictionary entries:

| Index:sub | Type       | Access | Value          | Description                       |
| --------- | ---------- | ------ | -------------- | --------------------------------- |
| 1200h:00  | UNSIGNED8  | Const  | 2              | *Communication Object SDO Server* |
| 1200h:01  | UNSIGNED32 | Const  | 600h + node ID | - SDO Server Request COBID        |
| 1200h:02  | UNSIGNED32 | Const  | 580h + node ID | - SDO Server Response COBID       |

##### Implementation

The following lines add the SDO server entries to the object dictionary:

```c
/* allocate variables for constant values in FLASH */
const  uint32_t Obj1200_01_20 = CO_COBID_SDO_REQUEST();
const  uint32_t Obj1200_02_20 = CO_COBID_SDO_RESPONSE();
  :
/* within object dictionary */
{CO_KEY(0x1200, 0, CO_OBJ_D___R_), CO_TUNSIGNED8,  (CO_DATA)(2)},
{CO_KEY(0x1200, 1, CO_OBJ__N__R_), CO_TUNSIGNED32, (CO_DATA)(&Obj1200_01_20)},
{CO_KEY(0x1200, 2, CO_OBJ__N__R_), CO_TUNSIGNED32, (CO_DATA)(&Obj1200_02_20)},
  :
```

The predefined COBIDs are dependent on the actual node ID. For this reason, the CANopen stack allows you to specify entries whose value depends on the current node ID at runtime. This behavior is specified using the `N` flag in the `CO_OBJ__N__R_` properties.

#### Application Object Entries

##### Description

We need to add some manufacturer specific object entries to support the clock of the example application:

| Index:sub | Type       | Access    | Value | Description    |
| --------- | ---------- | --------- | ----- | -------------- |
| 2100h:00  | UNSIGNED8  | Const     | 3     | *Clock Object* |
| 2100h:01  | UNSIGNED32 | Read Only | 0     | - Hour         |
| 2100h:02  | UNSIGNED8  | Read Only | 0     | - Minute       |
| 2100h:03  | UNSIGNED8  | Read Only | 0     | - Second       |

!!! info

    These entries are placed within the manufacturer-specific area (from `2000h` up to `5FFFh`) and can be chosen freely (see CiA301). Entries outside of this range cannot be chosen freely, and should conform to the various CiA standards and profiles (e.g. CiA301 for communication profile area, CiA401 for generic IO modules, etc).

##### Implementation

These entries are created using the following lines of code:

```c
/* allocate variables for dynamic runtime value in RAM */
uint32_t Obj2100_01_20 = 0;
uint8_t  Obj2100_02_08 = 0;
uint8_t  Obj2100_03_08 = 0;
  :
/* within object dictionary */
{CO_KEY(0x2100, 0, CO_OBJ_D___R_), CO_TUNSIGNED8,  (CO_DATA)(3)},
{CO_KEY(0x2100, 1, CO_OBJ____PR_), CO_TUNSIGNED32, (CO_DATA)(&Obj2100_01_20)},
{CO_KEY(0x2100, 2, CO_OBJ____PR_), CO_TUNSIGNED8,  (CO_DATA)(&Obj2100_02_08)},
{CO_KEY(0x2100, 3, CO_OBJ___APR_), CO_TUNSIGNED8,  (CO_DATA)(&Obj2100_03_08)},
  :
```

The flag `CO_OBJ___A___` for the object entry `2100h:03` enables the "*asynchronous transmission trigger*" for PDOs. This means: when changing the value of this object entry, all PDOs with an active mapping to this object are triggered for transmission. We use this mechanism to achieve the PDO transmission on each write access to the second.

!!! warning "Important"

    The asynchronous transmission trigger is provided for the basic type functions: `CO_TUNSIGNED8`, `CO_TSIGNED8`, `CO_TUNSIGNED16`, `CO_TSIGNED16`, `CO_TUNSIGNED32` and `CO_TSIGNED32`.

#### TPDO Communication

##### Description

The communication settings for the TPDO must contain the following object entries:

| Index:sub | Type       | Access | Value               | Description                       |
| ----------| ---------- | ------ | ------------------- | --------------------------------- |
| 1800h:00  | UNSIGNED8  | Const  | 2                   | *Communication Object TPDO #0*    |
| 1800h:01  | UNSIGNED32 | Const  | 40000180h + node ID | - PDO transmission COBID (no RTR) |
| 1800h:02  | UNSIGNED8  | Const  | 254                 | - PDO transmission type           |

##### Implementation

See the following lines in the object dictionary:

```c
/* allocate variables for constant values in FLASH */
const  uint32_t Obj1800_01_20 = CO_COBID_TPDO_DEFAULT(0);
  :
/* within object dictionary */
{CO_KEY(0x1800, 0, CO_OBJ_D___R_), CO_TUNSIGNED8 , (CO_DATA)(2)             },
{CO_KEY(0x1800, 1, CO_OBJ__N__R_), CO_TUNSIGNED32, (CO_DATA)(&Obj1800_01_20)},
{CO_KEY(0x1800, 2, CO_OBJ_D___R_), CO_TUNSIGNED8 , (CO_DATA)(254)           },
  :
```

The CANopen stack does not support remote CAN frames as they are no longer recommended for new devices. The use of RTR frames in CANopen devices has been deprecated for many years now. Bit 30 in `1800h:01` indicates that a remote transfer request (RTR) is not allowed for this PDO. The CAN identifier `180h + node-ID` is the recommended value from the pre-defined connection set.

#### TPDO Data Mapping

##### Description

The mapping settings for the TPDO must contain the following object entries:

| Index:sub | Type       | Access | Value     | Description                |
| --------- | ---------- | ------ | --------- | -------------------------- |
| 1A00h:00  | UNSIGNED8  | Const  | 3         | *Mapping Object TPDO #0*   |
| 1A00h:01  | UNSIGNED32 | Const  | 21000120h | - map: 32-bit clock hour   |
| 1A00h:02  | UNSIGNED32 | Const  | 21000208h | - map:  8-bit clock minute |
| 1A00h:03  | UNSIGNED32 | Const  | 21000308h | - map:  8-bit clock second |

How we get these values is explained in section configuration of [PDO mapping][].

##### Implementation

This way of defining the payload for PDOs is part of the CiA301 standard and leads us to the following lines in the object dictionary:

```c
/* allocate variables for constant values in FLASH */
const  uint32_t Obj1A00_01_20 = CO_LINK(0x2100, 0x01, 32);
const  uint32_t Obj1A00_02_20 = CO_LINK(0x2100, 0x02,  8);
const  uint32_t Obj1A00_03_20 = CO_LINK(0x2100, 0x03,  8);
  :
/* within object dictionary */
{CO_KEY(0x1A00, 0, CO_OBJ_D___R_), CO_TUNSIGNED8 , (CO_DATA)(3)             },
{CO_KEY(0x1A00, 1, CO_OBJ_____R_), CO_TUNSIGNED32, (CO_DATA)(&Obj1A00_01_20)},
{CO_KEY(0x1A00, 2, CO_OBJ_____R_), CO_TUNSIGNED32, (CO_DATA)(&Obj1A00_02_20)},
{CO_KEY(0x1A00, 3, CO_OBJ_____R_), CO_TUNSIGNED32, (CO_DATA)(&Obj1A00_03_20)},
  :
```

### EMCY Error Specification

#### Application EMCY

##### Description

We want to send a EMCY message, when we detect a hardware error (e.g. an unplugged EEPROM). The EMCY error code and error register flag is defined in the following table:

| EMCY ID             | Error Register (`1001h`)      | EMCY Code                      |
| ------------------- | ----------------------------- | ------------------------------ |
| `APP_ERR_ID_EEPROM` | Bit 0 (`CO_EMCY_REG_GENERAL`) | 0x5000 (`CO_EMCY_CODE_HW_ERR`) |

##### Implementation

We define all possible identifiers as a handy enumeration.

```c
enum {
  APP_ERR_ID_EEPROM = 0,

  APP_ERR_ID_NUM    /* number of EMCYs in application */
};
```

The CANopen stack behavior for each of these EMCY identifier is defined in an EMCY table. Here we define the related error register bit and the EMCY code. The defines `CO_EMCY_REG_...` for the error register bits and `CO_EMCY_CODE_...` for the EMCY code base values are specified values out of the CANopen specification.

```c
static CO_EMCY_TBL AppEmcyTbl[APP_ERR_ID_NUM] = {
  { CO_EMCY_REG_GENERAL, CO_EMCY_CODE_HW_ERR }    /* APP_ERR_ID_EEPROM */
};
```

!!! info "Possible extension"

    With these enumerations in place, we can call the EMCY service functions in our application. For example we can add the following line at the end of the clock application to store the operational time in `hh:mm`:

    ```c
      :
    /* store operational time (hour and minute) in NVM */
    if (second == 0) {
        uint32_t num;
        num  = COIfNvmWrite(&Clk.If, 0, &hour,   4);
        num += COIfNvmWrite(&Clk.If, 0, &minute, 1);
        if (num != 5) {
            COEmcySet(&Clk.Emcy, APP_ERR_ID_EEPROM, 0); /*no user data*/
        } else {
            COEmcyClr(&Clk.Emcy, APP_ERR_ID_EEPROM);
        }
    }
    ```

    *Reading the stored value during startup, or data protection with 2 toggling storage locations is left as training for you.*

### CANopen Timers

#### Application Timer

##### Description

The CANopen stack provides flexible timers for protocol and application usage. 

##### Implementation

Each software timer needs some memory for managing the lists and states of the timed action events:

```c
  :
#define APP_TMR_N         16u                /* Number of software timers */
  :
CO_TMR_MEM TmrMem[APP_TMR_N];                /* Allocate timer memory */
  :
```

### SDO Server Memory

#### Transfer Buffers

##### Description

The CANopen node requires for each SDO server a certain amount of transmission buffers, in case the client is using segmented or block transfers to access large objects.

##### Implementation

Each SDO server needs memory for the segmented or block transfer requests.

```c
uint8_t SdoSrvMem[CO_SSDO_N * CO_SDO_BUF_BYTE];
```

### Specification Structure

#### Additional Settings

##### Description

We want to act the CANopen node with

- a NodeId of `1`, and
- a CAN network baudrate of `250kBaud`

For our timer driver accuracy we want

- timer granularity: `1µs`

##### Implementation

The required CANopen definitions are simple defines, provided for use in the specification structure below.

We need to implement the timer driver granularity in the driver. The following define is used to fill the specification structure in a readable form with all the collected node specification settings:

```c
  :
#define APP_NODE_ID       1u          /* CANopen node ID        */
#define APP_BAUDRATE      250000u     /* CAN baudrate          */
#define APP_TICKS_PER_SEC 1000000u    /* Timer frequency in Hz */
  :
CO_NODE_SPEC AppSpec = {
  APP_NODE_ID,                          /* default Node-Id                */
  APP_BAUDRATE,                         /* default Baudrate               */
  (CO_OBJ *)&ClockOD[0],                /* pointer to object dictionary   */
  APP_OBJ_N,                            /* object dictionary max length   */
  &AppEmcyTbl[0],                       /* EMCY code & register bit table */
  &TmrMem[0],                           /* pointer to timer memory blocks */
  APP_TMR_N,                            /* number of timer memory blocks  */
  APP_TICKS_PER_SEC,                    /* timer clock frequency in Hz    */
  (CO_IF_DRV *)&AppDriver,              /* select drivers for application */
  &SdoSrvMem[0]                         /* SDO Transfer Buffer Memory     */
};
```

## Clock HW

### Driver Interface

#### Select Driver for Node

##### Description

For connecting a CANopen node to the microcontroller hardware, you need three drivers:

- for the CAN controller, 
- for a hardware timer, and
- for a non-volatile memory

You find the connection setup in the file `src/app/clock_hw.c`. The possible drivers are provided in the `/driver` directions.

##### Implementation

You can use a chip-vendor provided HAL for the implementation of the drivers, or use direct register access to perform the required actions - whatever you like. This example project provides drivers, using the ST Microelectronics HAL layer.

First we select and connect a set of drivers to the CANopen stack:

```c
  :
/* select application drivers */
#include "drv_can1_stm32f7xx.h"       /* CAN driver (CAN1)              */
#include "drv_timer2_stm32f7xx.h"     /* Timer driver (TIM2)            */
#include "drv_nvm_i2c1_at24c256.h"    /* NVM driver (AT24C256 via I2C1) */
  :
struct CO_IF_DRV_T AppDriver = {
    &STM32F7xxCan1Driver,
    &STM32F7xxTimer2Driver,
    &I2C1_AT24C256NvmDriver
};
  :
```

When you write your device driver, you will need to set up a hardware timer interrupt within your low-level layer and configure a periodic interrupt source with a frequency of `APP_TICKS_PER_SEC`. The timer interrupt service handler should look something like this:

```c
/* ST HAL TIM2 overflow interrupt callback */
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htmr)
{
    /* collect elapsed timed actions */
    COTmrService(&Clk.Tmr);
}
```

Furthermore, the CAN bus message reception should work with receive interrupts to avoid losing messages. The CAN receive interrupt handler should look similar to:

```c
/* ST HAL CAN receive interrupt callback */
void HAL_CAN_RxFifo0MsgPendingCallback(CAN_HandleTypeDef *hcan)
{
    /* process CAN frame with CANopen protocol */
    CONodeProcess(&Clk);
}
```

## Clock App

### Application Structure

The CANopen application is realized in functions, reflecting two phases of the application:

- *Application Startup*: where initialization of hardware and CANopen layer takes place. The function holds the background loop for processing timer events.
- *Application Callback*: the cyclic started function holding the running application.

The application for this tiny example is implemented in a single file `src/app/clock_app.c`.

#### Application Start

##### Description

The CANopen Stack needs to setup once during startup. We want to start the CANopen node automatically with the NMT `OPERATIONAL` mode.

##### Implementation

This function is responsible for the CANopen Stack startup. The startup needs to connect the CANopen Stack layer with the filled specification structure:

```c
CONodeInit(&Clk, (CO_NODE_SPEC *)&AppSpec);
if (CONodeGetErr(&Clk) != CO_ERR_NONE) {
  while(1);
}
```

We use CANopen software timer to create a cyclic function call to the callback function `AppClock()` with a period of 1s (equal: 1000ms):

```c
ticks = COTmrGetTicks(&Clk.Tmr, 1000, CO_TMR_UNIT_1MS);
COTmrCreate(&Clk.Tmr, 0, ticks, AppClock, &Clk);
```

Finally, we start the CANopen node and set it automatically to NMT mode: 'OPERATIONAL':

```c
CONodeStart(&Clk);
CONmtSetMode(&Clk.Nmt, CO_OPERATIONAL);
```

#### Application Callback

##### Description

The main functionallity is running once every second. We want to increase the clock values when in NMT `OPERATIONAL` mode. Otherwise, the clock stays unchanges.

##### Implementation

The timer callback function `AppClock()` includes the main functionality of the clock node:

```c
/* timer callback function, called every 1000ms */
static void AppClock(void *p_arg)
{
  CO_NODE  *node;
  CO_OBJ   *od_sec;
  CO_OBJ   *od_min;
  CO_OBJ   *od_hr;
  uint8_t   second;
  uint8_t   minute;
  uint32_t  hour;

  /* For flexible usage (not needed, but nice to show), we use the argument
   * as reference to the CANopen node object. If no node is given, we ignore
   * the function call by returning immediatelly.
   */
  node = (CO_NODE *)p_arg;
  if (node == 0) {
    return;
  }

  /* Main functionality: when we are in operational mode, we get the current
   * clock values out of the object dictionary, increment the seconds and
   * update all clock values in the object dictionary.
   */
  if (CONmtGetMode(&node->Nmt) == CO_OPERATIONAL) {

    od_sec = CODictFind(&node->Dict, CO_DEV(0x2100, 3));
    od_min = CODictFind(&node->Dict, CO_DEV(0x2100, 2));
    od_hr  = CODictFind(&node->Dict, CO_DEV(0x2100, 1));

    COObjRdValue(od_sec, node, (void *)&second, sizeof(second));
    COObjRdValue(od_min, node, (void *)&minute, sizeof(minute));
    COObjRdValue(od_hr , node, (void *)&hour  , sizeof(hour));

    second++;
    if (second >= 60) {
      second = 0;
      minute++;
    }
    if (minute >= 60) {
      minute = 0;
      hour++;
    }

    COObjWrValue(od_hr , node, (void *)&hour  , sizeof(hour));
    COObjWrValue(od_min, node, (void *)&minute, sizeof(minute));
    COObjWrValue(od_sec, node, (void *)&second, sizeof(second));
  }
}
```

!!! info

    The last write access with `COObjWrValue()` triggers the asynchronous transmission of the PDO, because the corresponding object entry is defined with the object property flag `A`.



[architectural overview]: ../assets/images/illustrations/quickstart-architecture.svg
    "Architectural Overview"
[object entry definition]: ../assets/images/illustrations/canopen-object-entry.svg
    "Object Entry Definition"
[canopen-stm32f7xx]: https://github.com/embedded-office/canopen-stm32f7xx
    "Repository: canopen-stm32f7xx"
[pdo mapping]: ../../usage/configuration/#pdo-mapping-value
    "Configuration of PDO mapping"
[property flags]: ../../usage/configuration/#property-flags
    "Configuration of Property Flags"
[object type interface]: ../../usage/configuration/#object-type-interface
    "Configuration of Object Type Interface"
