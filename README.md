# Linux kernel build script for Arch Linux based system.

My expectation of how this should work:
- after applying the kernel patch, some optimizations of modern processors will be available;
- after forcibly setting the gcc optimization flags, all makefiles will be modified to use them;
- after executing `make localmodconfig` a local hardware-specific .config file will be created;
- after applying the template, some required drivers will be added from the template, and gcc will be forced to use the local processor optimization if it is recognized;
- a menu for manual configuration of the kernel will be executed and after exiting from it, the build process will be launched.

> based on default Arch Linux Kernel build script
