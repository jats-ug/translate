ODGS := $(wildcard draw/*/*.odg)
PNGS := $(patsubst %.odg,%.png,${ODGS})

all: ${PNGS}

%.png: %.odg
	unoconv -n -f png -o $@.tmp $< 2> /dev/null   || \
          unoconv -f png -o $@.tmp $<                 || \
	  unoconv -n -f png -o $@.tmp $< 2> /dev/null || \
          unoconv -f png -o $@.tmp $<
	convert -resize 800x $@.tmp $@
	rm -f $@.tmp

.PHONY: clean
