TEVSL
=====

What is it?
-----------

Tevsl is a kind-of fragment-shading language for the Nintendo
GameCube & Wii consoles. The aim is to simplify programming of the
texture environment units of the Flipper/Hollywood chips, moving some
way towards making their fixed-function fragment-processing pipelines
appear more like those available on a modern fully-programmable GPU,
without incurring any performance loss.

Hopefully the friendly syntax of tevsl shader descriptions will make it easier
to fully exploit the capabilities of Flipper/Hollywood GPUs, by making it
clearer what each processing stage is doing and by making modifications to the
fragment-processing setup much easier to accomplish.

What is the TEV unit?
---------------------

The TEV (texture environment) unit is part of the GPU on the
GameCube & Wii which handles fragment (pixel, more or less) processing
operations. Its purpose is to take inputs from e.g. texture lookups and
lighting calculation results and combine them into a final colour for
writing to the embedded framebuffer (EFB), and henceforth to the screen
or a texture. It is not a fully-programmable processor, but can still
be used for quite sophisticated effects.

How does it work?
-----------------

Short stage-by-stage "shader" descriptions are fed into tevsl, which processes
them and emits a description which can be directly #included into C code. For
example basic texture mapping can be achieved by writing the following:

    stage 0:
      tev = texmap0[texcoord0];

The effect of this is to look up texture coordinates specified by texcoord0 in
the texture map texmap0, and assign the RGBA results to "tev". This can be
compiled as follows:

    $ tevsl basic-texture.tev -o basic-texture.inc

and then in your C program, you write something like:

    void setup_basic_texture_mapping (void)
    {
    #include "basic-texture.inc"
    }

Ideally, you should set up Makefile (or whatever other build system you use)
rules so that .inc files are automatically regenerated when you edit the
corresponding .tev file.

Tevsl sets up many of the parameters concerning TEV configuration, greatly
reducing the need for you to keep track of the TEV state yourself. See later
for a full list of the functions which tevsl subsumes.

Basic syntax
------------

You must write consecutively-numbered definitions for each of the stages you
want to use:

    stage 0:
      tev = ...;

    stage 1:
      tev = ...;

(The need to manually number stages may disappear at some point.)

The left-hand side of each assignment corresponds to the output of the TEV unit
for each stage, and the RHS of the assignment corresponds to the inputs to that
stage and the processing which is done on those inputs. You should only assign
to each colour/alpha channel once per stage: "tev =" write to all four
simultaneously, though you can also split the colour/alpha parts and perform
different operations for each (see below).

Though several kinds of expression are supported, note that tevsl can't
magically add programmability to the Flipper/Hollywood chip: so, you're quite
restricted in what you can write on the RHS of each stage description.

The "canonical" form of expression which is supported is as follows:

    stage 0:
      tev = (<accum> [+/-] ((1 - c) * a + c * b) + <bias>) * <scale>;

You can write such expressions out "in full", e.g.:

    stage 0:
      tev = (tev + ((1 - chan0) * texmap0[texcoord0] + chan0 * cr0) + 0) * 2;

Or, since you won't want to use all the available functionality per-stage all
the time, you can write simplified versions of the expression and tevsl will
rewrite the expression for you into the fixed format required by the hardware.
So you can write, e.g.:

    stage 0:
      tev = texmap0[texcoord0] + chan0;

and you will get the expected result. Beware that this is no computer algebra
system which can infer the correct form of expression from anything you feed it
though! Though tevsl makes some attempt to match what you give it, you might
need to manually rearrange your formulae so that tevsl can recognize them
properly, even if mathematically speaking it should be possible for it to deduce
the solution for a particular expression itself.

Further description of the features available are below.

Value ranges
------------

The normal TEV setup functions provide the illusion that colour and alpha values
fall in the range 0...1, corresponding to 8-bit colour channel values 0 to 255,
and tevsl follows the same convention.

Where texture coordinates are manipulated explicitly, e.g. by the
indirect-texture lookup syntax, values typically run from 0 to the actual size
of the texture (minus one). E.g. a 256x256 texture will use coordinates 0...255.

Clamping
--------

You can clamp the result of each TEV stage (to 0...1, i.e. the maximum and
minimum representable values for each of the R,G,B,A colour channels), by
writing:

    tev = clamp (<expr>);

around your expression. Channels are clamped independently, and you can clamp
colour channels but not alpha channels, or vice versa.

Blending
--------

A simplified form of syntax is provided for writing blending equations:

    tev = mix (<var1>, <var2>, <amt>);

This is internally rewritten to:

    tev = (1 - <amt>) * <var1> + <amt> * <var2>;

This is similar to the GLSL mix() function: 0 for <amt> gives the value of
<var1>, 1 for <amt> gives the value of <var2>, and inbetween values linearly
interpolate between the two. Channels are mixed independently.

Note that using this function means the canonical TEV expression can be written
more concisely as:

    tev = (<accum> [+/-] mix (<var1>, <var2>, <amt>) + <bias>) * <scale>;

Simplified forms
----------------

Quite a few simplified forms of the above expression are recognized. These are a
few examples:

    tev = tev + chan0;

    tev = tev * chan1;

    tev = tev * 2;

    tev = tev / 2;

    tev = clamp ((1 - tev) * 2);   (!!! this doesn't work, bug.)

    tev = clamp (chan0.aaa + chan0.rgb);

The last of these introduces a new feature, channel selection. This is described
more fully in the following sections.

Separate colour/alpha expressions
---------------------------------

The colour and alpha settings for each TEV stage are independent. You can
describe this to tevsl by writing separate equations, like so:

    stage 0:
      tev.rgb = chan0.rgb;
      tev.a = 1;

You can't split channels in different groupings from these on the LHS of these
expressions though: we're limited to what's supported by the TEV unit (and we
don't support merging/rearranging expressions setting different channels, even
when it might be technically possible. E.g. this will NOT work:

    stage 0:
      tev.r = chan0.r;
      tev.g = chan0.g;
      tev.b = chan0.b;
      tev.a = chan0.a;

it's not clear how useful such functionality would be.)

If you just write "tev" on the LHS of an expression, then both colour and alpha
expressions will be defined. In that case you're limited to the subset of
expressions which are valid for both the alpha and colour TEV parts.

You can also write all channels explicitly if you wish:

    tev.rgba = chan0.rgba;

This has the same meaning as omitting the ".rgba" parts from both the LHS and
RHS of the expression (if it doesn't, it's a bug!).

Available variables and constants
---------------------------------

The following variables and constants are available for use in expressions.

  * "tev". This may be used on the LHS or RHS of expressions, and is usually
    used to pass intermediate values between stages. The last stage MUST assign
    its output to tev.

  * "cr0", "cr1", "cr2". These may be used on the LHS of RHS of expressions, and
    correspond to the TEV registers which can be used for constant inputs to the
    TEV, or for intermediate results. These map to GX_TEVREG{0,1,2} on the LHS
    of expressions, or GX_CC_C{0,1,2} etc. on the RHS of expressions. When used
    as constants, these are set using GX_SetTevColor. (You must take care not to
    use a given register as *both* a constant and an intermediate value: that
    won't work.)

These variables can be used with channel selectors,

For colour channels, e.g.

    tev.rgb = tev.rgb;  (pass through colour unchanged)
    tev.rgb = cr0.aaa;
  
For alpha channels, e.g.:

    tev.a = cr1.a;

There's no facility to swizzle colours with the tev/crN variables beyond the
alpha selection already shown (.aaa). These are "signed 10 bit" quantities (at
least they are referred to as such in various documentation, although the sign
is in fact separate from the 10 bits, so -1024 to 1023 can be represented). See
later for more discussion on that subject.

  * "texmapN[texcoordM]". This is a regular texture lookup: coordinates
    "texcoordM" are looked up in texture "texmapN", and an RGB (or A) result is
    obtained.

  * "chan0", "chan1". These are the rasterised colour channels 0 and 1 (similar
    to "varying" values in GLSL), used for vertex colouring or the results of
    lighting calculations.

Both texmap and chan0/chan1 variables may use arbitrary swizzles, which are
automatically mapped to the TEV's "swap table" facility (there are only four
channel combinations possible in total: tevsl will attempt to merge tables
wherever possible, and will give an error if you attempt to use too many). So
you can write any of:

    tev.rgb = chan0.rgb;

    tev.rgb = chan0.rrr;

    tev.rgb = chan1.bgr;

    tev.a = texmap0[texcoord1].g;

    tev.rgb = chan1.aaa;

    tev.rgb = chan1.rrr + chan1.ggg;

Even corner cases such as the last are supported, since alpha and colour
channels can be referred to independently by the colour TEV channels, and both
are subject to remapping via swap tables (so in that example, alpha would be
mapped to green, and each of the RGB channels would be mapped to red).

You can use independent swizzles for texture-map lookups and for the rasterised
colour channels. You can only refer to one of chan0/chan1 in each TEV stage, and
you can only refer to a single regular texture in each stage also (you can't use
different colour channels/texture maps in colour and alpha expressions).

  * "k0", "k1", "k2", "k3". These four are extra "konstant" colour channels,
    which can be used to pass constant values to the TEV. These are somewhat
    like GLSL "uniform"s. These are unsigned 8-bit values only, and can be set
    with GX_SetTevKColor.

You can write each of these using ".rgb" (for the colour part of the TEV) or
".a" (for the alpha part of the TEV) selectors, or using all-same selectors,
".rrr", ".ggg" (for colour channels) or ".r", ".g" (alpha channels) etc.,
although you can't use swizzles other than those. E.g.:

    tev.rgb = k0.rgb;

    tev.a = k0.a;

    tev.rgb = k1.rrr;

    tev.a = k0.g;

For all variables, writing them without a selector infers ".rgb" for colour
channels (in tev.rgb = ... expressions), and ".a" for alpha channels (tev.a =
...).

Several scalar constants are available:

  * "0", "1", "0.5", "1/8", "2/8", "3/8", etc.. These may be mapped to either
    regular constants, or "konstant" constants, as appropriate. You probably
    don't normally need to worry about which will be used in a given expression,
    but you may only have one "konstant" value per colour- or alpha-part of each
    stage: tevsl will give an error if your expression needs more than that.

These constants are replicated across each channel.

Ternary (conditional) expressions
---------------------------------

You can write conditional expressions using C-like ternary operations. The
general form of these look like this:

    tev = <accum> [+/-] ((<var1> <cmp> <var2>) ? <true-val> : 0);

A concrete example, an 8-bit comparison:

    tev.rgb = (texmap1[texcoord1].r > tev.r) ? 1 : 0;

The R, G, and B channels of the TEV output for this stage will each be set to 1
(if the comparison is true), or 0 (if it is false). You can also use "==" for an
equality test.

You can also do element-wise comparisons:

    tev.rgb = (texmap1[texcoord1].rgb > tev.rgb) ? 1 : 0;

then three separate 8-bit comparisons will be performed for each of the R, G and
B channels, and the corresponding results written to the RGB channels of TEV.

You can also use a different syntax to "concatenate" channels, and do 16- or
24-bit comparisons:

    tev.rgb = (texmap1[texcoord1]{gr} > tev{gr}) ? 1 : 0;

In such expressions, the most-significant channel comes first, the
least-significant last. The "canonical" ordering of these channels is "bgr",
i.e. concatenation expressions are written "the other way round" from selection
expressions (using .rgb syntax). This shouldn't really be relevant unless you're
running short on swap tables, though.

The "false" arm of the ternary expression must be zero (a hardware limitation:
tevsl is not able to perform much rewriting on these expressions).

Signed 10-bit values
--------------------

Tevsl usually makes no distinction between unsigned 8-bit values and signed
10-bit values (it could try harder to do the right thing actually, although
there would still be ambiguous cases). If the data type is important, e.g. you
are accumulating a value over several stages, you should write an annotation
against the accumulation variable in question:

    tev = (tev:s10) + cr1;

This will attempt to ensure that the "tev" input will end up in the D input,
where it will be used properly as a signed 10-bit value. Other inputs truncate
to the lower 8-bits and interpret them as an unsigned value. Your expression
will fail to match if you try to use too many ":s10" annotations, rather than
silently ignoring them.

Indirect texture lookups
------------------------

Tevsl provides support for configuring indirect texture lookups. These can be
written using a couple of different syntaxes.

A simple form is as follows:

    tev = texmap0[indmtx0 ** texmap1[texcoord0] * indscale0];

"indmtx0" and "indscale0" here refer to the indirect matrix and scale set by the
GX_SetIndTexMatrix API. This expression will look up a texel from
texmap1[texcoord0], matrix-multiply GX_ITM_0 (indmtx0 in the above expression)
with it, then scale each component of the result (indscale0). The scale number
you use must be equal to the matrix number (a hardware limitation).

You can also use a bias expression:

    tev = texmapN[<indmtx> ** (texmapO[texcoordP] + <bias>) * indscaleM];

bias can be, e.g.:

    vec3 (-128, -128, 0)

where the values can each be either -128 or 0 per-channel. (Other biases are
used under certain circumstances, but those are not yet implemented.)

This is used to map colour values (which are read from the texture as 0...255)
to signed values -128...127. The indirect matrix and scale should map these
texture coordinates so that they match the scale of the regular texture (texmapN
above) -- unlike other texture lookups, coordinates are not automatically scaled
from 0...1 to the texture size.

You can add a regular texcoord to an indirectly looked-up texcoord:

    tev = texmap0[texcoord1 + indmtx0 ** texmap1[texcoord0] * indscale0];

The texture lookup (in texmap0 in the example) will then use the resulting
texture coordinates from this calculation. Texcoord1 will be automatically
scaled to texmap0's size before the addition as for normal texture lookups.

You can also use wrapping on the regular texture coordinate using a "modulus"
(%) operator:

    tev = texmap0[(texcoord1 % 32) + indmtx0 ** texmap1[texcoord0] * indscale0];

The right-hand side of the modulus operator can be an integer (as written), or a
two-element vector to use different moduli for the S and T coordinates:

    tev = texmap0[(texcoord1 % vec2 (32, 16)) + indmtx0 ...];

It is sometimes necessary (e.g. for "ST" bump mapping) to calculate an indirect
texture coordinate over several TEV stages. This is supported by tevsl. You can
perform indirect texture lookups without also doing a regular lookup by writing:

    itexcoord = indmtx0 ** texmap1[texcoord0] * indscale0;

and you can add to the indirect coordinate calculated by the previous stage
(only -- i.e. no inbetween stages doing other operations are permitted) by
writing:

    itexcoord = itexcoord + indmtx0 ** texmap1[texcoord0] * indscale0;

This technique can be used over multiple stages to accumulate a 2D (s,t)
texture-coordinate result before performing a final look-up in a regular texture
(which is the only thing you can do with the result).

An implementation detail: when you use this syntax, tevsl takes care of
inserting "GX_TEXMAP_DISABLE" for the appropriate GX_SetTevOrder command for
you.

If you want to use the "dynamic" S and T-type matrices, you can write that as,
for example:

    stage 2:
      itexcoord = itexcoord
		  + (t_dynmtx (texcoord2) ** (texmap2[texcoord0]
					      + vec3 (-128.0, -128.0, 0)))
		    * indscale0;
      tev = tev;

"s_dynmtx" and "t_dynmtx" depend on the texcoord from the "regular" texture
lookup for the stage, so are written using function-like syntax with that
texcoord as an argument. You can still add the same texcoord independently in
the same stage, if you wish (though the ability to do that might not be useful).

Tevsl automatically infers texmap and texcoord numbers to use for some of the
above usage scenarios, in an attempt to avoid extraneous details in the input
language. That code is quite experimental though, so beware that it might break
under some circumstances.

Recognition of indirect texture expressions is a little more strict than regular
TEV expressions, in terms of the order in which you must write terms, etc.. This
is probably a bug, and may be fixed in due course.

Texture coordinate scaling
--------------------------

A feature not mentioned in the previous section (currently slightly
experimental) is support for sharing texture coordinates between direct and
indirect stages. Normally texture coordinates are automatically scaled to the
size of the texture map which is the subject of the lookup in question: this is
true of both direct lookups and indirect lookups.

Sometimes it's useful to use the same texture coordinate for both an indirect
lookup AND a direct lookup: the hardware provides limited support for this, both
in cases where the indirect and direct texture maps are the same size, and when
the indirect texture map is a power-of-two factor smaller than the direct
texture map. You may also use separate scales for the width and height of the
indirect texture.

To use this feature in its simplest form with tevsl, just write the same
texcoord for direct and indirect parts:

    stage 0:
      tev.rgb = texmap0[texcoord0];

    stage 1:
      cr0.rgb = texmap1[indmtx0 ** texmap2[texcoord0] * indscale0];

In this case, the scaling factor for the indirect lookup will be set to 1, so
typically texmap0 and texmap2 will need to have the same size (the size of
texmap1 is irrelevent to the discussion here, always being "manually" controlled
by the indirect matrix).

To use the scaling feature, write the indirect part like so:

    stage 1:
      cr0.rgb = texmap1[indmtx0 ** texmap2[texcoord0 / 2] * indscale0];

or, for separate width/height scales, e.g:

    stage 1:
      texmap1[indmtx0 ** texmap2[texcoord0 / vec2(4, 8)] * indscale0];

You may use any power-of-two scale between 1 and 256.

Note that you cannot share texture *maps* between direct and indirect stages.
Any given map must be used exclusively for direct or exclusively for indirect
lookups.

Z textures
----------

You can use Z-texturing by writing an expression at the last stage, e.g.:

    z = <z + >texmapM:fmt[texcoordN]< + offset>;

(where <> indicate optional parts).

'fmt' must be z8, z16 or z24 (or z24x8, which is identical in meaning to the
last). This specifies the format of the Z texture texmapM, since the Z-texture
setup function needs to know that, and tevsl doesn't otherwise know what the
format should be.

'offset' is a 24-bit offset which may optionally be added to the Z texture. As a
special case you may write "$foo" here (for any foo corresponding to a valid C
variable name), and the variable "foo" will be emitted verbatim in the output
file. This allows injection of different values for the offset at runtime. See
below for an example.

You can either replace the Z buffer for the fragment in question (z = ...) or
add to the usual calculated value for the fragment (z = z + ...).

Use this for image-space colour & Z-buffer compositing. Using Z-textures
automatically configures Z-buffering to occur after texturing.

Alpha test
----------

Alpha test is supported using the following syntax:

    finally:
      alpha_pass = tev.a > 30 && tev.a < 70;

This should be written after the last stage in your input file, since that is
when the alpha-test logically takes place (though the placement of the "finally"
stage isn't actually enforced at present, which is a bug). The hardware supports
two comparisons (as written above), whose results may be combined in several
ways to determine the final true/false (render fragment/do not render fragment)
result.

Note that in alpha_pass expressions, the value of tev.a ranges from 0 to 255,
rather than the normalized range 0 to 1 used in regular TEV stage expressions.
This follows the convention used by the underlying API.

You can write any of the following for the comparison operators, which follow
their usual meaning:

    < <= == != => >

And you can combine two comparisons using the operators:

    && || ^ ==

where "^" means exclusive-OR (first condition or second condition true, but not
both), and "==" means equal or not-exclusive-OR (first and second condition both
true, or both false). The "&&" and "||" operators mean logical-AND and
logical-OR, as per usual C semantics.

You may also write a single comparison, e.g.:

    finally:
      alpha_pass = tev.a > 128;

Then, tevsl will insert a null condition for the second comparison for you.

You may write C variables (as above, like "$foo") instead of constants in
alpha-test expressions (see below).

Using alpha test automatically configures Z-buffering to occur after texturing
(as required by the hardware).

Using C variables in expressions
--------------------------------

In limited circumstances (Z texturing and alpha-test expressions), you may write
C variables instead of constants by writing a variable name preceded with a
dollar sign, e.g. "$variable". This variable name is then emitted verbatim in
the output, so will be captured from the point at which you include the compiled
shader ".inc" file.

E.g. for shader code, "texture-holes.tev":

    stage 0:
      tev = texmap0[texcoord0];

    finally:
      alpha_pass = tev.a > $threshold;

You might compile the shader and include like this:

    void setup_holes_shader (int threshold)
    {
    #include "texture-holes.inc"
    }

Then, you can vary the threshold by calling this function with a suitable
argument.

Note that C-variable substitution doesn't work at present for expressions other
than those mentioned above. Support for use in other types of expression may be
added in the future, if necessary and feasible.

List of automated APIs
----------------------

Tevsl can almost completely control the following set of APIs, freeing you of
the need to call any of them explicitly:

  * GX_SetNumTevStages
  * GX_SetNumChans
  * GX_SetNumTexGens
  * GX_SetTevSwapModeTable
  * GX_SetTevSwapMode
  * GX_SetNumIndStages
  * GX_SetIndTexOrder
  * GX_SetTevOrder
  * GX_SetTevDirect
  * GX_SetTevIndirect
  * GX_SetTevColorIn
  * GX_SetTevColorOp
  * GX_SetTevAlphaIn
  * GX_SetTevAlphaOp
  * GX_SetTevKColorSel
  * GX_SetTevKAlphaSel
  * GX_SetZTexture
  * GX_SetAlphaCompare
  * GX_SetZCompLoc
  * GX_SetIndTexCoordScale

You can also specify much of the functionality provided by the GX_SetTevInd*
helper functions (GX_SetTevIndBumpST, etc.) using tevsl expressions, hopefully
leading to easier-to-understand code.

Bugs & drawbacks
----------------

Many and varied! Most error detection and reporting is nonexistent or poor.
Compiled TEV descriptions are almost completely static, which might limit the
way effects can be combined or altered at runtime.

Tevsl works by tree rewriting, and does not have a type system or any kind of
understanding of the expressions it is manipulating. This might occasionally
cause it to do unexpected or incorrect things, particularly for meaningless
inputs.
