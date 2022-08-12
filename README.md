# Early-TA
Detail about Early TA

#What is Early TA
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

# so early use is limited to a subset of the TEE Internal Core API (crypto...)
https://github.com/OP-TEE/optee_os/commit/d0c636148b3a
https://github.com/OP-TEE/optee_os/commit/ed30b6c7493cff4c2d29064f75b037f5677ddb67
https://github.com/OP-TEE/optee_os/issues/4729
https://optee.readthedocs.io/en/latest/architecture/trusted_applications.html#early-ta
https://github.com/OP-TEE/optee_os/blob/3.6.0/mk/config.mk#L247-L253
https://github.com/OP-TEE/optee_os/issues/3305
