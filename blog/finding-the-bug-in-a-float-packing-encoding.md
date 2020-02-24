# WIP: Hex<img src="https://github.com/hexops/website/raw/master/media/png/agent.png" height="26px"></img>ps devlog: Finding the bug in a float packing encoding

Recently, we came across the need to pack multiple (lesser precision and normalized) values into a single 32-bit float:

```
encode(x, y, z, w) == k
decode(k) == (x, y, z, w)
```

This certainly isn't a new type of problem by any means, and with being able to reinterpret floats as binary values and bitshift them on CPUs, this is quite easy to do. However, in some environments such operations are not possible:

- Restrictive GUI shader node editors (e.g. Blender's, if you want a pure node editor implementation -- and likely some game engines' ones as well).
- Older GLSL versions, such as the ones used in WebGL 1.0 (unfortunately Safari has no support for WebGL 2.0 as of the time of writing this).

So, can you do this with just the basic arithmatic operators found in these older GLSL versions and GUI shader editors (where lookup tables are not possible)?

## Research

A bit of Googling turns up many articles and questions on the subject, including:

- http://aras-p.info/blog/2009/07/30/encoding-floats-to-rgba-the-final/
- http://marcodiiga.github.io/encoding-normalized-floats-to-rgba8-vectors

Perhaps the most useful being the latter, which goes into great detail about how this often copied-and-shared encoding works.

## Problem

After putting together [a quick implementation](https://play.golang.org/p/r_RX1vwX55N) in Go, I found to my surprise this relatively popular encoding actually has what appears to be a quite poor distribution of precision!

As an example, I tried to encode and decode `vec4{17, 15, 20, 25}` (remember that these values must be normalized, specifically by only providing inputs that are divisions of 256) you'll get:

```
vec4{17, 15, 20, 25} => vec4{32, 15, 20, 25}
```

This can't be right! 17 turned into 32! All the other values are correct though, so why is this? Even without digging into the specifics, we can deduce a few things just from the behavior alone:

- The Y, Z, and W components can store 255 unique values but have deviations of += 1 after actually dividing by 256.0. Odd!
- The W component can only store 7 unique values! Worse yet, [we run into overflow](https://play.golang.org/p/C7i34luvldW):

```
vec4{0, 15, 20, 25} => vec4{0, 15, 20, 25} good!
...
vec4{16, 15, 20, 25} => vec4{0, 15, 20, 25} data loss in X!
vec4{17, 15, 20, 25} => vec4{32, 15, 20, 25} data loss in X!
...
vec4{47, 15, 20, 25} => vec4{32, 15, 20, 25} data loss in X!
vec4{48, 15, 20, 25} => vec4{64, 15, 20, 25} data loss in X!
...
vec4{80, 15, 20, 25} => vec4{64, 15, 20, 25} data loss in X!
vec4{81, 15, 20, 25} => vec4{96, 15, 20, 25} data loss in X!
...
vec4{111, 15, 20, 25} => vec4{96, 15, 20, 25} data loss in X!
vec4{112, 15, 20, 25} => vec4{128, 15, 20, 25} data loss in X!
...
vec4{144, 15, 20, 25} => vec4{128, 15, 20, 25} data loss in X!
...
vec4{145, 15, 20, 25} => vec4{160, 15, 20, 25} data loss in X!
...
vec4{176, 15, 20, 25} => vec4{192, 15, 20, 25} data loss in X!
...
vec4{208, 15, 20, 25} => vec4{192, 15, 20, 25} data loss in X!
vec4{209, 15, 20, 25} => vec4{224, 15, 20, 25} data loss in X!
...
vec4{239, 15, 20, 25} => vec4{224, 15, 20, 25} data loss in X!
vec4{240, 15, 20, 25} => vec4{0, 16, 20, 25} OVERFLOW! And Y increased by 1!
...
vec4{255, 15, 20, 25} => vec4{0, 16, 20, 25}
```

Note: In [the marcodiiga implementation](http://marcodiiga.github.io/encoding-normalized-floats-to-rgba8-vectors), which mine is based on, the component with only 7 unique values is the R component in RGBA which is arguably worse (assuming you care more about color than alpha).

## Finding the solution

So is the math here wrong? Well if we try and narrow the problem down [by only encoding the X component](https://play.golang.org/p/nmGZ6N7gCKR) we find it actually _can_ encode and decode all values 0-255 without loss! What gives?

And [if we try](https://play.golang.org/p/EmkGg9ZNCxS) and encode all values `vec4{i, i, i, 0}`, where `i` is `0-255` we find _that also works_:

```
vec4{0, 0, 0, 0} => vec4{0, 0, 0, 0}
vec4{1, 1, 1, 0} => vec4{1, 1, 1, 0}
vec4{2, 2, 2, 0} => vec4{2, 2, 2, 0}
...
vec4{255, 255, 255, 0} => vec4{255, 255, 255, 0}
```

Yet if we change the W component to `1` instead of `0`, we see values start to get clobbered:

```
vec4{0, 0, 0, 1} => vec4{0, 0, 0, 1}
vec4{1, 1, 1, 1} => vec4{0, 1, 1, 1} // X is wrong
vec4{2, 2, 2, 1} => vec4{2, 2, 2, 1}
vec4{3, 3, 3, 1} => vec4{4, 3, 3, 1} // X is wrong
vec4{4, 4, 4, 1} => vec4{4, 4, 4, 1}
vec4{5, 5, 5, 1} => vec4{4, 5, 5, 1} // X is wrong
vec4{6, 6, 6, 1} => vec4{6, 6, 6, 1}
vec4{7, 7, 7, 1} => vec4{8, 7, 7, 1} // X is wrong
vec4{8, 8, 8, 1} => vec4{8, 8, 8, 1}
vec4{9, 9, 9, 1} => vec4{8, 9, 9, 1} // X is wrong
```

You may notice a pattern above: only every second number is wrong! What if we change the W component to `2`? [Well](https://play.golang.org/p/ut4dOyVj4j2):

```
vec4{0, 0, 0, 2} => vec4{0, 0, 0, 2}
vec4{1, 1, 1, 2} => vec4{0, 1, 1, 2} // X is wrong
vec4{2, 2, 2, 2} => vec4{0, 2, 2, 2} // X is wrong
vec4{3, 3, 3, 2} => vec4{4, 3, 3, 2} // X is wrong
vec4{4, 4, 4, 2} => vec4{4, 4, 4, 2}
vec4{5, 5, 5, 2} => vec4{4, 5, 5, 2} // X is wrong
vec4{6, 6, 6, 2} => vec4{8, 6, 6, 2} // X is wrong
vec4{7, 7, 7, 2} => vec4{8, 7, 7, 2} // X is wrong
vec4{8, 8, 8, 2} => vec4{8, 8, 8, 2}
vec4{9, 9, 9, 2} => vec4{8, 9, 9, 2} // X is wrong
```

Hmm, and [with W being `3`](https://play.golang.org/p/iPfHJ5lSZ0z):

```
vec4{0, 0, 0, 3} => vec4{0, 0, 0, 3}
vec4{1, 1, 1, 3} => vec4{0, 1, 1, 3} // X is wrong
vec4{2, 2, 2, 3} => vec4{0, 2, 2, 3} // X is wrong
vec4{3, 3, 3, 3} => vec4{4, 3, 3, 3} // X is wrong
vec4{4, 4, 4, 3} => vec4{4, 4, 4, 3}
vec4{5, 5, 5, 3} => vec4{4, 5, 5, 3} // X is wrong
vec4{6, 6, 6, 3} => vec4{8, 6, 6, 3} // X is wrong
vec4{7, 7, 7, 3} => vec4{8, 7, 7, 3} // X is wrong
vec4{8, 8, 8, 3} => vec4{8, 8, 8, 3}
vec4{9, 9, 9, 3} => vec4{8, 9, 9, 3} // X is wrong
```

No change.. And [with W being `4`](https://play.golang.org/p/1oUrGxKr4Fv)?:

```
vec4{0, 0, 0, 4} => vec4{0, 0, 0, 4}
vec4{1, 1, 1, 4} => vec4{0, 1, 1, 4} // X is wrong
vec4{2, 2, 2, 4} => vec4{0, 2, 2, 4} // X is wrong
vec4{3, 3, 3, 4} => vec4{0, 3, 3, 4} // X is wrong
vec4{4, 4, 4, 4} => vec4{0, 4, 4, 4} // X is wrong
vec4{5, 5, 5, 4} => vec4{8, 5, 5, 4} // X is wrong
vec4{6, 6, 6, 4} => vec4{8, 6, 6, 4} // X is wrong
vec4{7, 7, 7, 4} => vec4{8, 7, 7, 4} // X is wrong
vec4{8, 8, 8, 4} => vec4{8, 8, 8, 4}
vec4{9, 9, 9, 4} => vec4{8, 9, 9, 4} // X is wrong
```

Now we see it for sure: As W increases, we lose precision in the X component. What if we tried more values? How does the grouping of wrong vs. correct values change then?

- W=1 => 1 consecutive wrong values
- W=2 => 3 consecutive wrong values
- W=3 => 3 consecutive wrong values
- W=4 => 7 consecutive wrong values
- W=5 => 7 consecutive wrong values
- W=6 => 7 consecutive wrong values
- W=7 => 7 consecutive wrong values
- W=8 => 15 consecutive wrong values

You may notice from the above that _we lose half the X precision each time the W component almost doubles_. And [if we test this theory](https://play.golang.org/p/kQU3WUTT8H8), we indeed find that for a W component of 16 we get a grouping of 32 wrong values -- or nearly half the precision:

- W=16 => 32 consecutive wrong values
- W=32 => 64 consecutive wrong values
- W=64 => 128 consecutive wrong values

Suddenly, the pattern makes sense! For every additional bit that W consumes (doubling), we lose one bit of precision in X!

## WIP

This blog post is still a WIP!
