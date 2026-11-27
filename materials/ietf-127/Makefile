TXT := $(wildcard usecase*.txt)
PDF := $(patsubst %.txt,img/%.pdf,$(TXT))

.PHONY: all clean
all: $(PDF)

img/%.pdf: %.txt | img
	aasvg compatible < $< | inkscape --pipe --export-type=pdf --export-filename=$@

img: ; mkdir -p $@
clean: ; rm -rf img
