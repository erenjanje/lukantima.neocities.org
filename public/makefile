rfind = $(wildcard $1$2) $(foreach d,$(wildcard $1*),$(call rfind,$d/,$2))

DIRECTORY := ./
TEMPLATE := $(DIRECTORY)template.html
SOURCES := $(call rfind,$(DIRECTORY),*.md)
TARGETS := $(SOURCES:%.md=%.html)

.PHONY: all clean

all: $(TARGETS)
clean:
	@rm -f $(TARGETS)

%.html: %.md $(TEMPLATE)
	@echo PANDOC $<
	@pandoc $< --table-of-contents --standalone --toc-depth=4 --mathml --output $@ --template $(TEMPLATE) --shift-heading-level-by=-1 --wrap=preserve
