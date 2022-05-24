---
layout: post
title:  "My understanding of Bluetooth Low Energy"
date:   2022-05-24 10:28:00 +0100
categories: BLE
---
## My perspective on Bluetooth Low Energy

Feel free to connect me, if I am wrong at any point in time. This is the best I could come up with regarding my current knowledge.

About 3 months ago, I was hired by a company to implement Bluetooth low energy functionality into their existing products.
The requirements were 2 Years lifetime with a single very small battery cell. Custom GATT services and characteristics. Pairing and reconnecting after unexpected disconnect or previous shutdown. Button input enumeration (single button press, long button press). Deep sleep capability (2μA of draw) and scheduled wake ups to do some calculation not regarding Bluetooth low energy.


Basically the physical product is generating some values every 125ms and displaying it to an inbuilt LCD display. These values shall be written into a GATT characteristic and read occasionally by a test program I've written in python3 running on my Linux laptop. Also the physical device shall be configured via a GATT attribute. Deep sleep capability is already working without BLE and the 2 years of lifespan are met. Getting it to work with the BLE stack running is a whole other level. I am working with the STM32WB55 as a development board. It wont be the exact same in production, but is is close enough that I can develop on there. Initially I thought the board will handle all the "go to deep sleep" itself, but I was very wrong.

All that aside, what is Bluetooth Low Energy (further called BLE).

BLE is a form of wireless data transfer, optimized for low energy consumption and therefore very often used in IoT devices. Once the connection is established, the Peripheral device (i.e the BLE SERVER, SLAVE) and the Central device (i.e the BLE CLIENT, MASTER) will negotiate some parameters including "MIN/MAX_connection_interval", "slave_latency" and "supervision_timeout". Obviously there are way more parameters to be exchanged, but they are not that unique to BLE, so they won't be covered here. The parameters mentioned above are the main reason, why BLE can be so power efficient.

After these parameters are set, the peripheral device (i.e our physical device) can go to deep sleep mode between connections defined by the "connection_interval". So if the "connection_interval" is 1 sec. the peripheral can go to sleep for 1 sec. before it has to send an empty packet saying "hey I'm still here, please do not break our connection". If fore some reason our device is not sending any packets, and the time it had not sent any packets is larger than our negotiated "supervision_timeout", the connection is lost.

Assuming the "connection_interval" is set as before and the "supervision_timeout" is 10 sec. Now we would still (if everything goes as planned) send an empty packet every 1 sec. because that is the "connection_interval" we negotiated. Ff we want to save even more energy, we can set a "slave_latency". If the "slave_latency" is set to 5, we (as the peripheral) can skip 5 * "connection_interval" before we have to respond back with an empty packet. This way, we would only wake up once in 5 sec.

But what is the difference between ["slave_latency" of 5, "connection_interval" of 1] and ["slave_latency" of 0, "connection_interval" of 5] you may ask.  
Well, "slave_latency" is really only a part of the slave connection side. The master is still listening on every connection interval, if something was sent. So the slave will be able to occasionally send a packet in between two **"slave_latency" intervals** and the master will receive it, before the next "slave_latency" interval is occurring. Comparing that to setting the "connection_interval" to 5 and the "slave_latency" to 0, the slave cannot send any packet between **connection intervals**, because the master is not listening.

**Interrupts and Interrupt service routines**

Often micro-controllers offer such thing as interrupts. In the form of hardware pins that can be set to low and high externally or flags set in software by the two CPU's. The board I am developing on has two CPU's. One is running the closed source proprietary ST BLE stack (M0 core) and the other (M4 Core) is running everything else (i.e GPIO's including LED's and buttons, internal/external clocks, timeservers and so on). These two CPU's communicate via the inbuilt inter-processor communication controller (IPCC) implemented as a so called mailbox. A general presentation of the mailbox framework is available in the [Linux mailbox documentation](https://www.kernel.org/doc/html/latest/driver-api/mailbox.html). Each processor owns specific register bank and interrupts and the IPCC provides the signaling for six bidirectional channels. Is the M0 core receiving a BLE packet, it can (in some cases) "notify" the M4 core about it via the mailbox, triggering an interrupt. These interrupts can be handled by functions the user provides. A function pointer to those functions will be stored in a register and called when needed. These "notifications" from core to core can and mostly have information in them stored as a "packet" (predefined C-struct). These packets will also be available in the interrupt-handler functions.


**GAP and GATT**

The Generic Access Profile (following called GAP) is a Layer of the BLE stack that is controlling connections, advertisements, device role, advertisement data (i.e company identifier, device name, GATT attribute UUID's) and also scan response data.

There are some situations, where you'd only want the advertisement data, but no connection and no GATT services and characteristics. This is called the "broadcast" mode. This can be used as a "Beacon" to measure distance in a "smart" warehouse for example.

The Generic ATTribute Profile (following called GATT) is a layer of the BLE stack that defines how two connected devices communicate with one another. The concept of services and characteristics is introduced here. A GATT service is a container, that may contain another service (called secondary service) or/and up to a predefined number of GATT characteristics. Characteristics are again containers, but this time holding only one value. This value can range from one to 512 bytes.
Services and characteristics are addressed with Universally Unique Identifier (following called UUID's). One UUID is a unique 128 bit number and used as the standard identification method for computer software nowerdays. A random UUID could look like this: a70349f8-d042-4546-acaa-cce8538f456b ([uuidgenerator](https://www.uuidgenerator.net/)). These UUID's are later mapped to handles to identify the service or characteristic in code (done by the BLE stack as a can remember).

**Permissions**

The service and characteristic implementation has certain permission inbuilt. The user can decide in code running on the M4 (more on shared memory later), what permissions the client has to have, if it wants to access a specific service or characteristic. I don't know, if these permissions are recursive i.e if the a service is read only -> all characteristics in this service are read only. I should potentially test this out :).

**Even more energy efficiency**

Thanks to services and characteristics, the M0 core can decide based on user configuration what to do, if a value is read, written or notified. To save energy, the user would disable the interrupt triggered on value read. The packet containing the value is then sent by the M0 core even if the M4 core is still asleep. That is the same on value write and on value notify (this would make a notify characteristich pretty useless tho :D).

**Shared memory**

Configuring services and characteristics as well as interrupts that will eventually run on the M0 core in code running on the M4 core is possible with the concept of shared memory. This is a memory section accessible by both cores. How read and write access is managed, I don't know.

**Pairing and key exchange**

As I stated in the requirements section at the start of this post, the device should be able to bond to its partner. Bonding means, saving current connection information and using this to reconnect, if the connection is lost. Bonding also work with encrypted connections. The key exchange will happen on the first connect and then saved and reused on further connections [more on bonding](https://www.kynetics.com/docs/2018/BLE_Pairing_and_bonding/).
