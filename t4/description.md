## Simulate rain in a world of cubes

Write a program that calculates the volume of rain water collected in the cube
world described below.

The cube world &mdash; given as input &mdash; consists of a finite set of cubes
on integer coordinates `(x, y, z)`. The positive `y` coordinate means "up".

An infinite amount of rain then falls from an infinite height. Both of these
infinities are taken to really mean "large enough as to make no difference".
As it lands on cubes, the water will follow predictable rules:

* Rain falls everywhere.

* Water falling will land on the first cube below it. It does not fall through
  cubes.

* Water will collect on levels where walls on all sides will keep it in.

* Water will produce vertical waterfalls where such walls are missing.

* Cubes are packed tightly enough that gaps between cubes sharing an edge will
  not let water through. However, the same gaps will readily let air through if
  water needs to displace air for some reason.

Waterfalls work in the simplest way imaginable: if water "escapes" from a
structure of cubes, it will fall straight down along the first available
"chute" of cube-formed empty cells until it hits a cube. (Which it may not
necessarily do. A waterfall may go on to infinite depth.) As a waterfall hits a
cube, it behaves just like other kinds of water: it may spread, collect, and
form new waterfalls as needed.
