# HHKB Reverse engineering

I'm going to use this place to write down my notes and discoveries trying to reverse the HHKB Keymap Tool, I have never done such a thing before so a lot of this might be obvious. It is perhaps more of a writeup.

I have been thinking about doing this for about a year now, so i already knew it was written in .NET. Therefore i decided to use Windows, and after some quick researching i found dnSpy, and practiced by solving this [crackme](https://crackmes.one/crackme/5ab77f6633c5d40ad448cc20)

I first downloaded the tool and tried opening it up, but quickly figured out I should instead open the installed binary. After that i spent some time just looking around trying to understand. Looked at various enums. I quite hilariously found the full path and orignal line number of some source code in UpdateChecker.cs. I guess this is a compile-time debug thing, but I'm not quite sure. Finally i found the USBDriver.cs, which was quite interesting.

I've previously wrangeled with udev rules and such with HID devices, and I've thought a little about how I have no clue how I will be able to communicate with the keyboard. After reading the `FindKnownHidDevices` it sorta clicked that i was looking at the same thing here. I put a breakpoint and took a screenshot so i would be able to scan for the devices by their vendorId and productId.

```
vendorId: 0x4FE
productIds: [ 0x20, 0x21, 0x22 ]
pattern (?): "mi_02"
```

I'm learning rust so I also looked for a library/crate, and found the [hidapi](https://github.com/ruabmbua/hidapi-rs) [crate](https://crates.io/crates/hidapi), which i think i will be using to interface with the keyboard later. It's just a wrapper around the hidapi C library.

Now my plan is to continue reading the source code, try to understand the keyboard layout format (i already know it's just two lists of integers from skimming some of the source code, and the exported layout files), and to just try to get a "ping" or some other form of response using the rust library.

## UsbDriver
### `int Send(byte[] sendData, int timeout)`
This method sends the data to the HID device. It seems it opens a buffer that is as big as the device can handle(?), and then fills it up from the **2nd** index (`[1]`)...
```cs
byte[] Report = new byte[(int)device.Caps.OutputReportByteLength];
Report[0] = 0;
for (int i = 0; i < sendData.Length; i++) {
	Report[i + 1] = sendData[i];
}

// sig: (IntPtr, byte[] buff, uint writeBytes, ref uint bytesWritten, IntPtr)
USBDriver.WriteFile(device.HidDevice, Report, (uint)device.Caps.OutputReportByteLength, ref sentLength, IntPtr.Zero);

if (sentLength == 0U) {
	return -1;
}
```

I'm not sure why it never checks the `Report` buffer actually is big enough to fit `sendData`, nor why it starts at `[1]`. It also does not verify it wrote as much as asked for, it only checks that `sentLength != 0`.

### `byte[] Receive(int timeout)`
When receiving, it does the reverse of the +1 in send.

```cs
byte[] Report = new byte[(int)device.Caps.InputReportByteLength];

// -- snip --

if (receiveLength == 0U) {
	return nil;
}

byte[] array = new byte[ConstDefinition.BufferSizeUSB];
int num = 0;
while ((long)num < (long)((ulong)(receiveLength - 1U))) {
	array[num] = Report[num + 1];
	num++;
}
```

Interestingly here the `ConstDefinition.BufferSizeUSB` is a constant `64`. This seems kinda small to me. Two layers of `[a-z0-9]` is something like 72 keys. Perhaps it sends multiple messages?

I put some breakpoints and it Report had length 65 on the first breakpoint hit, and the first byte was a 0 as expected. The message was `[ 0x55, 0x55, 0x1, 0x0, 0x0, ... ]`. The 3 first bytes seem to be some sort of header, the pattern repeats for the second message. The remaining bytes are 0.

I stepped out and realized this was all part of a GetKeyboardInformation, which does a `WriteReadCheck`. It sends `[ 0xAA, 0xAA, 0x02 ]`, and expects the return data to start with `[ 0x55, 0x55, 0x02, 0x00, 0x00, 0x30, ... ]`. The signature is actually `byte[] WriteReadCheck(byte[] input, byte[] prefix, string id)`, which agrees with my explanation that it checks a prefix. The id in the sig is in this call the string "0x02" and is only used for debugging. For some reason mine returned the last byte of the prefix as `0x39` but it still worked. It seems it ignores the last element in the `ValidateReceivedData` method.

The format in the response is the following: (bytecnt, fmt, name)
```
 6 ?     prefix
20 ASCII typenumber
 4 ASCII revision
16 ASCII serial
 8 ASCII AppFirmVersion
 8 ASCII BootFirmVersion
 1 bool  RunningFirmware (0: AppFirmware, 1: BootFirmware)
 1 unused
```

I will now implement this, and it seems like other info is also simple to get.