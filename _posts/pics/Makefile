# Makefile

DOT = dot
UML = plantuml

all_dot = $(wildcard *.dot)
all_uml = $(wildcard *.uml)
dot_png = $(all_dot:.dot=.png)
uml_png = $(all_uml:.uml=.png)

all: $(dot_png) $(uml_png)

%.png: %.dot
	$(DOT) -Tpng $^ -o $@

%.png: %.uml
	$(UML) $^

clean:
	rm -rf $(dot_png) $(uml_png)

