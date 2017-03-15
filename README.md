# decom

This is the decom (**DE**vice **COM**munication) library.  
decom is a universal, cross platform, NON BLOCKING, high performance c++ communication library.
It is intended to be a minimalistic abstraction of the OSI layer model.

### Design goals:
- CLEAN code without any warnings on highest compiler level (4), LINT checked
- Production/industrial/automotive code quality
- Cross platform and embedded support
- Header only, no modules
- ONLY usage of standard C++, (C++11) and the STL library
- ONLY usage of standard stdint.h (cstdint) and bool datatypes
- NO other external libraries nor other platform dependencies except for util and communication layers which have architecture dependent folders (windows, linux etc.) because they must touch the hardware, OS API or vendor libs
- Own static/pool message memory management - NO usage of any dynamic memory allocation
- Zero copy of message data - if possible

### Usage of STL containers
In the moment, decom uses somw STL containers like <vector>.
On embedded systems special STL libs like "ustl" may be used.
In later versions, any usage of STL containers should be avoided due to dynamic memory management, which makes decom not suitable for automotive and small embedded applications.

The following layer classification is used:

## Device layer
This is the top most layer, often referred as application layer (7) in OSI model.
The upper layer has only one interface to its lower layer and device specific functions which
are used by the application like read or write.
The namespace for devices is `decom::dev`.

## Protocol layer
This layer has two interfaces. One to the upper layer and one to the lower layer.
Typically a protocol layer does clothing, stripping, checksum calculations, flow control etc.
Any desired count of protocol layers can be chained together.
Protocols may be inserted/deleted dynamically out of the stack. Dynamic object creation is supported.
Try to avoid using threads in protocol layers cause not all platforms may support threads, use `util::timer` instead.
The namespace for protocols is `decom::prot`.

## Communication layer
This is the lowest layer (2). It has only one interface to an upper layer and sends/receives data to/from the hardware or OS API/HAL.
This layer is arcitecture dependent.
The namespace for communicators is `decom::com`.

## Mandatory layer functions
Every layer consists of the decom standard interface routines to pass and receive data from the upper and lower layer. These are:

- `open()`
  Open the layer to send/receive data.
- `close()`
  Close the layer.
- `send()`
  Called by the upper layer to send data to this layer.
- `receive()`
  Called by lower layer to pass received data to this layer.
- `indication()`
  Called by lower layer to indicate a status code/condition to the upper layer.

These functions MUST be implemented for every layer.
Further each layer can have specific API functions to set layer specific protocol params
like baudrate, timings, flow control etc. e.g. and can have a special param ctor.


## Stack creation
The stack is always created with bottom (com) layer FIRST to top (dev) layer last.
Dynamic protocol generation, binding and unbinding (layer delete) is supported.
Due to this mechanism it is possible to insert, for example, a file transfer protocol like
xmodem between a device and a communication port.


The minimalistic stack is a device bound to a communication class without any protocol:

```c++
// create a mini stack
decom::com::serial  ser(1);      // layer2 - create serial COM1 with default params
decom::dev::generic dev(&ser);   // layer7 - create generic device and bind it directly to serial interface
// then use the stack
dev.open(...);           // open the device (and implicit open the rest of the stack)
dev.write(...);          // write something to device
dev.read(...);           // read something from device
dev.close();             // close device (and implicit close the rest of the stack)
```

Example for a more complex stack creation with 3 protocols:

```c++
// create the stack
decom::com::serial ser(1, 115200);        // layer2 - open COM1 with 115 kBaud and default params
decom::prot::test1 prot1(&ser);           // layer3 - create layer 3 routing protocol
decom::prot::test2 prot2(&prot1, p1, p2); // layer4 - create layer 4 transport protocol with additional params P1 and P2
decom::prot::session sess(&prot2);        // layer5 - create layer 5 session protocol
decom::dev::generic dev(&sess);           // layer7 - create generic device and bind to session protcol
// use the stack
dev.open(...);           // open the device (and implicit open the rest of the stack)
dev.write(...);          // write something to device
dev.read(...);           // read something from device
dev.close();             // close device (and implicit close the rest of the stack)
```