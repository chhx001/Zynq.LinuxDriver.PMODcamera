ifneq ($(KERNELRELEASE),)
obj-m := camera.o
else
KERNEL_DIR := /home/xup/cross/trdkernel/linux-xlnx/
#KERNEL_DIR ?= /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)
all:
	echo $(MAKE)
	$(MAKE) -C $(KERNEL_DIR) M=$(PWD) modules ARCH=arm
clean:
	rm *.o *.ko *.mod.c modules.* Module.*
endif
