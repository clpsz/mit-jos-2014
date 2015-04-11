AS = nasm
LD = ld
CFLAGS =
LDFLAGS =

target = hello
srcs = hello.s
objs = $(srcs:.s=.o)


.PHONY: all
all: $(target)

$(target): FORCE check_toolchain $(objs) $(headers)
	$(LD) -o $@ $(objs) $(LDFLAGS)

check_toolchain:
	@if ! type nasm > /dev/null 2>&1; then echo please install nasm; exit 1; fi;

$(objs):%.o:%.s
	@$(AS) -f elf32 -o $@ $<

.PHONY: FORCE
FORCE: 

clean:
	rm -f $(target) *.o

