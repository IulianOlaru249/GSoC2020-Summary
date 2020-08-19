## Goal
The goal is to enhance the Linux kernel debugging support on imx8 and imx8m platforms by capturing traces when the DSP panics.

Before working on thisWorking on this, I had no prior experience with open source projects and kernel development. It has been a great learning experience for me, and I would like to thank [Daniel Baluta](https://github.com/dbaluta) for his awesome guidance.

### The patches that make up the project

- [1. This allows imx platforms to use functions from the common core to acces information about the stack and registers.](https://github.com/thesofproject/linux/pull/2322)
- [2. This adds a common memory region in the mailbox so that the firmware can pass panic messages to the kernel.](https://github.com/thesofproject/linux/pull/2341)
- [3. This tells the firmware to send the panic message to the kernel and generate an interrupt.](https://github.com/thesofproject/sof/pull/3282)
- [4. This adds the functions that will gather the information about registers, stack , file name and line number and then will print it to the console.](https://github.com/thesofproject/linux/pull/2348)

## How was it done?

The first step was to get the hardware ready and set up the development envirnment.
I was working on an [I.MX8qm board](https://www.nxp.com/design/development-boards/i-mx-evaluation-and-development-boards/i-mx-8quadmax-multisensory-enablement-kit-mek:MCIMX8QM-CPU). The dev environment was set-up via u-boot, with the rootfs on the host machine and parted via an NFS server. The kernel image is cross compiled on the host machine and passed via a TFTP server.

### Setting up U-boot with NFS an TFTP

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block
```
# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)


For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/IulianOlaru249/GsOC2020-Summary/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
