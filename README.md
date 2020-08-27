	
![gsoc_logo](images/gsoc_logo.png)

# Overview

The goal is to enhance the Linux kernel debugging support on imx8 and imx8m platforms by capturing traces when the DSP panics.

Before working on this, I had no prior experience with open source projects and kernel development. It has been a great learning experience for me, and I would like to thank [Daniel Baluta](https://github.com/dbaluta) for his awesome guidance.

### The patches that make up the project

- [1. This allows imx platforms to use functions from the common core to acces information about the stack and registers.](https://github.com/thesofproject/linux/pull/2322/commits/58f860ba61ee2c5fe947f7f9acad4cda34c2f757)
- [2. This adds a common memory region in the mailbox so that the firmware can pass panic messages to the kernel.](https://github.com/thesofproject/linux/pull/2341/commits/a316ce5fdd7a304e59555d5faacc8ae51e8e112f)
- [3. This tells the firmware to send the panic message to the kernel and generate an interrupt.](https://github.com/thesofproject/sof/pull/3282/commits/d334d9890e28439828251aac16e3ba17e2a7a583)
- [4. This adds the functions that will gather the information about registers, stack , file name and line number and then will print it to the console.](https://github.com/thesofproject/linux/pull/2348/commits/39cdee64070e063c6d8802362ace3c7b54b370b1)

# How was it done?

## Hardware
The first step was to get the hardware ready and set up the development environment.
I was working on an [i.MX8QM  board](https://www.nxp.com/design/development-boards/i-mx-evaluation-and-development-boards/i-mx-8quadmax-multisensory-enablement-kit-mek:MCIMX8QM-CPU). The dev environment was set-up via u-boot, with the rootfs on the host machine and parted via an NFS server. The kernel image is cross compiled on the host machine and passed via a TFTP server.

### [Setting up U-boot with NFS and TFTP](https://github.com/IulianOlaru249/GSoC2020-Summary/blob/master/hardware-setup-tutorial.md)

## Software
We need to keep track of the program flow from two points of view: the Firmware and the Kernel.

### Firmware
In the case of an oops on the firmware side, the exception handler will be called. Further, it will call the panic dump function which will collect information about the stack, registers and filename and will place it in the shared memory. The platform panic function, shown below, is called. The panic message is added in the **debug box** region so the kernel can pick it up. The kernel is then notified via an **interrup**.
```c
static inline void platform_panic(uint32_t p)
{
	/* Store the error code in the debug box so the
	 * application processor can pick it up. Takes up 4 bytes
	 * from the debug box.
	 */
	mailbox_sw_reg_write(SRAM_REG_FW_STATUS, p);

	/* Notify application processor */
	imx_mu_xcr_rmw(IMX_MU_xCR_GIRn(1), 0);
}
```

### Kernel
The kernel will receive a message in the **debug box**. It will check if it is a panic code, and act accordingly.
```c
static void imx8_dsp_handle_request(struct imx_dsp_ipc *ipc)
{
	struct imx8_priv *priv = imx_dsp_get_data(ipc);
	u32 p; /* panic code */

	/* Read the message from the debug box. */
	sof_mailbox_read(priv->sdev, priv->sdev->debug_box.offset + 4, &p, sizeof(p));

	/* Check to see if the message is a panic code (0x0dead***) */
	if ((p & SOF_IPC_PANIC_MAGIC_MASK) == SOF_IPC_PANIC_MAGIC)
		snd_sof_dsp_panic(priv->sdev, p);
	else
		snd_sof_ipc_msgs_rx(priv->sdev);
}
```

If it is a panic message, snd_sof_dsp_panic will be called, which, in turn, will call snd_sof_dsp_dbg_dump. This is just an alias for the imx_dump below.
```c
/**
 * imx8_dump() - This function is called when a panic message is
 * received from the firmware.
 */
void imx8_dump(struct snd_sof_dev *sdev, u32 flags)
{
	struct sof_ipc_dsp_oops_xtensa xoops;
	struct sof_ipc_panic_info panic_info;
	u32 stack[IMX8_STACK_DUMP_SIZE];
	u32 status;

	/* Get information about the panic status from the debug box area.
	 * Compute the trace point based on the status.
	 */
	sof_mailbox_read(sdev, sdev->debug_box.offset + 0x4, &status, 4);

	/* Get information about the registers, the filename and line
	 * number and the stack.
	 */
	imx8_get_registers(sdev, &xoops, &panic_info, stack,
			   IMX8_STACK_DUMP_SIZE);

	/* Print the information to the console */
	snd_sof_get_status(sdev, status, status, &xoops, &panic_info, stack,
			   IMX8_STACK_DUMP_SIZE);
}
```

Information about registers will be gathered by imx8_get_registers.
```c
/**
 * imx8_get_registers() - This function is called in case of DSP oops
 * in order to gather information about the registers, filename and
 * linenumber and stack.
 * @sdev: SOF device
 * @xoops: Stores information about registers.
 * @panic_info: Stores information about filename and line number.
 * @stack: Stores the stack dump.
 * @stack_words: Size of the stack dump.
 */
void imx8_get_registers(struct snd_sof_dev *sdev,
			struct sof_ipc_dsp_oops_xtensa *xoops,
			struct sof_ipc_panic_info *panic_info,
			u32 *stack, size_t stack_words)
{
	u32 offset = sdev->dsp_oops_offset;

	/* first read registers */
	sof_mailbox_read(sdev, offset, xoops, sizeof(*xoops));

	/* then get panic info */
	if (xoops->arch_hdr.totalsize > EXCEPT_MAX_HDR_SIZE) {
		dev_err(sdev->dev, "invalid header size 0x%x. FW oops is bogus\n",
			xoops->arch_hdr.totalsize);
		return;
	}
	offset += xoops->arch_hdr.totalsize;
	sof_mailbox_read(sdev, offset, panic_info, sizeof(*panic_info));

	/* then get the stack */
	offset += sizeof(*panic_info);
	sof_mailbox_read(sdev, offset, stack, stack_words * sizeof(u32));
}
```

And finally, all information will be printed to the console on the kernel side
```bash
root@imx8qxpmek:~# aplay -Dhw:0,0 -f S32_LE -c 2 -r 4800 /Metallica\ One\ \(Official\ Music\ Video\)\ \(online-audio-converter.com\).wav 
[   46.500179] sof-audio-of 556e8000.dsp: error : DSP panic!
[   46.505609] sof-audio-of 556e8000.dsp: error: runtime exception
[   46.511539] sof-audio-of 556e8000.dsp: error: trace point 0dead006
[   46.517722] sof-audio-of 556e8000.dsp: error: panic at src/arch/xtensa/init.c:48
[   46.525126] sof-audio-of 556e8000.dsp: error: DSP Firmware Oops
[   46.531051] sof-audio-of 556e8000.dsp: EXCCAUSE 0x0000003f EXCVADDR 0x00000000 PS       0x00060125 SAR     0x15222604
[   46.541663] sof-audio-of 556e8000.dsp: EPC1     0x627400c3 EPC2     0x9241f43a EPC3     0x00000000 EPC4    0x00000000
[   46.552280] sof-audio-of 556e8000.dsp: EPC5     0x00000000 EPC6     0x200828fa EPC7     0xa4d4a0ae DEPC    0x00000000
[   46.562896] sof-audio-of 556e8000.dsp: EPS2     0x00060020 EPS3     0x00000000 EPS4     0x00000000 EPS5    0x00000000
[   46.573512] sof-audio-of 556e8000.dsp: EPS6     0x242636a0 EPS7     0x800a2407 INTENABL 0x660e066c INTERRU 0x00000408
[   46.584129] sof-audio-of 556e8000.dsp: stack dump from 0x9244f4f0
[   46.590235] sof-audio-of 556e8000.dsp: 0x9244f4f0: 00000000 00000000 00000000 00000000
[   46.598163] sof-audio-of 556e8000.dsp: 0x9244f4f4: 6574782f 2f61736e 74696e69 0000632e
[   46.606083] sof-audio-of 556e8000.dsp: 0x9244f4f8: 3e1bd700 45219c4c 1244be00 ffff8000
[   46.614006] sof-audio-of 556e8000.dsp: 0x9244f4fc: 00000001 00000000 1244be30 0dead006
[   46.621931] sof-audio-of 556e8000.dsp: 0x9244f500: 10e07190 ffff8000 1244be90 ffff8000
[   46.629852] sof-audio-of 556e8000.dsp: 0x9244f504: 100cde20 ffff8000 12108000 ffff8000
[   46.637775] sof-audio-of 556e8000.dsp: 0x9244f508: 1244c000 ffff8000 11bc0008 ffff8000
[   46.645698] sof-audio-of 556e8000.dsp: 0x9244f50c: 121092c8 ffff8000 1244bf30 00000000
[   48.517562] sof-audio-of 556e8000.dsp: error: firmware boot failure
[   48.523877] sof-audio-of 556e8000.dsp: error: runtime exception
[   48.529827] sof-audio-of 556e8000.dsp: error: trace point 0dead006
[   48.536046] sof-audio-of 556e8000.dsp: error: panic at src/arch/xtensa/init.c:48
[   48.543484] sof-audio-of 556e8000.dsp: error: DSP Firmware Oops
[   48.549434] sof-audio-of 556e8000.dsp: EXCCAUSE 0x0000003f EXCVADDR 0x00000000 PS       0x00060125 SAR     0x15222604
[   48.560085] sof-audio-of 556e8000.dsp: EPC1     0x627400c3 EPC2     0x9241f43a EPC3     0x00000000 EPC4    0x00000000
[   48.570733] sof-audio-of 556e8000.dsp: EPC5     0x00000000 EPC6     0x200828fa EPC7     0xa4d4a0ae DEPC    0x00000000
[   48.581592] sof-audio-of 556e8000.dsp: EPS2     0x00060020 EPS3     0x00000000 EPS4     0x00000000 EPS5    0x00000000
[   48.592257] sof-audio-of 556e8000.dsp: EPS6     0x242636a0 EPS7     0x800a2407 INTENABL 0x660e066c INTERRU 0x00000408
[   48.602915] sof-audio-of 556e8000.dsp: stack dump from 0x9244f4f0
[   48.609245] sof-audio-of 556e8000.dsp: 0x9244f4f0: 00000000 00000000 00000000 00000000
[   48.617202] sof-audio-of 556e8000.dsp: 0x9244f4f4: 6574782f 2f61736e 74696e69 0000632e
[   48.625148] sof-audio-of 556e8000.dsp: 0x9244f4f8: 3e1bd700 45219c4c 12efb680 ffff8000
[   48.633097] sof-audio-of 556e8000.dsp: 0x9244f4fc: 00000004 00000000 00000000 ffff8000
[   48.641037] sof-audio-of 556e8000.dsp: 0x9244f500: 091c4854 ffff8000 f79e2c10 ffff0008
[   48.648998] sof-audio-of 556e8000.dsp: 0x9244f504: 10873780 ffff8000 12efb710 ffff8000
[   48.656952] sof-audio-of 556e8000.dsp: 0x9244f508: f79e2810 ffff0008 12efb740 ffff8000
[   48.664902] sof-audio-of 556e8000.dsp: 0x9244f50c: 10873544 ffff8000 f79e2c10 ffff0008
[   48.672838] sof-audio-of 556e8000.dsp: error: failed to boot DSP firmware after resume -5
Warning: format is changed to S16_LE[   48.687923] sof-audio-of 556e8000.dsp: error: ipc error for 0x60010000 size 20

Playing WAVE '/Metallica One (Official Music Video) (online-aud[   48.696563] sof-audio-of 556e8000.dsp: error: hw params ipc failed for stream 0
io-converter.com).wav' : Signed 16 bit Little Endian, Rate 44100 [   48.709514] sof-audio-of 556e8000.dsp: ASoC: 556e8000.dsp hw params failed: -19
Hz, Stereo
Warning: rate is not accurate (requested = 44100Hz, g[   48.722485]  Port0: ASoC: hw_params FE failed -19
ot = 48000Hz)
         please, try the plug plugin 
aplay: set_params:1403: Unable to install hw params:
ACCESS:  RW_INTERLEAVED
FORMAT:  S16_LE
SUBFORMAT:  STD
SAMPLE_BITS: 16
FRAME_BITS: 32
CHANNELS: 2
RATE: 48000
PERIOD_TIME: 85000
PERIOD_SIZE: 4080
PERIOD_BYTES: 16320
PERIODS: (4 5)
BUFFER_TIME: 341000
BUFFER_SIZE: 16368
BUFFER_BYTES: 65472
TICK_TIME: 0

```

# What I learned !

While working on this project, I had the opportunity to learn from amazing people and I tried to take full advantage of that. I learned:
- how set up the hardware environment
- how the kernel works underneath the hood :)
- how to cross compile the Linux Kernel
- how to make my way trough the Linux Kernel source code
- how the Sound Open Firmware works
- how a firmware and an application processor (such as the kernel) communicate
- how to work better with git
- how to make contributions to an open source project

And those are just the highlights. Google Summer of Code has been an amazing journey and I am grateful for the opportunity. I came a long way this summer and I intend to keep in touch with my mentor and the community since I stil have a lot to learn from them.

# What is next?

The next step is to work on integrating a gdb stub, so that the developer can do live debugging whenever there is something wrong with the firmware.
