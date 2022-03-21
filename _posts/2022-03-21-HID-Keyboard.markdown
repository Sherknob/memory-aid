---
layout: post
title:  "HID-Keyboard"
date:   2022-03-21 10:33:00 +0100
categories: HID
---
### My Notes on HID emulation on the Pinephone running Mobian

Github Project: [PineDucky](https://github.com/Sherknob/PineDucky)  

**This blog post is written to explain, how sending reports to HID device works**  

To create a virtual device that will take our reports and pass then trough the USB interface of our Kernel, we'll use a kernel module called configfs. This allows us to create a kernel object from userspace ⁰.  

These kernel objects are called gadgets.  
A few examples are lan over usb, mass storage, human interface device, etc.  
  
**Btw.** this requires kernel version >= 3.19, since configfs was added in 3.19.  

On how to enable configfs and setup your Pinephone check [Setup](https://github.com/Sherknob/PineDucky/tree/master/setup).  
After this, you will need to setup a usb-gadget. Where you'll have to specify, what it can do and how it will anounce itself to the connected host system. This step can be automated with a little script [setup-gadget.sh](https://github.com/Sherknob/PineDucky/blob/master/setup/start-gadget.sh).  

After the setup is complete, you should have a device called `hidg0` in your `/dev/` directory. You can write hid_reports to this device, and they will be transferred to the host system your Pinephone is connected to. 

#### Writing a report
... is pretty straightforward.  

A hid_report consistst of eight uint_8 values in hex. So an example would be `01 00 0b 00 00 00 00 00`.  
**The first hex value** stands for the modifier key.  
The modifier can be one of:  
[Right Meta | Right Alt | Right Shift | Right Control | Left Meta | Left Alt | Left Shift | Left Control].  
 
To set the modifier to left shift for example, just take a uint_8 integer and change the bit of the corrisponding modifiervalue to 1. In our left shift example, this would be `00000010`. This would give us a modifiervalue of hex `2`. You can also chain modifiers together simply by adding another 1 in the corrisponding position. For example left shift + left control would be 00000011, hex `3`.  

**The second hex value** is reserved and shall not be set to anything but `00`.  
And now you can fill all the other spots with your desired keyvalues.  

**Attention for windows user:** Weird `g` key... I encountered a problem while testing my PineDucky project on windows. Everything ran fine on linux, but on windows, just the `g` key was not recognised. After sniffing on the connection with wireshark, I noticed that windows is not picking up any bytes after it sees `0a`. So the string would be transferred incomplete resulting in not printing the `g`. Always positioning the `g` at the last position of the hid_report resolved this issue.

**Letting go of all keys.** This may be needed, if you want to type the same letter twice.  
If you just send for instance the keyvalue for `i`, you are basically holding `i` down, until you let go with a line of zero values. This can also be needed, if you want to capitalize one letter and leave others uncapitalized. Then you'd have to let go of all keys in order to get rid of the shift modifier. **Note:** If you care about performance, you should not let go of all keys after every keypress. In my PineDucky project I checked based on a few conditions whether I should let go of all keys or not. 



Sources:  
⁰ [Linux USB gadget configured through configfs](https://www.kernel.org/doc/html/latest/usb/gadget_configfs.html)  

