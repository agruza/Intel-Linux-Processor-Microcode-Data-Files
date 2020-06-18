# Intel Processor Microcode Package for Linux

## About

CPU microcode is a mechanism to correct certain errata in existing systems.
In addition, CPU microcode is responsible for starting SGX enclave, it pays
a vital role in implementing complex behaviors (like assists), an more.
When it comes to Microcode Updates (MCU), the preferred method to apply
MCU is using the system BIOS, but for a subset of Intel's processors
this can be done at runtime using the operating system. Intel Microcode
Package shared here contains those processors that support OS loading
of microcode updates.

## Why update the microcode?
Microcode Updates prevent from potential security vulnerabilities in CPUs.
Installing MCU with security and stability (voltage/thermal-related) updates
ensures user doesn't experience unpredictable program behavior, such as hangs,
bogus crashes, reboots, memory corruption, incorrect calculations, etc. that
can be difficult to track down. To learn more about udating microcode, see [Microcode Update Guidance](https://software.intel.com/security-software-guidance/insights/microcode-update-guidance).

## Loading microcode updates

The target user for this package are OS vendors such as Linux distributions
for inclusion in their OS releases. Intel recommends getting the microcode
using the OS vendor update mechanism. A good starting point is [OS and Software Vendor](https://software.intel.com/security-software-guidance/insights/guidance-system-administrators-mitigate-transient-execution-side-channel-issues). Expert users can of course update their microcode directly outside the OS vendor mechanism. However, this method is complex and thus could be error prone.

Microcode is best loaded from the BIOS. Certain microcode must only be applied
from the BIOS. Such processor microcode updates are never packaged in this
package since they are not appropriate for OS distribution. An OEM can receive
microcode packages that might be a superset of what is contained in this
package.

OS vendors may choose to also update microcode that kernel can consume for early
loading. For e.g. Linux can update processor microcode very early in the kernel
boot sequence. In situations when the BIOS update isn't available, early loading
is the next best alternative to updating processor microcode. **Microcode states
are reset on a power reset, hence its required to be updated everytime during
boot process.**

## Recommendation

Loading microcode using the initrd method is recommended so that the microcode
is loaded at the earliest time for best coverage. Systems that cannot tolerate
downtime may use the late-load method to update a running system without a
reboot.

## About Processor Signature, Family, Model, Stepping and Platform ID

Processor signature is a number identifying the model and version of a
Intel processor. It can be obtained using the *CPUID instruction*, and can
also be obtained via the command *lscpu* or from the content of */proc/cpuinfo*.
It's usually presented as 3 fields: Family, Model and Stepping.
```
**File Name: 06-9e-0b**

| Family-Model-Stepping | Platform ID | Revision ID | Date       | Processor Signature | Extended Signature |
| :-------------------- | :---------- | :-----------| :--------- | :------------------ | :----------------- |
|        06-9e-0b       |   00000002  |   000000ca  | 2020-06-05 |       000906eb      |                    |

```

The width of Family/Model/Stepping is 12/8/4bit, but when arranged in the
32bit processor signature raw data is in a form of 0FFM0FMS, hexadecimal.
e.g. if a processor signature is 0x000906eb, it means
Family=0x006, Model=0x9e and Stepping=0xb

A processor product can be implemented for multiple types of platforms,
So in MSR(17H), Intel processors have a 3bit Platform ID field,
that can specify a platform type from at most 8 types.
A microcode file for a specified processor model can support multiple
platforms, so the Platform ID of a microcode is a 8bit mask, each set
bit indicates a platform type that it supports. One can find the
platform ID on Linux using rdmsr from [msr-tools.](https://github.com/intel/msr-tools)

## Microcode update instructions

[intel-ucode](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/tree/master/intel-ucode) direcetory contains binary microcode files named in
`family-model-stepping` pattern. The file is supported in most modern Linux
distributions. It's generally located in the /lib/firmware directory,
and can be updated through the microcode reload interface (follow late-load update instructions below).

### Early-load update
To update early loading initrd, consult your distribution on how to package
microcode files for early loading. Some distros use `update-initramfs` or `dracut`.
As recommended above, please use the OS vendors recommended method to ensure
microcode file is updated for early loading before attempting the late-load
procedure below.

### Late-load update
To update the intel-ucode package to the system, one need:
1. Ensure the existence of `/sys/devices/system/cpu/microcode/reload`
2. Download the latest microcode firmware</br> `$ git clone https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files.git`
or</br> `$ wget https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/archive/master.zip`
3. Copy `intel-ucode` directory to `/lib/firmware`, overwrite the files in
/lib/firmware/intel-ucode/
4. Write the reload interface to 1 to reload the microcode files, e.g.</br>
  `$ echo 1 > /sys/devices/system/cpu/microcode/reload`</br>
  Microcode updates will be applied automatically without rebooting the system.
5. Update an existing initramfs so that next time it get loaded via kernel:</br>
`$ sudo update-initramfs -u`</br>
`$ sudo reboot`
6. Verify the microcode got updated on boot or reloaded by echo command:</br>
`$ dmesg | grep microcode` or</br>
`$ cat /proc/cpuinfo | grep microcode | sort | uniq`

If you are using the OS vendor method to update microcode, the above steps may
have been done automatically during the update process.


[intel-ucode-with-caveats](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/tree/master/intel-ucode-with-caveats) directory holds microcode that might need special handling.
BDX-ML microcode is provided in directory, because it need special commits in
the Linux kernel, otherwise, updating it might result in unexpected system
behavior.
OS vendors must ensure that the late loader patches (provided in
linux-kernel-patches\) are included in the distribution before packaging the
BDX-ML microcode for late-loading.


[linux-kernel-patches](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/tree/master/linux-kernel-patches) directory consists of kernel patches that fix verious issues related to microcode update.

## Notes

* You can only update to a higher version of the microcode (downgrade not possible with provided instructions)
* To calculate Family-Model-Stepping use Linux command:</br>
`$ printf "%x\n" <number_to_convert_to_hex>`
* There are multiple ways to check microcode version number BEFORE update. After cloning this Intel Microcode repo run:
  - `$ iucode_tool -l intel-ucode | grep -wF sig` ([iucode_tool](https://gitlab.com/iucode-tool/iucode-tool/-/wikis/home) package is required)
  - `$ od -t x4 <Family-Model-Stepping>` will read the first 16 bytes of the microcode binary header specified in \<Family\-Model\-Stepping\>. Third block is your microcode version. For example:
`$ od -t x4 06-55-04`</br>
`0000000 00000001 *02000065* 09052019 00050654`

## License

See the [license](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/blob/master/license) file for details.

## Security Policy

See the [security.md](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/blob/master/security.md) file for details.

## Release Note

See the [releasenote](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/blob/master/releasenote) file for details.
