RISCV := /opt/riscv
BUILDROOT := ${RISCV}/buildroot
IMAGES := ${BUILDROOT}/output/images
WALLY := $(shell dirname $(shell pwd))
WALLYLINUX := $(shell pwd)
DIS := ${IMAGES}/disassembly
BRPACKAGES := $(WALLYLINUX)/buildroot-packages
BR2023 := $(WALLYLINUX)/buildroot-config-src/buildroot-2023.05.1

# Buildroot Config Stuff
WALLYBOARDSRC := $(WALLYLINUX)/buildroot-config-src/wally
WALLYBOARD := $(BUILDROOT)/board/wally

# Buildroot Package Stuff
PACKAGE_SOURCE := ${WALLYLINUX}/buildroot-packages/package-source
FPGA_AXI_SDC := ${WALLYLINUX}/buildroot-packages/fpga-axi-sdc
DRIVER := ${PACKAGE_SOURCE}/fpga-axi-sdc.c
PATCHFILE := $(BRPACKAGES)/package-2023.05.1.patch

# Device tree files 
DTS ?= $(shell find -type f -regex ".*\.dts" | sort)
DTB := $(DTS:%.dts=%.dtb)
DTB := $(foreach name, $(DTB), $(IMAGES)/$(shell basename $(name)))

# Disassembly stuff
BINARIES := fw_jump.elf vmlinux busybox
OBJDUMPS := $(foreach name, $(BINARIES), $(basename $(name) .elf))
OBJDUMPS := $(foreach name, $(OBJDUMPS), $(DIS)/$(name).objdump)

# LINUXDIR := $(shell ls $(BUILDROOT)/output/build | grep -e '^linux-[0-9]\+\.[0-9]\+\.[0-9]\+$$' )
# LINUXDIR := $(BUILDROOT)/output/build/$(LINUXDIR)
# BUSYBOXDIR := $(shell ls $(BUILDROOT)/output/build | grep -e '^linux-[0-9]\+\.[0-9]\+\.[0-9]\+$$' )
# BUSYBOXDIR := $(BUILDROOT)/output/build/$(BUSYBOXDIR)

# Gets Linux and Busybox output folders for objedect dumps
# LINUXDIR ?= $(shell find $(BUILDROOT)/output/build -maxdepth 2 -type d -regex ".*/linux-[0-9]+\.[0-9]+\.[0-9]+$$")
# BUSYBOXDIR ?= $(shell find $(BUILDROOT)/output/build -maxdepth 2 -type d -regex ".*/busybox-[0-9]+\.[0-9]+\.[0-9]+$$")

define linuxDir =
$(shell find $(BUILDROOT)/output/build -maxdepth 2 -type d -regex ".*/linux-[0-9]+\.[0-9]+\.[0-9]+$$")
endef

define busyboxDir =
$(shell find $(BUILDROOT)/output/build -maxdepth 2 -type d -regex ".*/busybox-[0-9]+\.[0-9]+\.[0-9]+$$")
endef

.PHONY: all generate disassemble install clean cleanDTB cleanDriver test

# Generate all device trees -------------------------------------------
# TODO: Add configuration for only generating device tree for specified
#       supported FPGA.

all:
	$(MAKE) install
	make -C $(BUILDROOT) --jobs
	$(MAKE) generate
	# TODO: Need to find a way to set the PATH for child processes.
	# source ../setup.sh; $(MAKE) disassemble

# Temp rule for debugging
test:
	@echo $(linuxDir)
	@echo $(busyboxDir)

generate: $(DTB) $(IMAGES)

$(IMAGES)/%.dtb: ./devicetree/%.dts
	dtc -I dts -O dtb $< > $@

$(IMAGES):
	@ echo "No output/images directory in buildroot."
	@ echo "Run make --jobs in buildroot directory before generating device tree binaries."; exit 1

$(RISCV):
	@ echo "ERROR: No $(RISCV) directory. Make sure you have installed the Wally Toolchain."
	@ echo "this can be done with <WALLY>/bin/wally-tool-chain-install.sh"

# Disassembly rules ---------------------------------------------------

disassemble:
	mkdir -p $(DIS)
	make -j $(OBJDUMPS)

$(DIS)/%.objdump: $(IMAGES)/%.elf
	riscv64-unknown-elf-objdump -DS $< >> $@

$(DIS)/%.objdump: $(IMAGES)/%
	riscv64-unknown-elf-objdump -S $< >> $@

$(IMAGES)/vmlinux: $(call linuxDir)/vmlinux
	cp $< $@

$(IMAGES)/busybox: $(call busyboxDir)/busybox
	cp $< $@

# Generating new Buildroot directories --------------------------------

# This directive should be run as: make install BUILDROOT=path/to/buildroot
install: $(BUILDROOT)/package/fpga-axi-sdc $(WALLYBOARD)
	cp $(WALLYBOARD)/main.config $(BUILDROOT)/.config

# CONFIG DEPENDENCIES 2021.05 -----------------------------------------
# $(WALLYBOARD)/main.config: $(WALLYBOARD) $(BRPACKAGES)/wally.config
# 	cp $(BRPACKAGES)/wally.config $@

# $(WALLYBOARD)/linux.config: $(BRPACKAGES)/linux.config $(WALLYBOARD)
# 	cp $(BRPACKAGES)/linux.config $@

# $(WALLYBOARD): $(BUILDROOT)
# 	cp -r $(WALLYBOARDSRC) $(BUILDROOT)/board
# 	cp $(BRPACKAGES)/wally.config $(WALLYBOARD)/main.config
# 	cp $(BRPACKAGES)/linux.config $(WALLYBOARD)/linux.config

# CONFIG DEPENDENCIES 2023.05.1 ---------------------------------------
$(WALLYBOARD): $(BUILDROOT)
	cp -r $(WALLYBOARDSRC) $(BUILDROOT)/board
	cp $(BR2023)/main.config $(WALLYBOARD)/main.config
	cp $(BR2023)/linux.config $(WALLYBOARD)/linux.config

# Buildroot Package ---------------------------------------------------
$(BUILDROOT)/package/fpga-axi-sdc: $(BUILDROOT) $(PATCHFILE) $(BRPACKAGES)/fpga-axi-sdc
	cp -r $(BRPACKAGES)/fpga-axi-sdc $(BUILDROOT)/package
	sed -i 's|FPGA_AXI_SDC_SITE =|FPGA_AXI_SDC_SITE = $(PACKAGE_SOURCE)|1' $(BUILDROOT)/package/fpga-axi-sdc/fpga-axi-sdc.mk
	cd $(BUILDROOT); if git apply --check $(PATCHFILE) > /dev/null ; then git apply $(PATCHFILE); fi

$(PATCHFILE):
	cd $(BUILDROOT); git apply $(PATCHFILE)

$(BUILDROOT):
	git clone https://github.com/buildroot/buildroot.git $@
	cd $@; git checkout 2023.05.x
	#cd $@; git checkout 2021.05

#$(DRIVER):
#	@ if [ -d "$(WALLY)/addins/vivado-risc-v" ] ; then git submodule update --init $(WALLY)/addins/vivado-risc-v; fi
#	cp ../addins/vivado-risc-v/patches/fpga-axi-sdc.c $@
	# For 2021.05
	#sed -i "s|card_hw_reset|hw_reset|1" $@

# ---------------------------------------------------------------------

cleanDriver:
	rm -f $(DRIVER)

cleanDTB:
	rm -f $(IMAGES)/*.dtb

clean:
	rm -rf $(DIS)
