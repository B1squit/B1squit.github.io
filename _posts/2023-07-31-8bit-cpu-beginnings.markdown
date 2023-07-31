---
layout: post
title: "Logic Operations with a Full Binary Adder"
date: 2023-07-31 22:30 +0100
categorys: CPU
---
Before fully attempting to create an 8 bit CPU, I have decided to investigate the title of this post. The question is why do this when I could implement all the logic individually for the ALU using seperate components.

The primary motivation behind all this is to investigate whether this utilises less logic gates than seperately implementing each desired function separately, since as of writing this post I am in possession of a limited number of 74 series ICs. If this does lead to a decrease in component count, this could also give rise to the question of whether this reduced parts count consumes less power. Nevertheless, to begin with a full binary adder needs to be implemented for which I will be using Logisim-Evolution to simulate it, though I do plan on learning VHDL soon after this and reimplementing it. 

### Specififcations

Initial specifications for the adder will be:
  * Full binary adder logic
  * 2 inputs from parallel to serial shift registers 
  * 1 output to a serial to parallel shift register
  * 1 D type latch to store the carry bit/flag
  * 8 bit word

This specification will form a 1 bit full binary adder which will perform calculation in series, and in simulation would look like this: ![fba](/blog/assets/imgs/8-bit-cpu/full-binary-adder.png)

And an example to show that this does work: ![fba example](/blog/assets/imgs/8-bit-cpu/full-binary-adder-example.png)

### Addition of logic

A full binary adder can be expressed with 2 logic equations, one for the sum represented by `A xor B xor C` and the carry bit represented by `A and (B xor C) or B and C`. From these we can start to figure out which inputs to change to achieve certain logic functions. For now one of the simplest operations we can achieve is `and` and to achieve this we should inspect the carry bit statement. Below is a truth table for the carry bit:

|   Cin   |   A   |   B   |  Cout   |
| :-----: | :---: | :---: | :-----: |
|    0    |   0   |   0   |    0    |
|    0    |   0   |   1   |    0    |
|    0    |   1   |   0   |    0    |
|    0    |   1   |   1   |    1    |
|    1    |   0   |   0   |    0    |
|    1    |   0   |   1   |    1    |
|    1    |   1   |   0   |    1    |
|    1    |   1   |   1   |    1    |

If we focus on the rows where `Cin` is set to 0, we find that the only operation performed by the logic gates is an `and` between `A` and `B`, which can be verified by simplifying the expression for `Cout`. Hence we can conclude that to gain the `and` capability we need to set the carry line low, and select for the `Cout` output to be read into the bit shift register which requires modification to the original circuit. Also of note, when `Cin` is set to 1 the output acts as an `or`.

To simplify, the sum bit circuitry will be removed to focus on adding functionality to the carry. During normal operation the carry bit needs to be fed back into the circuit, but when logic functionality is chosen the carry needs to be excluded. Blocking the signal will involve placing an `and` gate at the feedback from the carry, meaning the carry will be active on high and `and`/`or` on low. To inject the custom bit we can insert an `or` gate afterwards (which will be known as logic select or in this example `or`/`and` enable). This does introduce the issue of `or` existing with a don't care bit since the logic select will override the carry bit not matter if the carry line is enabled or not, but this will be solved later. The circuit is implemented as follows: ![fba and or](/blog/assets/imgs/8-bit-cpu/full-binary-adder-and-or.png)

Now it is time to focus on the sum bit, which can be expressed by `A xor B xor Cin`. This quite clearly shows that an `xor` operation is possible, so at this point it may be worth discussing `not`. While we may not have this gate explicitly present, we can still create it. When looking at the truth table for `xor` it is evident that if one input is constantly set to 1, the gate inverts the other. Since we can already manipulate one of these inputs, namely the carry line, we just need to ensure one of the inputs `A` or `B` stays set to 0 to not intefere with operation. To control this, what was done for the other logic control can be applied here, where register B is cut off from input using an `and` gate while the carry utilises the control from the previously added functionality. This would require a seperate control bit for register B, so I tried to simplify this since `not` cannot occur during an `xor` and vice versa by splitting the logic select line and placing an inverter feeding into the B register enable, but this interfered with the previously added logic (don't worry if that didn't make sense, something I tried just didn't work), so a fix for this was found later. For now the logic table for these functions looks like this:

|   Carry Enable    |   Logic Select    |   Function 1   |   Function 2   |
| :---------------: | :---------------: | :------------: | :------------: |
|         0         |         0         |       AND      |       XOR      |
|         1         |         0         |      Carry     |       Sum      |
|         X         |         1         |       OR       |       NOT      |

As shown, `or` and `not` share a don't care state. We can infact exploit this to gain the register B cutoff that was mentioned previously by differentiating between the two functions, so if we have the logic switch high and, for example, the carry enable switch high we want register B to be disabled, but in any other case of the two we want register B to be enabled. This logic is equivalent to a `nand` gate so we introduce it for control, and realistically we cannot get any simpler than introducing 1 extra gate. ![fba not xor](/blog/assets/imgs/8-bit-cpu/full-binary-adder-not-xor.png) If the opposite case were to be explored, we would need the output to be disabled when the logic switch is high and carry enable is low, meaning this input would need inverting introducing an extra gate ontop of the required `nand` gate. Thanks to some aditional testing it also became evident that this had the unintended side effect of creating an `nxor` and buffer function, writing the input in register A to the accumulator.

|   Carry Enable    |   Logic Select    |   Function 1   |   Function 2   |
| :---------------: | :---------------: | :------------: | :------------: |
|         0         |         0         |       AND      |       XOR      |
|         0         |         1         |       OR       |       NXOR     |
|         1         |         0         |      Carry     |       Sum      |
|         1         |         1         |      Buffer    |       NOT      |

This would complete the exploration of potential functions within the ALU, however for now both functions exist as seperate circuits providing separate outputs. These need to be combined together so that they output to the same output register which again will be done using combinational logic, for which an extra bit will be needed, as suggested by the need for the two function columns (which could be interpreted in a sense as a K-map). The selection could be arbitrary so I will decide that the function 2 column will be active low, and the opposite for the function 1 column. In the end, the logic selection and schematic for the logic will be:

|   Function Enable   |   Carry Enable    |   Logic Select    |   Function   |
| :-----------------: | :---------------: | :---------------: | :----------: |
|          0          |         0         |         0         |      XOR     |
|          0          |         0         |         1         |      NXOR    |
|          0          |         1         |         0         |      Sum     |
|          0          |         1         |         1         |      NOT     |
|          1          |         0         |         0         |      AND     |
|          1          |         0         |         1         |      OR      |
|          1          |         1         |         0         |     Carry    |
|          1          |         1         |         1         |     Buffer   |

![fba final](/blog/assets/imgs/8-bit-cpu/full-binary-adder-final.png)

The circuit does become somewhat larger than a pure full binary adder, however it was possible to add an aditional 6 functions by using 8 more gates which is not all that much. Another way this implementation could have been approached is using a corresponding logic gate for a given function, with its output tied to an input on a multiplexer, which would add an extra gate for each of the 5 logic operations and introduce a whole new component entirely, though simplify implementation greatly.

### Closing thoughts
 Looking at the design created here and the proposed 8x1 mux design, I have began to think of some potential benefits and drawbacks of both with the first big one being the main motivation behind this experimentation. If the 8x1 mux is used as an IC it would greatly simplify implementation, since in an ideal world only one new chip would be introduced to the circuit as well as a few already discussed logic gates to emulate the behaviour created, but do note the ideal, since we do not live in such a world. I don't currently own one of these chips, so I would have to implement the mux in its entirety using 74 series logic which would explode the part count and use more space than I have, so in my circumstances it would seem that this addition of a few logic gates is a valid work around. However what the use of a mux does allow for is the use of custom functions rather than being tied to the realm of the full binary adder, which would be extremely nice to have, even allowing functions to be implemented as blocks that can be interchanged.

 Some other issues that are present is the lack of commonly supported functions in hardware, such as subtraction and a logical shifts. The first can be emulated in software as a subroutine though, with the ability to perform a bitwise not, an addition of 1 to turn this number into two's complement and then adding this to the required value. As for the logical shifts, the simplest way to apply a logical left shift would be by simply adding a number to itself and repeating this process with the next calculated value n number of times. A second more interesting and slightly more challenging way to implement a left as well as right arithmetic shifts could be implemented by using the buffer function and regulating how many times the clock is cycled for the ALU, increasing and decreasing this as needed. This cycle control will be necessary to implement anyway since all the other functions require as many clock pulses as there are bits in the word being processed. 

 So looking onward, the next step will be as mentioned to learn VHDL to implement this in a development environment that will scale better and quicken workflow, as well as potentially implement this on a breadboard to see how this would in reality or continue development in simulation.
