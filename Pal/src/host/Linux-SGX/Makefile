include ../../../../Makefile.configs
include Makefile.am

ias_cert_url ?= https://certificates.trustedservices.intel.com/Intel_SGX_Attestation_RootCA.pem
ias_cert_file = quote/$(notdir $(ias_cert_url))

CFLAGS	+= -I. -Iinclude -I../.. -I../../../include -I../../../lib -I../../../lib/crypto/mbedtls/include -Isgx-driver
ASFLAGS += -I. -I../.. -I../../../include

host_files = libpal-Linux-SGX.a pal-sgx debugger/sgx_gdb.so pal.map generated_offsets.py

defs	= -DIN_PAL -DPAL_DIR=$(PAL_DIR) -DRUNTIME_DIR=$(RUNTIME_DIR)
CFLAGS += $(defs)
ASFLAGS += $(defs)
enclave-objs = $(addprefix db_,files devices pipes eventfd sockets streams memory \
		 threading mutex events process object main rtld \
		 exception misc) \
	       $(addprefix enclave_,ocalls ecalls framework platform pages untrusted xstate)
enclave-asm-objs = enclave_entry
urts-objs = $(addprefix sgx_,enclave framework platform main rtld thread process exception graphene) \
	    quote/aesm.pb-c clone-x86_64
urts-asm-objs = sgx_entry
graphene_lib = .lib/graphene-lib.a

.PHONY: all
all: sgx-driver/isgx_version.h $(host_files)

ifeq ($(DEBUG),1)
CC += -gdwarf-2 -g3
CFLAGS += -DDEBUG
ASFLAGS += -DDEBUG
export DEBUG
endif

../../host_endian.h: host_endian.h
	$(MAKE) -C ../../ $<

libpal-Linux-SGX.a: $(addsuffix .o,$(enclave-objs) $(enclave-asm-objs))
	$(call cmd,ar_a_o)

$(addsuffix .o,$(enclave-objs)): CFLAGS += -DIN_ENCLAVE
$(addsuffix .i,$(enclave-objs)): CFLAGS += -DIN_ENCLAVE
$(addsuffix .s,$(enclave-objs)): CFLAGS += -DIN_ENCLAVE
$(addsuffix .o,$(enclave-asm-objs)): ASFLAGS += -DIN_ENCLAVE
$(addsuffix .s,$(enclave-asm-objs)): ASFLAGS += -DIN_ENCLAVE

$(addsuffix .o,$(urts-objs)): quote/aesm.pb-c.h
$(addsuffix .o,$(enclave-objs)): quote/generated-cacert.h

%.o: %.c
	$(call cmd,cc_o_c)

%.i: %.c
	$(call cmd,cpp_i_c)

%.s: %.c
	$(call cmd,cc_s_c)

%.o: %.S
	$(call cmd,as_o_S)

%.s: %.S
	$(call cmd,cpp_s_S)

pal-sgx: $(addsuffix .o,$(urts-objs) $(urts-asm-objs)) $(graphene_lib)
	@echo [ host/Linux-SGX/$@ ]
	@$(CC) $(CFLAGS) -Wl,-z,relro,-z,now -pie $^ -lc -lprotobuf-c -o $@

quote/aesm.pb-c.o: quote/aesm.pb-c.c quote/aesm.pb-c.h
quote/aesm.pb-c.c quote/aesm.pb-c.h: quote/aesm.proto
	@echo [ host/Linux-SGX/quote/aesm.pb-c.c ]
	@echo [ host/Linux-SGX/quote/aesm.pb-c.h ]
	@protoc-c --c_out=. $<

quote/generated-cacert.h: $(ias_cert_file)
	@echo "#define IAS_CA_CERT \"$(shell cat $< | tr -d '[\r\n]')\"" > $@

$(ias_cert_file):
	@wget $(ias_cert_url) -O $@

debugger/sgx_gdb.so: debugger/sgx_gdb.c debugger/sgx_gdb.h sgx_arch.h
	@echo [ host/Linux-SGX/$@ ]
	$(CC) -MD -MP -Wall -fPIC -O2 -std=c11 -c debugger/sgx_gdb.c -o debugger/sgx_gdb.o
	$(LD) -shared debugger/sgx_gdb.o -o debugger/sgx_gdb.so -lc

enclave_entry.o sgx_entry.o: asm-offsets.h

sgx-driver/isgx_version.h:
	$(MAKE) -C sgx-driver $(notdir $@)

ifeq ($(filter clean,$(MAKECMDGOALS)),)
include $(wildcard *.d) $(wildcard debugger/*.d)
endif

include ../../../../Makefile.rules

CLEAN_FILES += $(notdir $(pal_static) $(pal_lib) $(pal_loader))
CLEAN_FILES += debugger/sgx_gdb.o
CLEAN_FILES += quote/aesm.pb-c.c quote/aesm.pb-c.h quote/aesm.pb-c.d quote/aesm.pb-c.o
CLEAN_FILES += $(ias_cert_file) quote/generated-cacert.h

.PHONY: clean
clean:
	rm -rf *.o *.e *.i *.s $(host_files) $(CLEAN_FILES) *.d debugger/*.d signer/*.pyc __pycache__ \
	       signer/__pycache__
	$(MAKE) -C sgx-driver $@

.PHONY: test
test:
