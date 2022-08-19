# Structure

## Layout of Repository

The following description explains the directories within this repository.

```
  root
  +- src            : --- CANopen Stack source code ---
  |  +- config      : configuration files
  |  +- core        : core services
  |  +- hal         : hardware abstraction interface
  |  +- object      : object type functions
  |  |  +- basic    : basic types
  |  |  +- cia301   : CiA301 types
  |  +- service     : protocol service files
  |  |  +- cia301   : standard protocols
  |  |  +- cia305   : layer setting service
```

When using the CANopen Stack, you need the `src` directory tree only.

### Directory: config

This directory contains the configuration of the CANopen Stack. The intended purpose is to create a library with your cross-compiler with all source files in the directory `src` and a specific configuration, defined in the file `co_cfg.h`:

```
mandatory settings
- CO_SSDO_N     : maximum number of possible SDO servers (default: 1)
- CO_CSDO_N     : maximum number of possible SDO clients (default: 1)
- CO_TPDO_N     : maximum number of possible TPDOs (default: 4)
- CO_RPDO_N     : maximum number of possible RPDOs (default: 4)
- CO_EMCY_N     : maximum number of possible emergency codes (default: 32)
```

```
settings for optimizing resource usage
- USE_LSS       : enable/disable support for LSS functionality (default: 1)
- USE_CSDO      : enable/disable support for SDO clients (default: 1)
```

### Directory: core

This directory holds the general CANopen functions, which are implemented according the specification Cia301.

### Directory: hal

This directory holds a common interface definition used within the CANopen stack to access different chip vendor hardware abstraction layers via `drivers`. The drivers are not included in this directory, they are part of the target specific project. (see an example in [canopen-stm32f7xx][1]).

### Directory: object

This directory includes the implementation of object type functions. The subdirectory `basic` includes the basic data types, which handles values in object dictionary entries.

Other subdirectories (e.g. `cia301`) includes object types, which performs some kind of functionality during reading or writing from/to the object entry. The name of the subdirectory describes the source of specification for this object type.

### Directory: service

This directory holds the implementation of CAN protocols. The subdirectory `cia301` and `cia305` describes the source of specification for the included protocols.

[1]: https://github.com/embedded-office/canopen-stm32f7xx
