<image src="images/gsoc_logo.png">
# Overview

The goal is to enhance the Linux kernel debugging support on imx8 and imx8m platforms by capturing traces when the DSP panics.

Before working on thisWorking on this, I had no prior experience with open source projects and kernel development. It has been a great learning experience for me, and I would like to thank [Daniel Baluta](https://github.com/dbaluta) for his awesome guidance.

### The patches that make up the project

- [1. This allows imx platforms to use functions from the common core to acces information about the stack and registers.](https://github.com/thesofproject/linux/pull/2322/commits/58f860ba61ee2c5fe947f7f9acad4cda34c2f757)
- [2. This adds a common memory region in the mailbox so that the firmware can pass panic messages to the kernel.](https://github.com/thesofproject/linux/pull/2341/commits/a316ce5fdd7a304e59555d5faacc8ae51e8e112f)
- [3. This tells the firmware to send the panic message to the kernel and generate an interrupt.](https://github.com/thesofproject/sof/pull/3282/commits/d334d9890e28439828251aac16e3ba17e2a7a583)
- [4. This adds the functions that will gather the information about registers, stack , file name and line number and then will print it to the console.](https://github.com/thesofproject/linux/pull/2348/commits/39cdee64070e063c6d8802362ace3c7b54b370b1)

# How was it done?

## Hardware
The first step was to get the hardware ready and set up the development envirnment.
I was working on an [I.MX8qm board](https://www.nxp.com/design/development-boards/i-mx-evaluation-and-development-boards/i-mx-8quadmax-multisensory-enablement-kit-mek:MCIMX8QM-CPU). The dev environment was set-up via u-boot, with the rootfs on the host machine and parted via an NFS server. The kernel image is cross compiled on the host machine and passed via a TFTP server.

### [Setting up U-boot with NFS and TFTP](https://github.com/IulianOlaru249/GSoC2020-Summary/blob/master/hardware-setup-tutorial.md)

## Software
We need to keep track of the program flow from two points of view: the Firmware and the Kernel.

### Firmware
In the case of an oops on the firmware side, the exception handler will be called. Further, it will call the panic dump function which will collect information aboutstack registers and filename and will place it in the shared memory. The platform panic function, shown below, is called. The panic message is added in the **debug box** region so the kernel can pick it up. The kernel in then notified via an **interruption**.
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
```markdown
```
