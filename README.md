# Early-TA
Detail about Early TA

# What is Early TA

Early TAs and Pseudo TAs are part of the OP-TEE binary so they are available immediately after boot. Early TAs run in user space, Pseudo TAs run in kernel space.
early TAs are virtually identical to the REE FS TAs, but instead of being loaded from the Normal World file system, they are linked into a special data section in the TEE core blob.

Therefore, they are available even before tee-supplicant and the REE’s filesystems have come up. Please find more details in the early TA commit. 
(Before Linux booting i.e you can communicate ate U-Boot level only by commands.)

Early TAs are user-mode Trusted Applications that are embedded at link time in the TEE binary. 
 
A Python script (scripts/ta_bin_to_c.py) takes care of converting the TAs into a C source file with the proper linker section attribute.
Script path: https://github.com/OP-TEE/optee_os/blob/master/scripts/bin_to_c.py

Previously ".rodata.early_ta" section used. In below commit:
https://github.com/OP-TEE/optee_os/commit/d0c636148b3a
A special read-only data section is used to store them (.rodata.early_ta).

But in later vesrion OP-TEE, ".scattered_array" section used for it:
https://github.com/OP-TEE/optee_os/commit/ed30b6c7493cff4c2d29064f75b037f5677ddb67

The feature is disabled by default. To enable it, the paths to the TA binaries have to be given in $(EARLY_TA_PATHS). They should be ELF files. Typical build steps:
```ruby
  $ make ... CFG_EARLY_TA=y ta_dev_kit                    # (1)
  $ # ... build the TAs ...                               # (2)
  $ make ... EARLY_TA_PATHS=path/to/<uuid>.stripped.elf   # (3)
```

Notes:
- Setting CFG_EARLY_TA=y during the first step (1) is not necessary, but it will avoid rebuilding libraries during the third step (3)
- CFG_EARLY_TA is automatically enabled when EARLY_TA_PATHS is non-empty in step (3)
- Several TAs may be given in $(EARLY_TA_PATHS) (3)

Early TAs are given a higher load priority than REE FS TAs, since they should be available even before tee-supplicant is ready.

For example you can specify the PATH to your TA file with macro EARLY_TA_PATH as below (if QEMU is used):
```ruby
[Example on QEMU platform]
$ make -j10
$ make -j10 EARLY_TA_PATHS=../out-br/build/optee_examples_ext-1.0/hello_world/ta/out/8aaaf200-2450-11e4-abe2-0002a5d5c51b.stripped.elf run
[On secure console]
I/TC: OP-TEE version: 3.15.0-dev (gcc version 10.2.1 20201103 (GNU Toolchain for the A-profile Architecture 10.2-2020.11 (arm-10.16))) #8 Thu Dec 23 15:55:20 UTC 2021 aarch64
.....
D/TC:0 0 early_ta_init:56 Early TA 8aaaf200-2450-11e4-abe2-0002a5d5c51b size 29691 (compressed, uncompressed 75492)
```

Note: Many services exported to TAs may need tee-supplicant, so early use is limited to a subset of the TEE Internal Core API (crypto...)

You can provide multiple Erly TA paths:
```ruby
#   $ make ... \
#     EARLY_TA_PATHS="path/to/8aaaf200-2450-11e4-abe2-0002a5d5c51b.stripped.elf \
#                     path/to/cb3e5ba0-adf1-11e0-998b-0002a5d5c51b.stripped.elf"
```

# How to load Early TA at U-Boot level using U-Boot commands.

In U-Boot, OP-TEE driver will be available in this path:
u-boot/drivers/tee/optee/
https://github.com/u-boot/u-boot/tree/a94ab561e2f49a80d8579930e840b810ab1a1330/drivers/tee/optee

The communication between OP-TEE and U-Boot explained here:
u-boot/doc/README.tee
https://github.com/u-boot/u-boot/blob/a94ab561e2f49a80d8579930e840b810ab1a1330/doc/README.tee

There is commands to test AVB and RPMB at U-Boot level:
https://github.com/u-boot/u-boot/blob/51aef405550e603ff702c034f0e2cd0f15bdf2bb/cmd/optee_rpmb.c

```ruby
U-Boot > optee_rpmb
Provides commands for testing secure storage on RPMB on OPTEE

U-Boot > optee_rpmb -h
optee_rpmb read_pvalue <name> <bytes> - read a persistent value <name>
optee_rpmb write_pvalue <name> <value> - write a persistent value <name>

Example:

U-boot > optee_rpmb write_pvalue test 1234
D/TC:0   tee_entry_exchange_capabilities:100 Asynchronous notifications are disabled
D/TC:0   tee_entry_exchange_capabilities:109 Dynamic shared memory is enabled
D/TC:0 0 core_mmu_xlat_table_alloc:513 xlat tables used 3 / 8
D/TC:? 0 tee_ta_init_pseudo_ta_session:296 Lookup pseudo TA 023f8f1a-292a-432b-8fc4-de8471358067
D/TC:? 0 ldelf_load_ldelf:95 ldelf load address 0x40006000
D/LD:  ldelf:134 Loading TS 023f8f1a-292a-432b-8fc4-de8471358067
.....
D/TC:0   tee_entry_exchange_capabilities:100 Asynchronous notifications are disabled
D/TC:0   tee_entry_exchange_capabilities:109 Dynamic shared memory is enabled
D/TC:0 0 core_mmu_xlat_table_alloc:513 xlat tables used 3 / 8
D/TC:? 0 tee_ta_init_pseudo_ta_session:296 Lookup pseudo TA 023f8f1a-292a-432b-8fc4-de8471358067
D/TC:? 0 ldelf_load_ldelf:95 ldelf load address 0x40006000
D/LD:  ldelf:134 Loading TS 023f8f1a-292a-432b-8fc4-de8471358067
....
Wrote 5 bytes

```

https://github.com/u-boot/u-boot/blob/a94ab561e2f49a80d8579930e840b810ab1a1330/doc/README.tee

## AVB Early TA
Source code of Early TA code for AVB in OP-TEE OS : https://github.com/OP-TEE/optee_os/tree/master/ta/avb

UUID of AVB TA: 
https://github.com/OP-TEE/optee_os/blob/master/ta/avb/include/ta_avb.h#L7
#define TA_AVB_UUID { 0x023f8f1a, 0x292a, 0x432b, \
		      { 0x8f, 0xc4, 0xde, 0x84, 0x71, 0x35, 0x80, 0x67 } }

To include this Early TA in your final OP-TEE binary, add a entry like this:
core/arch/arm/plat-<soc>/conf.mk
CFG_IN_TREE_EARLY_TAS += avb/023f8f1a-292a-432b-8fc4-de8471358067

 Example for hikey platform they added like this:
 https://github.com/OP-TEE/optee_os/blob/master/core/arch/arm/plat-hikey/conf.mk#L62
 
 Finaly generated OP-TEE binary (tee.bin/tee.elf/uTee) will contain this .ta contents also. 
 
 In U-Boot, sandbox driver available for testing AVB and RPMB with RPC:
https://github.com/u-boot/u-boot/blob/a94ab561e2f49a80d8579930e840b810ab1a1330/drivers/tee/sandbox.c

# How to add helloworld as Early TA and implement a command in U-Boot to communicate with HelloWorld TA

## 1. Download the prebuilt binaries.
Download the binaries provided in this repository.

```ruby
$ git clone https://github.com/DevendraDevadiga/Early-TA.git
$ cd Early-TA-Demo/optee_qemu_armv8a_prebuilt_binaries
$ ls
optee_qemu_armv8a  README.md
$ cd optee_qemu_armv8a
$ chmod 777 *
```
## 2. Run QEMU
If want share the folders between your Host PC and QEMU Linux, run below command:
```ruby
$ sudo ./run_qemu_share_files.sh
```
Otherwise run the below command:
```ruby
$ sudo ./run_qemu.sh
```
Two terminals will be opened ans pop-up. One for Secure world (OP-TEE) and another for Normal World (Linux) as below

![qemu_secure_and_normal_world_terminals](https://user-images.githubusercontent.com/36186082/147329631-b5e6bca0-c334-4418-971b-f210c0a6038e.png)

Type "c" (Continue) on qemu console. OP-TEE prints will display on Secure world console. 

![Run-QEMU](https://user-images.githubusercontent.com/36186082/184496639-ba4b2044-78d9-487e-8044-dd23523778bf.png)

## 3. Stop at U-Boot
Currently boot delay is given as 5 seconds. Just press enter in the console to stop at U-Boot prompt

![Stop at U-boot](https://user-images.githubusercontent.com/36186082/184496747-859860bc-cb1f-4b1f-8f2f-0f350c6cbcef.png)

## 4. Early TA already initialized 
In the secure console you are able to observe Early TA related messages. Currently "Hello World" TA is added as Early TA which built with OP-TEE binary.

![Early-TA-HelloWorld](https://user-images.githubusercontent.com/36186082/184496772-3b01a9ef-1bf3-44f8-ad78-c1c21bbaea8e.png)

## 5. Run the command "optee_helloworld" at U-Boot
In U-Boot added the implementation of "optee_helloworld" command. To get the usage information, just type the command as below:

![Stop at U-Boot and run Hello world](https://user-images.githubusercontent.com/36186082/184496718-777709f7-c48a-4c69-8d30-0f52bd66da32.png)

## 6. Observe both console logs
Put the console side by side to see the logs. Press enter many times in secure console log to see the logs when we just run the "optee_helloworld" command.

![Both Console](https://user-images.githubusercontent.com/36186082/184496797-1885301c-bfb3-43b3-8137-e96e108a8292.png)

## 7. Run "optee_helloworld" command at U-Boot, which will load "Hello World" early TA in OP-TEE
Pass the value as 99 to command, so that in OP-TEE, Hello World TA will incrment the value to 100.
![Early-TA](https://user-images.githubusercontent.com/36186082/184496807-6e5dadef-52c8-4955-9b6d-090c39bb4c69.png)

Run the another test by passing value 199 to command, so that Early TA will increase the value to 200.

![Incement The value](https://user-images.githubusercontent.com/36186082/184496825-4e2ee2f6-e307-4cab-b0a5-f761c47ab3a3.png)





# Useful Links:

https://github.com/OP-TEE/optee_os/commit/d0c636148b3a

https://github.com/OP-TEE/optee_os/commit/ed30b6c7493cff4c2d29064f75b037f5677ddb67

https://github.com/OP-TEE/optee_os/issues/4729

https://optee.readthedocs.io/en/latest/architecture/trusted_applications.html#early-ta

https://github.com/OP-TEE/optee_os/blob/3.6.0/mk/config.mk#L247-L253

https://github.com/OP-TEE/optee_os/issues/3305


# OP-TEE OS 
# Add an early TA
# Early TA/ User Space TA/ Pseudo TA - Which is loaded into OP-TEE first?
# QEMU supports early TA?
# Integrity check & encryption of early TA 
# early TA: avb
# Run ta before normal world starts
# stm32mp1: CFG_EARLY_TA
# EARLY_TA_PATHS
# CFG_EARLY_TA=y

# Add an early TA
# Early TA/ User Space TA/ Pseudo TA - Which is loaded into OP-TEE first?
# QEMU supports early TA?
# Integrity check & encryption of early TA 
# early TA: avb
# Run ta before normal world starts
# stm32mp1: CFG_EARLY_TA
# EARLY_TA_PATHS
# CFG_EARLY_TA=y
