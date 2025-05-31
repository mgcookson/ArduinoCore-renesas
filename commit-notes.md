3 changes
1) USB: Do not create USB DFU device on Uno R4 WiFi
- include screen shot of windows device manager showing the unknown device
2) USB: Use correct USB device class when CDC function is enabled
- include screen shot of windows sdk usbview utility showing warning
3) USB: Enable remote wake up when acting as an HID input device




# Motivation
When the Uno R4 Minima or Uno R4 WiFi device is connected to a host PC and acting as an HID device (ex: mouse or keyboard), if the host PC happens to be in sleep mode when the Uno device attempts to move the mouse | press a key the Uno device will hang until host PC exits sleep mode. The desired behavior is for the device to wake the host PC so it can proceed with whatever mouse | keyboard related task it was attempting to execute. This desired behavior mimics the existing behavior of AVR based Arduino devices (ex: Leonardo).

This proposed change allows the Uno R4* devices to automatically wake the host PC in this scenario, requiring no changes to user sketch.

# Chosen Approach
The TinyUSB module has latent support for the USB remote wake feature when Uno R4* device is acting as a USB device. This proposed change enables the remote wake feature on the USB device descriptor and adds code to automatically wake the USB host device if required whenever an HID report is sent to it.

This approach appears to be consistent with the remote wake implemention in all example apps included in the tinyusb project itself. For example: https://github.com/hathach/tinyusb/tree/master/examples/device/hid_boot_interface.

Note that the Portenta C33 has CDC, DFU, HID, *and* MSC USB device functions enabled in its variant of tusb_config.h. Since it only has a way to disable CDC and DFU at compile time, it will never be able to take advantage of this HID remote wakeup behavior. That stated, I'm not certain the Portenta C33 really supports HID device function to begin with since it is not specifically documented as such anywhere and I do not have access to one for test. If this proposed PR needs more work to properly handle Portenta C33, let me know.

To use:
* Compile user sketch with the existing DISABLE_USB_SERIAL directive. This is required since Arduino/tinyusb emulates a USB 2.0 device, which only supports remote wake on single function devices. Disabling the USB Serial feature of the USB port converts it from a composite device (with both CDC and HID interfaces) to a single function device (HID interface only) device.
* Invoke the desired Mouse | Keyboard input function (e.g. Mouse.move(...) | Keyboard.write(...) | etc.). If the host happens to be in sleep mode *and* has armed the device for remote wake prior to entering sleep mode, the device will wake the host automatically.

# Notes
* It could be argued that the user sketch should be responsible for the decision to wake the host. This would require publicly exposing the USB API to do so and updating all related documentation. My assumption, however, is that the Arduino community benefits from a reasonable level of behavioral compatibility between different boards, and I observe that the automatic remote wake approach proposed in this PR is behaviorally identical to that already implemented within the current Arduino AVR core.
  * AVR core enabled support for single function (HID only) mode via this PR: https://github.com/arduino/ArduinoCore-avr/pull/383.
  * AVR core enabled automatic remote wake via this commit: https://github.com/arduino/ArduinoCore-avr/commit/7874386
  * To use the remote wake feature in AVR core, you must compile user sketch with the CDC_DISABLED directive. This directive is not defined within the Renesas core; the DISABLE_USB_SERIAL directive takes its place.
* The DISABLE_USB_SERIAL directive will have following two side effects:
  * All Serial.* functions will no longer work.
  * You will need to double press the device's reset button to enable uploading a new application via its USB port.

# Questions
1. If approved at all, should this change be held until the next feature release? It *is* a change in behavior (albiet an opt-in one), but could also arguably be considered a fix to reinstate behavior exhibited by older Arduino devices. 
2. Should the DISABLE_USB_SERIAL directive be specifically documented somewhere?
3. Instead of or in addition to above, should I include an example application in this PR demonstrating use of the directive? Since the behavior does not require any application level code other than compiling with the directive defined, it doesn't seem all that useful.


# Motivation
I observed that when the Uno R4 WiFi was connected to my Windows host, the Windows Device Manager app showed an error that the device's DFU port was missing a driver. Since the Uno R4 WiFi uploads via serial, not DFU, it seemed that the device should not be presenting a DFU port to the host at all.

# Chosen Approach
Modify the 
# Notes
1. The PID for the Uno R4 WiFi isn't present in DFU device driver installation package's renesas.inf file here:
    C:\Users\mikec\AppData\Local\Arduino15\packages\arduino\hardware\renesas_uno\1.4.1\drivers
This makes sense since the device handles uploads via Serial instead of DFU, and further supports the approach chosen in this proposed change.



  https://www.usb.org/sites/default/files/iadclasscode_r10.pdf

  https://www.usb.org/defined-class-codes

  C:\Program Files (x86)\Windows Kits\10\Debuggers\x64

  
todo:
- try moving HID to the first device and see if that allows it to work as remote wakeup even with other
  devices defined.
  - RESULT: did not work
- right now, the feature is enabled via the DISABLE_USB_SERIAL directive. However, that directive does not
  disable the MSD device, which means that on the Portenta C33 board the HID remote wake feature still won't work.
  Do we need to create another directive named something like ENABLE_USB_HID_ONLY, and drive the behavior through
  that directive instead?
  - Portenta C33 is the only board variant that defines CFG_TUD_MSC=1.
  - Does it even support HID? I cannot see anything in its documentation stating that it does.
- check all other renesas board docs to see which explicitly support HID.
  - none other say they do. Probably worth mentioning in pull request that I'm not sure that all board variants
    have their CFG_TUD_* directives properly defined.

issue
- wifi running timed wake sketch compiled without DISABLE_USB
  - tried uploading new sketch via com5 (the one and only serial port exposed by device)
  - bossac reset the device and it entered upload mode, but on com3. bossac then failed. USBView showed that USB device was published directly by tinyusb (running on the ESP32).
  - ran upload again on com3 and it worked
- looks like this might be expected behavior. See here:
  - https://docs.arduino.cc/tutorials/uno-r4-wifi/usb-hid/?_gl=1*1s3vbtd*_up*MQ..*_ga*MTcwNTI3ODg0Mi4xNzQzMDIxMzU4*_ga_NEXN8H46L5*MTc0MzAyMTM1Ni4xLjEuMTc0MzAyMTQ0MS4wLjAuMTI5OTc0OTUzMA..#sketch-upload-interference

DFU
- in boards.txt, all boards use dfu-util except wifi, which uses bossac
- the renesas.cfg file within the drivers folder does not contain entry to install dfu driver for the wifi board,
  implying it does not expect it to be used.
- https://github.com/arduino/arduino-renesas-bootloader/blob/main/Makefile.wifi
  - CFLAGS are set to enable BOSSA loader
  - all other makefiles in same project enable DFU loader instead
- the tusb_config.h file in all board variant folders has '#define CFG_TUD_DFU_RUNTIME 1'. Given above two points, seems like the wifi variant should define it as 0 instead, and the USB setup code should respect that by not creating the DFU interface.