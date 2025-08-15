# Booting using Limine

Now that we have a kernel let's just run it! Well it is a bit harder than that. To execute it we will need a bootloader. A bootloader is the system that runs after the BIOS when the computer starts up. For this tutorial we will use Limine.

## Limine config file

Step 1 is to create a Limine configuration file named `limine.conf`

First we set a timeeout. It tells how much time it will take before Limine automatically boots unless a key has been pressed:

```yaml
timeout: 3
```

Then we create a new entry called "TutorialOS" (or the name of your own OS). We specify that we use the Limine boot protocol and set the kernel path to be `boot():/boot/kernel`. Keep in mind that the `kernel_path` is not the path on your host OS but instead in the ISO file we are going to create in a moment.

```yaml
/Limine TutorialOS
    protocol: limine
    kernel_path: boot():/boot/kernel
```

## The Makefile

First we make sure that all of the default gmake rules and variables are gone. We also create a simple utility macro for creating user variables if they arent already defined.

```py
MAKEFLAGS += -rR
.SUFFIXES:

override USER_VARIABLE = $(if $(filter $(origin $(1)),default undefined),$(eval override $(1) := $(2)))
```

Then we set the default architecture to x86_64 and set the QEMU to have 2G RAM

```py
$(call USER_VARIABLE,KARCH,x86_64)
$(call USER_VARIABLE,QEMUFLAGS,-m 2G)
```
