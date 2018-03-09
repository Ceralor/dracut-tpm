# dracut-tpm
This project provides a *simple* module for dracut to allow reading keys from TPM 1.2 modules to unlock LUKS devices at boot time.

# Requirements
This project uses **ncat** to communicate with the systemd-ask-password socket; this program is available in CentOS' nmap-ncat package.
One of the two options for reading from the NVRAM must be chosen:
1. Use **tcsd** and **tpm_nvread** - this requires the *trousers* and *tpm-tools* packages in CentOS
2. Use the standalone program **nv_readvalue** - this requires building **nv_readvalue** from [this repository](http://github.com/gastamper/tpm-luks)

Why choose one over the other?
**tcsd** and **tpm_nvread** are included in the base repositories for CentOS, making this path somewhat more straightforward.  As a downside, it is more complicated "under the hood" so for those who prefer simplicity and a slim initramfs, this may not be the ideal options.  **nv_readvalue** however can be built as a standalone program, making this single program the only dependency for the dracut module.

It is ultimately a matter of preference with little practical value.  For most users, option 1 is sufficient.

# Installation
1. Create and store keys in your preferred NVRAM slot (see below)
2. Clone this repository
3. Run install.sh; this performs the following steps:
   1. Ensures that the necessary files, either **tcsd** and **tpm_nvread** or **nv_readvalue** are available on your system.
   2. Ensures that the TPM module is owned, active and enabled.
   3. If not using default values, updates the module scripts to use the user-specified NVRAM index and size.
   4. Creates a directory, /usr/lib/dracut/modules.d/50dracuttpm
   5. Copies the dracut module files to the above directory: *module-setup.sh* and *nv-hook.sh*
4. Reboot system and confirm automatic unlocking works.

# Storing keys in NVRAM
Keys are stored in NVRAM by using the **tpm_nvwrite** command, part of the tpm-tools package.  The steps for performing this process are as follows; note that this assumes that you have already taken ownership (initialized and taken control) of the TPM module using the **tpm_takeownership** command.
```sh
# Create and a 1MB RAMFS to hold our data
mkdir -p /mnt/ramfs
mount -t tpmfs -o size=1m tmpfs /mnt/ramfs
# Generate 256 bytes of random data to serve as our key
dd if=/dev/random of=/mnt/ramfs/key bs=1 count=256
# Define a new NVRAM area at the specified index, of the specified size
# See 'man tpm_nvdefine' for permissions explanation
tpm_nvdefine -i 1 -s 256 -p "OWNERWRITE|READ_STCLEAR" -o <owner_password> [-z <PCR1> -z <PCR2> ... n]
# Write the data to index 1, size 256
tpm_nvwrite -i 1 -s 256 -f /mnt/ramfs/key -z -p
```

# tpm_nvdefine explained
The **tpm_nvdefine** command is used to not only define the area within the NVRAM in which to store the key, but also to assign a specific set of PCRs (platform configuration registers) to which the area should be bound; in the event that these PCRs change, the NVRAM area will be inaccessible.  This prevents, for example, removing the device from one machine and accessing it from another, or accessing it using a custom kernel or bootloader.

The PCR table is as follows for TPM 1.2:

| PCR Number | Allocation       |
| :---------- | ---------------- |
|0           | BIOS             |
|1         | BIOS configuration |
|2         | Option ROMs |
|3         | Option ROM configuration |
|4         | MBR |
|5         | MBR configuration |
|6         | State transition/wake events |
|7         | Platform manufacturer specification measurement |
|8-15         | Static operating system |
|16         | Debug |
|23         | Application support |

While recommending a specific set of PCRs as 'optimal' is outside of scope for this project, typicall 0-5 would provide a reasonable starting point.  It is worth noting that an NVRAM area can be bound to no, one, some, or all PCRs depending on preference.

# The READ_STCLEAR flag
The *READ_STCLEAR* flag may be useful when defining an NVRAM area since it effectively "locks" the NVRAM area from further reading until the next reboot.  This flag may be triggered by issuing a read of size zero to a flagged index, f.e. `tpm_nvread -i 1 -s 0`.

# Using nv_readvalue
If you wish to use nv_readvalue, follow the below instructions:
1. Clone http://github.com/gastamper/tpm-luks
2. Build the TPM-LUKS project:
   1. autreconf -ivf
   2. configure
   3. make
3. Copy nv_readvalue to /usr/bin: `cp swtpm-utils/nv_readvalue /usr/bin`

# Considerations
**tpm_nvdefine** uses GNU _GETPASSWD_ to prompt for passwords if using the --pwdo option (prompt for non-commandline input of owner password), which always attempts to read input from the terminal device rather than stdin.  As a result, input redirection (storing the password in a file in ramfs) is problematic.  If you are interested in this functionality, please submit an issue and I will see about adding it to **tpm_nvdefine**.

# Future work
[ ] Add options to **tpm_nvdefine** and **tpm_nvwrite** to read password from stdin or a designated file.

# Acknowledgements
Special thanks to [Kent Yoder](https://github.com/shpedoikal) for providing the original TPM-LUKS framework for **nv_readvalue** and the other TPM-related commands, and [Nathaniel McCallum](https://npmccallum.gitlab.io/about/) for his work on Clevis, whose dracut hooks provide the basis for this project.
