---
layout: post
title: "Implementing the ALU in Verilog"
date: 2024-08-18 15:00 +0100
categorys: CPU
---
It's been a year since the last post! Well, it's been a year ... but there have been no posts since. University has really taken my time out this year, especially with packing so many modules into the first term. Nonetheless I have managed to work on the project.

First things first, there has been a minor change to the specification of the project, instead of being written in VHDL the language of choice is now Verilog/SystemVerilog since one of my second year courseworks was written entirely in Verilog.

For the development process I tried to work as closely as possible to a development cycle followed in industry, where tests are developed before the main application itself. Following this principle the rest of the system was developed bottom up, with the full binary adder, shifters and their corresponding tests developed individually, considering edge cases and writing general tests using random value inputs. Once these base modules were implemented they were later joined in both the Arithemtic Logic Unit module and testbench.

What I find rather amusing in all of this is that most of my time was spent on implementing, testing and correcting the testbenches rather than the modules they were meant to test, although there were occasions where the tests did come in handy for verifying the modules. In the end though everything did work out correctly with both the tests and modules working fine.