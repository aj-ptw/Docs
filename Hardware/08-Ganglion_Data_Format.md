# Ganglion Data Format

This discussion of the OpenBCI data format only applies to the OpenBCI Ganglion. The Ganglion contains a Simblee microcontroller that can both be programmed through the Arduino IDE or over-the-air (OTA). The Simblee has an on-board radio module. The format of the Ganglion data as seen on the PC is defined by a combination of the Arduino code on the Ganglion board. So, if you don't like the data format defined here, feel free to change it! In general, and believe us, we tired, you can't send more then 100 BLE packets per second. For more info on the byte stream parsing, or for a working example, see the [NodeJS Ganglion Driver](https://github.com/OpenBCI/OpenBCI_NodeJS_Ganglion).

## Standard Bluetooth 4.n BLE Setup

OpenBCI Ganglion uses a standard Bluetooth 4.n (BLE) connection.

Allowing connection to any BLE compatible device!

All BLE devices have specific _Service_, _Receive_, _Send_, and _Disconnect_, the Simblee's are:
**Service** `fe84'`
**Receive** `2d30c082f39f4ce6923f3484ea480596`
**Send** `2d30c083f39f4ce6923f3484ea480596`
**Disconnect** `2d30c084f39f4ce6923f3484ea480596`

## Startup
###**Ganglion Board**

The Ganglion does go through a reset cycle when its BLE connection is opened. Because of this, it's *NOT* possible to connect/disconnect from the Ganglion mid stream.

## Initiating Binary Transfer

Once the Ganglion has initialized itself with a BLE connection it waits for commands. In other words, it sends no data until it is told to start sending data. To begin data transfer, transmit a single ASCII **b**. Once the **b** is received, continuous transfer of data in binary format will ensue. To turn off the fire hose, send an **s**.

## Binary Format

Each packet contains a Byte ID followed by 19-bytes of data for a grand total of 20-bytes. The Byte ID determines how to parse the packets 19-bytes of data. Because of the BLE 100Hz packet transmission restriction, we introduced a delta compression protocol. Grab a notebook, and get ready to have some fun! Here are details on the format.

You establish a connection to the Simblee on your Ganglion using the Simblee characteristics. Now you want the data!

The Ganglion takes 24-bit signed, MSB first, samples measurements. If we did not compress the data, we could only fit one sample on each BLE packet because 24-bits is 3-bytes times 4 samples for each channel equates to 12-bytes, which would lead to a max sample rate of 100Hz, remember upper limit of BLE is somewhere around 100 packets per second. Therefore we aimed to get two samples per packet, which gives a real time 200Hz streaming sample rate! Pretty cool, here is how we did it:
  
By default we use a 19-bit delta compression. If we want to use the on-board accelerometer, an **a** ASCII command has been sent, the Ganglion will automatically switch to 18-bit delta compression. 

All delta compressions start with a raw uncompressed packet with Byte Id of `0x00`, that's 12 bytes of uncompressed 24-bit signed integer samples.

**IMPORTANT** 18-bit and 19-bit compression store the signed bit in the LSB! Negative numbers have a `1` in the LSB and positive numbers always have `0` in the LSB. If we kept the signed bit in the MSB, we would loose half of our dynamic range (bad), by placing it in the LSB, we loose a nominal rounding error. 

Byte ID Decimal | Byte ID HEX | Data Type | Description
--------|--------|--------|--------
`0` | `00` | `24bit` | Raw uncompressed
`1`-`100` | `0x01`-`0x64` | `18bit` | 18-bit compression with Accelerometer 
`101`-`200` | `0x65`-`0xC8` | `19bit` | 19-bit compression without Accelerometer
`201` | `0xC9` | `impedance` | Impedance Channel 1
`202` | `0xCA` | `impedance` | Impedance Channel 2
`203` | `0xCB` | `impedance` | Impedance Channel 3
`204` | `0xCC` | `impedance` | Impedance Channel 4
`205` | `0xCD` | `impedance` | Impedance Channel Reference
`206` | `0xCE` | `multi` | Part of ASCII message
`207` | `0xCF` | `multiStop` | End of ASCII message

**24bit**

Raw uncompressed is always saved. If you get a Byte ID of `0`, you're going to want to save that.

**18bit**

Let's take a practical example by looking at the automated test used in the [Ganglion NodeJS driver](https://github.com/OpenBCI/OpenBCI_NodeJS_Ganglion/blob/master/test/openBCIGanglionSample-test.js) to approach explaining 18-bit delta compression strategy. 

    let buffer = new Buffer(
    [
      0b00000001, // 0
      0b00000000, // 1
      0b00000000, // 2
      0b00000000, // 3
      0b00000000, // 4
      0b00100000, // 5
      0b00000000, // 6
      0b00101000, // 7
      0b00000000, // 8
      0b00000100, // 9
      0b10000000, // 10
      0b00000000, // 11
      0b10111100, // 12
      0b00000000, // 13
      0b00000111, // 14
      0b00000000, // 15
      0b00101000, // 16
      0b11000000, // 17
      0b00001010, // 18
      0b00001110  // 19
    ]);
    let expectedValue = [[0, 2, 10, 4], [131074, 245760, 114698, 49162]];

The first compressed channel in sample one would be derived from the first two bytes plus the first two bits of the third byte:
Channel 1 - Sample 1 - `0b000000000000000000` - or `0` in decimal.

The second compressed channel in sample one would be derived from the last six bits of byte three, all of byte four, and the first four bits of byte five:
Channel 2 - Sample 1 - `0b000000000000000010` - or `2` in decimal.

The third compressed sample in sample two would be derived from the last four bits of byte 14, all of byte 15, and the first six bits of byte 16:
Channel 3 - Sample 2 - `0b011100000000001010` - or `114698` in decimal.

To get 10Hz 8-bit accelerometer data out of 18-bit packets you must modulus 10 the Byte ID: 
If (Byte ID % 10) is 1, 2, and 3 then pull the last byte to get X, Y, and Z, respectably.

In our example above, byte 19, the last byte, has a value of `14` or `0x0E` which would be stored as the X axis for this accelerometer sample.

The sample number for Sample 1 is `1` and the sample number for Sample 2 is `2`. If the Byte Id was `47`, Sample 1's sample number is `93` and Sample 2's sample number is `94`.

Now let's look at some negative values!

    let buffer = new Buffer(
    [
      0b00000001, // 0
      0b11111111, // 1
      0b11111111, // 2
      0b01111111, // 3
      0b11111111, // 4
      0b10111111, // 5
      0b11111111, // 6
      0b11100111, // 7
      0b11111111, // 8
      0b11110101, // 9
      0b00000000, // 10
      0b00000001, // 11
      0b01001111, // 12
      0b10001110, // 13
      0b00110000, // 14
      0b00000000, // 15
      0b00011111, // 16
      0b11110000, // 17
      0b00000001  // 18
    ]);
    let expectedValue = [[-3, -5, -7, -11], [-262139, -198429, -262137, -4095]];
    
Keep in mind we look to the LSB for the sign of the 18-bit number!
    
The first compressed channel in sample one would be derived from the first two bytes plus the first **two** bits of the third byte for a total of 19-bits:
Channel 1 - Sample 1 - `0b111111111111111101` - or `-3` in decimal.  

The second compressed channel in sample two would be derived from the last six bits of byte 12, all of byte 13 and first four bits from byte 14:
Channel 2 - Sample 2 - `0b001111100011100011` - or `-198429` in decimal. **The MSB is `0`, yet this is a negative number because of the `1` in the LSB**
 
**19Bit**

Once again, let's take a practical example by looking at the automated test used in the [Ganglion NodeJS driver](https://github.com/OpenBCI/OpenBCI_NodeJS_Ganglion/blob/master/test/openBCIGanglionSample-test.js) to approach explaining 19-bit delta compression strategy. 

    let buffer = new Buffer(
    [
      0b01100101, // 0
      0b00000000, // 1
      0b00000000, // 2
      0b00000000, // 3
      0b00000000, // 4
      0b00001000, // 5
      0b00000000, // 6
      0b00000101, // 7
      0b00000000, // 8
      0b00000000, // 9
      0b01001000, // 10
      0b00000000, // 11
      0b00001001, // 12
      0b11110000, // 13
      0b00000001, // 14
      0b10110000, // 15
      0b00000000, // 16
      0b00110000, // 17
      0b00000000, // 18
      0b00001000  // 19
    ]);
    let expectedValue = [[0, 2, 10, 4], [262148, 507910, 393222, 8]];

The first compressed channel in sample one would be derived from the first two bytes plus the first **three** bits of the third byte for a total of 19-bits:
Channel 1 - Sample 1 - `0b0000000000000000000` - or `0` in decimal.

The second compressed channel in sample two would be derived from the last bit in byte 12, all of bytes 13 and 14, and only the first two bits of byte 15:
Channel 2 - Sample 2 - `0b1111100000000000110` - or `507910` in decimal. **Note: the MSB is a `1` but this number is even because the LSB is `0`**

The sample number for Sample 1 is `1` and the sample number for Sample 2 is `2`. If the Byte Id was `104`, Sample 1's sample number is `7` and Sample 2's sample number is `8`.

Now let's look at some negative values!

    let buffer = new Buffer(
    [
      0b01100101, // 0
      0b11111111, // 1
      0b11111111, // 2
      0b10111111, // 3
      0b11111111, // 4
      0b11101111, // 5
      0b11111111, // 6
      0b11111100, // 7
      0b11111111, // 8
      0b11111111, // 9
      0b01011000, // 10
      0b00000000, // 11
      0b00001011, // 12
      0b00111110, // 13
      0b00111000, // 14
      0b11100000, // 15
      0b00000000, // 16
      0b00111111, // 17
      0b11110000, // 18
      0b00000001  // 19
    ]);
    let expectedValue = [[-3, -5, -7, -11], [-262139, -198429, -262137, -4095]];
    
Keep in mind we look to the LSB for the sign of the 19-bit number!
    
The first compressed channel in sample one would be derived from the first two bytes plus the first **three** bits of the third byte:
Channel 1 - Sample 1 - `0b1111111111111111101` - or `-3` in decimal.  

The forth compressed channel in sample two would be derived from the last three bits of byte 17 and all of bytes 18 and 19:
Channel 4 - Sample 2 - `0b001111100011100011` - or `-198429` in decimal. **The MSB is `0`, yet this is a negative number because of the `1` in the LSB**

**impedance**

Impedance values are sent with Byte IDs and are in ASCII format ending with a `Z`. Parse from byte 1 till you hit the `Z`. 

## 18-Bit Signed Data Values

For the compressed EEG data values, you will note that we are transferring the data as a 18-bit signed integer, which is a bit unusual. We are using this number format because it is the native format used by the A/D chip that is at the core of the Ganglion board. To convert this unusual number format into a more standard 32-bit signed integer, you can steal some ideas from the example NodeJS (aka, JavaScript) code:

    /**
     * Converts a special ganglion 18 bit compressed number
     *  The compressions uses the LSB, bit 1, as the signed bit, instead of using
     *  the MSB. Therefore you must not look to the MSB for a sign extension, one
     *  must look to the LSB, and the same rules applies, if it's a 1, then it's a
     *  negative and if it's 0 then it's a positive number.
     * @param threeByteBuffer {Buffer}
     *  A 3-byte buffer with only 18 bits of actual data.
     * @return {number} A signed integer.
     */
    function convert18bitAsInt32 (threeByteBuffer) {
      let prefix = 0;
    
      if (threeByteBuffer[2] & 0x01 > 0) {
        // console.log('\t\tNegative number')
        prefix = 0b11111111111111;
      }
    
      return (prefix << 18) | (threeByteBuffer[0] << 16) | (threeByteBuffer[1] << 8) | threeByteBuffer[2];
    }

## 19-Bit Signed Data Values

For the compressed EEG data values, you will note that we are transferring the data as a 19-bit signed integer, which is a bit unusual. We are using this number format because it is the native format used by the A/D chip that is at the core of the Ganglion board. To convert this unusual number format into a more standard 32-bit signed integer, you can steal some ideas from the example NodeJS (aka, JavaScript) code:

    /**
     * Converts a special ganglion 19 bit compressed number
     *  The compressions uses the LSB, bit 1, as the signed bit, instead of using
     *  the MSB. Therefore you must not look to the MSB for a sign extension, one
     *  must look to the LSB, and the same rules applies, if it's a 1, then it's a
     *  negative and if it's 0 then it's a positive number.
     * @param threeByteBuffer {Buffer}
     *  A 3-byte buffer with only 19 bits of actual data.
     * @return {number} A signed integer.
     */
    function convert19bitAsInt32 (threeByteBuffer) {
      let prefix = 0;
    
      if (threeByteBuffer[2] & 0x01 > 0) {
        // console.log('\t\tNegative number')
        prefix = 0b1111111111111;
      }
    
      return (prefix << 19) | (threeByteBuffer[0] << 16) | (threeByteBuffer[1] << 8) | threeByteBuffer[2];
    }

## Interpreting the EEG Data

Once you receive and parse and decompress the data packets, it is important to know how to interpret the data so that the EEG values are useful in a quantitative way. The critical piece of information is the scale factor.

For the scale factor, this is the multiplier that you use to convert the EEG values from “counts” (the int32 number that you parse from the binary stream) into scientific units like “volts”. Scale factor is set and baked into the hardware, therefore use a scale factor of:


	Scale Factor (Volts/count) = 1.2 Volts * 8388607.0 * 1.5 * 51.0;

This equation is from the MCP3912 data sheet, specifically it is from the text surrounding Table 7. This scale factor has also been confirmed experimentally using known calibration signals.

Accelerometer data must also be scaled before it can be correctly interpreted. The equation used to scale Accelerometer data is as follows (We assume 4Gs, so 2mG per digit):


	Accelerometer Scale Factor = 0.032;
