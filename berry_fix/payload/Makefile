AS := ../../tools/binutils/bin/arm-none-eabi-as
CPP := $(CC) -E
LD := ../../tools/binutils/bin/arm-none-eabi-ld
OBJCOPY := ../../tools/binutils/bin/arm-none-eabi-objcopy

ifeq ($(OS),Windows_NT)
EXE := .exe
else
EXE :=
endif

SHELL := /bin/bash -o pipefail

CPPFLAGS := -I ../../tools/agbcc/include -I ../../tools/agbcc -iquote include -nostdinc -undef

ROM := payload.gba
OBJ_DIR := build
CC1             := ../../tools/agbcc/bin/agbcc$(EXE)
override CFLAGS += -mthumb-interwork -Wimplicit -Wparentheses -Werror -O2 -fhex-asm


ELF = $(ROM:.gba=.elf)
MAP = $(ROM:.gba=.map)

C_SUBDIR = src
ASM_SUBDIR = asm
DATA_ASM_SUBDIR = data

C_BUILDDIR = $(OBJ_DIR)/$(C_SUBDIR)
ASM_BUILDDIR = $(OBJ_DIR)/$(ASM_SUBDIR)
DATA_ASM_BUILDDIR = $(OBJ_DIR)/$(DATA_ASM_SUBDIR)

ASFLAGS := -mcpu=arm7tdmi

LDFLAGS = -Map ../$(MAP)

LIB := -L ../../../tools/agbcc/lib -lagb_flash -lsiirtc -lagbsyscall -lgcc

SHA1 := $(shell { command -v sha1sum || command -v shasum; } 2>/dev/null) -c
GFX := ../../tools/gbagfx/gbagfx$(EXE)
AIF := ../../tools/aif2pcm/aif2pcm$(EXE)
MID := ../../tools/mid2agb/mid2agb$(EXE)
SCANINC := ../../tools/scaninc/scaninc$(EXE)
PREPROC := ../../tools/preproc/preproc$(EXE)
RAMSCRGEN := ../../tools/ramscrgen/ramscrgen$(EXE)
FIX := ../../tools/gbafix/gbafix$(EXE)

# Clear the default suffixes
.SUFFIXES:
# Don't delete intermediate files
.SECONDARY:
# Delete files that weren't built properly
.DELETE_ON_ERROR:

# Secondary expansion is required for dependency variables in object rules.
.SECONDEXPANSION:

.PHONY: rom clean compare tidy

C_SRCS := $(wildcard $(C_SUBDIR)/*.c $(C_SUBDIR)/*/*.c $(C_SUBDIR)/*/*/*.c)
C_OBJS := $(patsubst $(C_SUBDIR)/%.c,$(C_BUILDDIR)/%.o,$(C_SRCS))

ASM_SRCS := $(wildcard $(ASM_SUBDIR)/*.s)
ASM_OBJS := $(patsubst $(ASM_SUBDIR)/%.s,$(ASM_BUILDDIR)/%.o,$(ASM_SRCS))

DATA_ASM_SRCS := $(wildcard $(DATA_ASM_SUBDIR)/*.s)
DATA_ASM_OBJS := $(patsubst $(DATA_ASM_SUBDIR)/%.s,$(DATA_ASM_BUILDDIR)/%.o,$(DATA_ASM_SRCS))

SONG_SRCS := $(wildcard $(SONG_SUBDIR)/*.s)
SONG_OBJS := $(patsubst $(SONG_SUBDIR)/%.s,$(SONG_BUILDDIR)/%.o,$(SONG_SRCS))

MID_SRCS := $(wildcard $(MID_SUBDIR)/*.mid)
MID_OBJS := $(patsubst $(MID_SUBDIR)/%.mid,$(MID_BUILDDIR)/%.o,$(MID_SRCS))

OBJS     := $(C_OBJS) $(ASM_OBJS) $(DATA_ASM_OBJS) $(SONG_OBJS) $(MID_OBJS)
# OBJS_REL := $(patsubst $(OBJ_DIR)/%,%,$(OBJS))

SUBDIRS  := $(sort $(dir $(OBJS)))

$(shell mkdir -p $(SUBDIRS))

rom: $(ROM)

# For contributors to make sure a change didn't affect the contents of the ROM.
compare: $(ROM)
	@$(SHA1) rom.sha1

clean: tidy
	rm -f sound/direct_sound_samples/*.bin
	rm -f $(SONG_OBJS) $(MID_OBJS) $(MID_SUBDIR)/*.s
	find . \( -iname '*.1bpp' -o -iname '*.4bpp' -o -iname '*.8bpp' -o -iname '*.gbapal' -o -iname '*.lz' -o -iname '*.latfont' -o -iname '*.hwjpnfont' -o -iname '*.fwjpnfont' \) -exec rm {} +

tidy:
	rm -f $(ROM) $(ELF) $(MAP)
	rm -r build/*

%.s: ;
%.png: ;
%.pal: ;
%.aif: ;

%.1bpp: %.png  ; $(GFX) $< $@
%.4bpp: %.png  ; $(GFX) $< $@
%.8bpp: %.png  ; $(GFX) $< $@
%.gbapal: %.pal ; $(GFX) $< $@
%.gbapal: %.png ; $(GFX) $< $@
%.lz: % ; $(GFX) $< $@
%.rl: % ; $(GFX) $< $@


ifeq ($(NODEP),1)
$(C_BUILDDIR)/%.o: c_dep :=
else
$(C_BUILDDIR)/%.o: c_dep = $(shell $(SCANINC) -I include $(C_SUBDIR)/$*.c)
endif

$(C_BUILDDIR)/%.o : $(C_SUBDIR)/%.c $$(c_dep)
	@$(CPP) $(CPPFLAGS) $< -o $(C_BUILDDIR)/$*.i
	@$(PREPROC) $(C_BUILDDIR)/$*.i charmap.txt | $(CC1) $(CFLAGS) -o $(C_BUILDDIR)/$*.s
	$(AS) $(ASFLAGS) -o $@ $(C_BUILDDIR)/$*.s

ifeq ($(NODEP),1)
$(ASM_BUILDDIR)/%.o: asm_dep :=
else
$(ASM_BUILDDIR)/%.o: asm_dep = $(shell $(SCANINC) $(ASM_SUBDIR)/$*.s)
endif

$(ASM_BUILDDIR)/%.o: $(ASM_SUBDIR)/%.s $$(asm_dep)
	$(AS) $(ASFLAGS) -o $@ $<

ifeq ($(NODEP),1)
$(DATA_ASM_BUILDDIR)/%.o: data_dep :=
else
$(DATA_ASM_BUILDDIR)/%.o: data_dep = $(shell $(SCANINC) $(DATA_ASM_SUBDIR)/$*.s)
endif

$(DATA_ASM_BUILDDIR)/%.o: $(DATA_ASM_SUBDIR)/%.s $$(data_dep)
	$(PREPROC) $< charmap.txt | $(CPP) -I include | $(AS) $(ASFLAGS) -o $@

$(SONG_BUILDDIR)/%.o: $(SONG_SUBDIR)/%.s
	$(AS) $(ASFLAGS) -I sound -o $@ $<

$(OBJ_DIR)/sym_bss.ld: sym_bss.txt
	$(RAMSCRGEN) .bss $< ENGLISH > $@

$(OBJ_DIR)/sym_common.ld: sym_common.txt $(C_OBJS) $(wildcard common_syms/*.txt)
	$(RAMSCRGEN) COMMON $< ENGLISH -c $(C_BUILDDIR),common_syms,../../tools/agbcc/lib > $@

$(OBJ_DIR)/sym_ewram.ld: sym_ewram.txt
	$(RAMSCRGEN) ewram_data $< ENGLISH > $@

$(OBJ_DIR)/ld_script.ld: ld_script.txt $(OBJ_DIR)/sym_bss.ld $(OBJ_DIR)/sym_common.ld $(OBJ_DIR)/sym_ewram.ld
	cd $(OBJ_DIR) && sed -f ../../ld_script.sed ../$< | sed "s#tools/#../tools/#g" > ld_script.ld

$(ELF): $(OBJ_DIR)/ld_script.ld $(OBJS)
	cd $(OBJ_DIR) && ../$(LD) $(LDFLAGS) -T ld_script.ld -o ../$@ $(LIB)

$(ROM): $(ELF)
	$(OBJCOPY) -O binary $< $@

