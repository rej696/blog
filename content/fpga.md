# FPGA Notes

## The Plan
Using a Nandland GoBoard with GHDL and Yosys-ghdl-plugin opensource fpga toolchain for lattice ice40 fpgas, to work through the Nandland Getting Started with FPGAs book (https://nandland.com/book-getting-started-with-fpga/)

Work through the book, doing all the exercices

Do some projects:
- UART
- VGA
- ADC/DAC?
- Audio Something?
- Basc Processor (e.g. for running forth)?

Read Free Range VHDL (https://github.com/fabriziotappero/Free-Range-VHDL-book/tree/master)

## GHDL
Followed tutorials for using GHDL https://ghdl.github.io/ghdl/quick_start/index.html

see https://github.com/rej696/sandbox/tree/main/vhdl

### Docker
Using containers provided here https://github.com/hdl/containers

### Makefile
Adapted Makefile from https://github.com/rej696/nandland-goboard-vhdl-examples for building and programming

Added featuers for multifile projects, simulators. check full_adder.

```make
TOP ?= top
WORKDIR ?= build
PCF ?= goboard.pcf

UID = $(shell id -u)
GID = $(shell id -g)

DOCKER = docker run -it --rm -u $(UID):$(GID) -v $(PWD):/src -w /src
DOCKER-BUILD = $(DOCKER) hdlc/impl:icestorm
DOCKER-PROG = $(DOCKER) hdlc/prog

GHDL = $(DOCKER-BUILD) ghdl
YOSYS = $(DOCKER-BUILD) yosys
NEXTPNR = $(DOCKER-BUILD) nextpnr-ice40
ICEPACK = $(DOCKER-BUILD) icepack
# Don't use docker for programming, device pass through doesn't work
ICEPROG = iceprog

VHDL_SOURCES = $(shell find . -name "*.vhdl")

# Nandland Go Board Settings
GHDL_GENERICS = #-gCLK_FREQUENCY=25000000
PACKAGE = vq100
DEVICE = -hx1k
FREQ = --freq 25

.PHONY: all build synth pnr sim prog clean

all: $(WORKDIR)/$(TOP).bin

build: $(WORKDIR)/$(TOP).bin

synth: $(WORKDIR)/$(TOP).json

pnr: $(WORKDIR)/$(TOP).asc

sim: $(VHDL_SOURCES)
	mkdir -p $(WORKDIR)
	$(GHDL) -i -v --workdir=$(WORKDIR) $(VHDL_SOURCES)
	$(GHDL) -m -v --workdir=$(WORKDIR) $(TOP)
	$(GHDL) -r -v --workdir=$(WORKDIR) $(TOP)

prog: $(WORKDIR)/$(TOP).bin
	$(ICEPROG) $<

clean:
	rm -rf $(WORKDIR)

$(WORKDIR)/$(TOP).bin: $(WORKDIR)/$(TOP).asc
	$(ICEPACK) $< $@

$(WORKDIR)/$(TOP).asc: $(WORKDIR)/$(TOP).json
	$(NEXTPNR) --hx1k --json $< $(FREQ) --pcf $(PCF) --pcf-allow-unconstrained --package $(PACKAGE) --asc $@

$(WORKDIR)/$(TOP).json: $(VHDL_SOURCES)
	mkdir -p $(WORKDIR)
	$(GHDL) -i -v --workdir=$(WORKDIR) $(VHDL_SOURCES)
	$(GHDL) -m -v --workdir=$(WORKDIR) $(TOP)
	$(YOSYS) -m ghdl -p "ghdl --workdir=$(WORKDIR) $(TOP); synth_ice40 -json $@"
```

### iceprog
iceprog is for programming, must be natively installed (not in a container).

There are scripts located here to install https://github.com/ddm/icetools/tree/master

I built from source:
```bash
git clone https://github.com/YosysHQ/icestorm.git ~/opt/icestorm
cd ~/opt/icestorm
make
sudo make install
```


### Links
- Learn VHDL resources from GHDL: https://github.com/ghdl/ghdl/issues/1291
- VHDL Reference Manual: https://edg.uchicago.edu/~tang/VHDLref.pdf
- Free Range VHDL open source book: https://github.com/fabriziotappero/Free-Range-VHDL-book/tree/master
- FPGA4Fun resources and project ideas: https://www.fpga4fun.com/
