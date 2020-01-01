TARGETS= \
    java-list-interface-details.pdf \
    java-list-interface-details.png \
    persistentvector-classes-interfaces.pdf \
    persistentvector-classes-interfaces.png

all: $(TARGETS)

clean:
	/bin/rm -f $(TARGETS)

%.pdf: %.dot
	dot -Tpdf $< > $@

%.png: %.dot
	dot -Tpng $< > $@
