# Xilinx Vivado Lessons Learned

Some issues I had along the way.

## [Place 30-575] Sub-optimal placement of clock

This happen when you are not dealing with clock generation in a proper way.
Being able to understand why requires some knowledge of the clocking architecture.

My case: this shows up when I was integrating a MAC reference design and a MIG
reference design. Both of them have a CMT(MMCM/PLL) to generate their required clocks.
Each of them works fine on its own. But when I integrated whem, the Implementation complains
the [Place 30-575] issue. Basically the two CMTs are not in the same clock region.

I went to check `UG472 7 series clocking` and found the following:
```
When a clock input drives a single CMT, the clock-capable input and CMT (MMCM/PLL)
must be in the same clock region.

A single clock input can drive other CMTs in the same column. In this case, an MMCM/PLL
must be placed in the same clock region as the clock-capable input..... If you have to
drive a CMT from a clock-capable input that is not in the same clock region,
the attribute CLOCK_DEDICATED_ROUTE=BACKBONE must be set.
```

This explains. My CMTs are in two different clock regions.
To fix the issue, I modified both reference designs and added one CMT at top-level,
whose output clocks are used to drive both MAC and MIG.

--  
YS  
Jan 26, 2019

## Importance of Testbench

So I was merging the MAC and MIG part, along with some logic.
In order to do that, I did some hacking and added some Verilog glue code.
Besides, I have to change the MAC reference design a little bit to use
the new AXI-S interface. Initial merging does not work. I was not good at
writing verilog and lazy enough to construct a new TB for the changed MAC.
I was trying to use ILA to debug, all the way after rounds and rounds of
Synthesis&Implementation. I could go take a shower during the compilation time.

I stop wasting my time and take a step back to modify the MAC's original TB.
I changed it to accomadate the new AXI-S interface. And bang, I got the bug:
an `axi_s_rst` signal was not assigned, it was at `Z` state. Stupid like that.

This is pretty much like an uninitialized variable in C, but in the parameter list.
GCC won't complain it for sure. But for Verilog, this output port has to be assigned
to something, isn't the compiler supposed to give a warning or error or something?

Anyhow, the takeaway lesson is: Life is short, write testbench. Don't do that stupid
`Synthesis->Implementation->In-system Debug` thing.

--  
YS  
Jan 27, 2019

## Auto Timing Constraints of Clock Wizard 

So today I was trying to understand how `Timing violation` and `Timing Constraints` actually work.
And keep thinking how Vivado knows that the clock that a certain RTL module connected to,
is using a legitimate frequency. Because it seems we only have `create_clock` for the primary clock,
those Clock Wizard generated clocks do not have associcated timing constraints.
So I went through `UG903 Chapter 3 Defining Clocks`. This chapter is very informative.

Takeaways:
- Use `report_clocks` to check all clocks.
- Vivado treats all clocks as `propagated clocks`, that is, non-ideal. When you use
`report_clocks` in TCL console, all clocks has `P` attribute.
- Vivado automatically creates the constraint for these on the output pins of CMBs.
This means, you don't need to add `create_clock` for Clock Wizard generated output clocks.

I think, once Vivado knows what pins are clock, and what frequency they are, it will
be able to analyze everything. Just to simply what Vivado does: We let Vivado know
what pins, wires, or ports are clock signals. Then when Vivado is doing synthesis or
implementation, it knows what frequency a certain logic is using, all because you
have clocks to every RTL module. Along with some additional input/output delay,
timing exceptions, Vivado is able to do more sophisticated stuff. All in all,
we should 1) define clocks, 2) write good timing constrains (know what commands
belong to timing constrains catagory).

Reference on Timing:
- UG903, Chapter 3 Defining Clocks
- UG906, Chapter 2 Timing Analysis Features
- UG949
  - Chapter 3 Design Creation -> defining clock constraints
  - Chapter 5 Design Closure -> Timing Closure

--  
YS  
Jan 28, 2019

## Rename an auto-generated clock, the processing_order matters.

At some point, we will want to rename those auto-generated clocks (mostly output of clock wizard).
In order to do this, we will put `create_generated_clock` command on our xdc file.

But I found the `processing_order` of the constraint file which the command is in, matters a lot.
If the `processing_order` is `EARLY`, Vivado will complain it cannot find the name of the auto-generated clock.
Setting the `processing_order` to `NORMAL` will work.

I suspect `EARLY` is just way to early, even before the clock wizard (CMBs) are processed, which means
before the auto-generated clocks are created.

--  
YS  
Mar 15, 2019
