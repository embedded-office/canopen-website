---
title: Firmware Upload Example Descriptions
description: This example describes some highlights when uploading and storing data in a FLASH domain.
---

# Firmware Upload

The CANopen block transfer protocoll is efficient for uploading and programming a firmware update. This example explains the implementation of a flash-domain object entry.

## Example Goal

The example focus on a manufacturer specific data type, which allows writing a domain into a FLASH memory area.


## Object Entry Definitions

We define some manufacturer specific entries in the object dictionary (free choice):

| Index | Subindex | Type       | Access     | Value | Description        |
| ----- | -------- | ---------- | ---------- | ----- | ------------------ |
| 6789h | 0        | UNSIGNED8  | Const      | 2     | Highest Subindex   |
| 6789h | 1        | UNSIGNED32 | Write Only | 1     | Control Commands   |
| 6789h | 2        | DOMAIN     | Write Only | 0     | Firmware Image     |

The **subindex 1h** is used to control the firmware reprogramming with some specific commands. For example writing the value `0xdead` will erase the current firmware. Some other command ideas are:

- unlock firmware (statemachine with passphrase or a seed/key algorithm)
- activate firmware (calculate and store checksum to activate new image)
- verify firmware (calculate and return checksum for user interface)

The **subindex 2h** is used to program the firmware image into the FLASH memory region.

### Type: Control Commands

Lets implement the firmware control user type as shown in [CANopen Usage: User Object][1]:

```c
uint32_t FwCtrlSize (CO_OBJ *obj, CO_NODE *node, uint32_t width);
CO_ERR   FwCtrlWrite(CO_OBJ *obj, CO_NODE *node, void *buffer, uint32_t size);

const CO_OBJ_TYPE FwCtrl = { FwCtrlSize, 0, 0, FwCtrlWrite };

#define FW_CTRL ((CO_OBJ_TYPE*)&FwCtrl)
```

#### Erase Firmware

The write access to this object entry triggers actions depending on the writing value. For a most simple implementation, we start erasing the firmware when writing the value `0xdeadbeef` to the object entry. All other write values are ignored.

```c
uint32_t FwCtrlSize(CO_OBJ *obj, CO_NODE *node, uint32_t width)
{
  const CO_OBJ_TYPE *uint32 = CO_TUNSIGNED32;
  return uint32->Size(obj, node, width);
}
CO_ERR FwCtrlWrite(CO_OBJ *obj, CO_NODE *node, void *buffer, uint32_t size)
{
  uint32_t command = *((uint32_t *)buffer);
  CO_ERR   result  = CO_ERR_TYPE_WR;

  if (command == '0xdeadbeef') {

    /* erase your firmware region in FLASH here */

    result = CO_ERR_NONE;
  }
  return result;
}
```

!!! warning

    You should implement a much more sophisticated sequence for unlocking and erasing your FLASH device. This is simplified for demo purpose only!

### Type: Firmware Image

Implementing the firmware image upload user type needs three callback functions:

```c
uint32_t FwImageSize (CO_OBJ *obj, CO_NODE *node, uint32_t width);
CO_ERR   FwImageInit (CO_OBJ *obj, CO_NODE *node);
CO_ERR   FwImageWrite(CO_OBJ *obj, CO_NODE *node, void *buffer, uint32_t size);

const CO_OBJ_TYPE FwImage = { FwImageSize, FwImageInit, 0, FwImageWrite };

#define FW_IMAGE ((CO_OBJ_TYPE*)&FwImage)
```

#### Image size

We want to allow uploading firmware images which are smaller than whole FLASH domain area. For this reason, we return the given width of the started firmware upload sequence as size of this domain.

```c
uint32_t FwImageSize(CO_OBJ *obj, CO_NODE *node, uint32_t width);
{
  CO_OBJ_DOM *domain = (CO_OBJ_DOM*)(obj->Data);
  uint32_t    size   = domain->Size;

  /* allow firmware image smaller or equal to the domain memory area */
  if ((width < size) && (width > 0)) {
    size = width;
  }
  return size;
}
```

#### Image initialization

During multiple the SDO server transfers, the working offset changes. This is internaly done with calling this function with the function id `CO_CTRL_SET_OFF` and a corresponsing offset value in the argument `para`

```c
CO_ERR FwImageInit(CO_OBJ *obj, CO_NODE *node)
{
  CO_OBJ_DOM *domain = (CO_OBJ_DOM*)(obj->Data);

  domain->Offset = 0;
  return (CO_ERR_NONE);
}
```

#### Image write

The core of the firmware image write function is your project specific FLASH write function. The required basic information for calling your function are managed by the stack.

```c
CO_ERR FwImageWrite(CO_OBJ *obj, CO_NODE *node, void *buffer, uint32_t size);
{
  CO_OBJ_DOM *domain  = (CO_OBJ_DOM*)(obj->Data);
  CO_ERR      result  = CO_ERR_TYPE_WR;
  uint32_t    address = (uint32_t)domain->Start + domain->Offset;
  uint32_t    success;

  /* use your FLASH driver for writing the buffer to given address, e.g.: */
  success = MyFlashDriverWrite(address, (uint8_t *buffer), size);

  if (success)
    domain->Offset += size;
    result = CO_ERR_NONE
  }

  return (result);
}
```

!!! info

    If you need to consider additional constraints due to your FLASH backend hardware, this constraints should be handled within your FLASH driver function, e.g.: programm granularity of 8 byte portions due to continuously running ECC checks, and similar.



[1]: ../../usage/dictionary#user-objects
