#
# Copyright (c) 2011, Intel Corporation
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#
#    * Redistributions of source code must retain the above copyright
#      notice, this list of conditions and the following disclaimer.
#    * Redistributions in binary form must reproduce the above
#      copyright notice, this list of conditions and the following
#      disclaimer in the documentation and/or other materials provided
#      with the distribution.
#    * Neither the name of Intel Corporation nor the names of its
#      contributors may be used to endorse or promote products derived
#      from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
# "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
# LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS
# FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE
# COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT,
# INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
# BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
# LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
# CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
# LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN
# ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
# POSSIBILITY OF SUCH DAMAGE.
#

RELEASE:=0
HV_OBJDIR:=build
HV_FILE:=acrn
EFI_OBJDIR:=$(HV_OBJDIR)/bsp/uefi/efi
C_SRCS = boot.c pe.c malloc.c
ACRN_OBJS := $(patsubst %.c,$(EFI_OBJDIR)/%.o,$(C_SRCS))
INCLUDE_PATH += ../include/bsp/

OBJCOPY=objcopy

HOST  = $(shell $(CC) -dumpmachine | sed "s/\(-\).*$$//")
ARCH := $(shell $(CC) -dumpmachine | sed "s/\(-\).*$$//")

ifeq ($(ARCH),x86_64)
	LIBDIR := $(shell if [ -d /usr/lib64 ]; then echo /usr/lib64; \
			else if [ -d /usr/lib ]; then echo /usr/lib; fi ; fi;)
	FORMAT=efi-app-x86-64
else
	ARCH=ia32
	LIBDIR=/usr/lib32
	FORMAT=efi-app-ia32
endif

INCDIR := /usr/include

# gnuefi sometimes installs these under a gnuefi/ directory, and sometimes not
ifneq ("$(wildcard $(LIBDIR)/gnuefi/crt0-efi-$(ARCH).o)","")
	CRT0 := $(LIBDIR)/gnuefi/crt0-efi-$(ARCH).o
	LDSCRIPT := $(LIBDIR)/gnuefi/elf_$(ARCH)_efi.lds
else
	CRT0 := $(LIBDIR)/crt0-efi-$(ARCH).o
	LDSCRIPT := $(LIBDIR)/elf_$(ARCH)_efi.lds
endif



CFLAGS=-I. -I.. -I$(INCDIR)/efi -I$(INCDIR)/efi/$(ARCH) \
		-DEFI_FUNCTION_WRAPPER -fPIC -fshort-wchar -ffreestanding \
		-Wall -I../fs/ -D$(ARCH) -O2

CFLAGS += -mno-mmx -mno-sse -mno-sse2 -mno-80387 -mno-fp-ret-in-387

ifeq ($(ARCH),ia32)
	ifeq ($(HOST),x86_64)
		CFLAGS += -m32
	endif
endif
ifeq ($(ARCH),x86_64)
	CFLAGS += -mno-red-zone
endif

LDFLAGS=-T $(LDSCRIPT) -Bsymbolic -shared -nostdlib -znocombreloc \
		-L$(LIBDIR) $(CRT0)
EFIBIN=$(HV_OBJDIR)/$(HV_FILE).efi
BOOT=$(EFI_OBJDIR)/boot.efi

CONF_FILE=$(CURDIR)/../clearlinux/acrn.conf

all: $(EFIBIN)
	$(OBJCOPY)  --add-section .hv="$(HV_OBJDIR)/$(HV_FILE).bin"  --change-section-vma .hv=0x6e000 --set-section-flags .hv=alloc,data,contents,load  --section-alignment 0x1000 $(EFI_OBJDIR)/boot.efi  $(EFIBIN)

install: $(EFIBIN) install-conf
	install -D $(EFIBIN) $(DESTDIR)/usr/share/acrn/$(HV_FILE).efi

$(EFIBIN): $(BOOT)

$(EFI_OBJDIR)/boot.efi: $(EFI_OBJDIR)/boot.so

$(EFI_OBJDIR)/boot.so: $(ACRN_OBJS) $(FS)
	$(LD) $(LDFLAGS) -o $@ $^  -lgnuefi -lefi $(shell $(CC) $(CFLAGS) -print-libgcc-file-name)

install-conf: $(CONF_FILE)
	install -d $(DESTDIR)/usr/share/acrn/demo
	install -t $(DESTDIR)/usr/share/acrn/demo -m 644 $^

clean:
	rm -f $(BOOT) $(HV_OBJDIR)/$(HV_FILE).efi $(EFI_OBJDIR)/boot.so $(ACRN_OBJS) $(FS)

$(EFI_OBJDIR)/%.o:%.S
	[ ! -e $@ ] && mkdir -p $(dir $@); \
	$(CC) $(CFLAGS)   -c -o $@ $<

$(EFI_OBJDIR)/%.o: %.c
	[ ! -e $@ ] && mkdir -p $(dir $@); \
	$(CC) $(patsubst %, -I%, $(INCLUDE_PATH)) -I. -c $(CFLAGS) $(ARCH_CFLAGS) $< -o $@

%.efi: %.so
	$(OBJCOPY) -j .text -j .sdata -j .data -j .dynamic -j .dynsym -j .rel \
		-j .rela -j .reloc --target=$(FORMAT) $*.so $@
