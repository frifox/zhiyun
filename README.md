# Zhiyun Gimbal Bluetooth Research

I have Zhiyun Crane Plus gimbal and Sony A6500 camera, but info below may apply to other gimbal & camera combos. Most of the data was obtained by watching bluetooth traffic from Zhiyun's ZY Play app using Apple's PacketLogger.

# Bluetooth Packets
Gimbal is controlled by sending it bluetooth packets. All commands of interest were sent to `D44BC439-ABFD-45A2-B575-925416129600` characteristic. Packets started with 06 byte, followed by 2-byte command, followed by 2-byte value, and ended with 2-byte CRC16-CCITT checksum. For example:

https://play.golang.org/p/F3CI8iqkd6l

```
package main

import (
  "fmt"
  "bytes"
  "encoding/binary"
  "github.com/howeyc/crc16"
)

func main() {
  packet := new(bytes.Buffer)

  packet.Write([]byte{0x6})       // packet prefix
  packet.Write([]byte{0x1, 0x27}) // command
  packet.Write([]byte{0x0, 0x0})  // value

  // append packet's CRC
  crc := crc16.Checksum(packet.Bytes(), crc16.CCITTFalseTable)
  binary.Write(packet, binary.BigEndian, crc)

  // compiled packet
  fmt.Printf("%X\n", packet.Bytes())
}
```
All packet examples in this readme will use `XXXX` in place of actual CRC

# Switch Gimbal Modes

Gimbal modes can be switched with 1 packet:
```
06 8127 0000 XXXX // control Y axis, XZ are locked
06 8127 0001 XXXX // control XY axis, Z is locked
06 8127 0002 XXXX // control Z axis, XY are locked
06 8127 0003 XXXX // no control, XYZ are locked
```

# Toggle Motors
Enabling and disabling motors require 2 packets each, first to 0107 register and second to 8107 register.
```
// disable motors
06 0107 0000 XXXX
06 8107 0000 XXXX

// enable motors
06 0107 0000 XXXX 
06 8107 0001 XXXX
```

# Control XYZ motion
ZY Play moves gimbal around by sending it 3 packets. Motion will be short, so ZY Play sends these 3 packets every 200ms to keep motion continuous.

```
06 1001 YYYY XXXX // Y axis
06 1002 YYYY XXXX // Z axis
06 1003 YYYY XXXX // X axis
```

YYYY above are values between 0 and 4095 (0x0 and 0xFFF)
* 0 to 2047 will go one direction
* 2048 will keep axis not moving
* 2049 to 4095 will go the other direction

The further you're from 2048, the faster gimbal will rotate. For example:
```
// turn left at full speed
06 1001 0800 XXXX
06 1002 0800 XXXX
06 1003 0000 XXXX // X axis

// turn right at full speed
06 1001 0800 XXXX
06 1002 0800 XXXX
06 1003 0FFF XXXX // X axis

// go up at 50% speed (2048 + 2048*0.5 = 3072, 0xC00)
06 1001 0C00 XXXX // Y axis
06 1002 0800 XXXX
06 1003 0800 XXXX

// go down at 10% speed (2048 - 2048*0.1 = 1843, 0x733)
06 1001 0733 XXXX // Y axis
06 1002 0800 XXXX
06 1003 0800 XXXX 

// go up at 50% and right at 100%
06 1001 0C00 XXXX // Y axis
06 1002 0800 XXXX
06 1003 0FFF XXXX // X axis
```

# Control Camera
Connect your camera to gimbal using Zhiyun's included Multi-to-Micro USB cable. Tested with Sony A6500. If you have a different camera you can still try it, let me know how it goes.
```
06 C120 0036 XXXX // start/stop recording
06 C120 003D XXXX // take a photo

06 C120 0007 XXXX // zoom out
06 C120 0008 XXXX // zoom in
06 C120 0027 XXXX // stop zooming
```

# Selfie Mode
In ZY Play iOS app the selfie button doesn't work. In Zhiyun Assistant app it did work, but bluetooth traffic with it was extremely noisy. Below is what I got so far:
```
06 C121 0001 XXXX // rotate 180*
```
This seems to work nicely if you're in the lock-XYZ-controls (0x3) mode.
