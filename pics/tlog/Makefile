# Makefile

DOT = dot
PIC = pic
ROFF = groff
CONVERT = convert

pic_files = $(wildcard *.pic)
eps_files = $(pic_files:.pic=.eps)
pic_png = $(pic_files:.pic=.png)

dot_files = $(wildcard *.dot)
dot_png = $(dot_files:.dot=.png)

all: $(pic_png) $(dot_png)

%.eps: %.pic
	$(ROFF) -Kutf8 -p $^ | ps2eps > $@

%.png: %.eps
	$(CONVERT) -density 600 $^ -resize 100% $@

%.png: %.dot
	$(DOT) -Tpng $^ -o $@

clean:
	rm -rf $(eps_files) $(pic_png) $(dot_png) 

