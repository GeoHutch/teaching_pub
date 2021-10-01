# ModelSim: First Steps for students in ECE154A

This note is a mix of writing up my 24 Sep section, and answering questions that have floated to the surface since then.
You may find it especially helpful if you're still struggling with Lab 1.

A quick note up front: you may find legacy documentation (or friends who took the class in previous semesters) talking about running ModelSim on your personal computer.
Unfortunately, licensing changes from Mentor make that infeasible this year.
This documentation is meant to summarize a few sources containing relevant information to get students connected, coding, and simulating RTL designs.

## Setup & Requirements
You'll need a few things set up on your machine to get connected to a working envirnoment:

* An RDP client program: this will vary by OS, but most work roughly the same
  * `Remote Desktop` by Microsoft is default-installed on Windows, and can be installed on Mac
  * `xfreerdp` is reccomended by cluster admins for use on Linux (consider `remmina` if you want a more advanced graphical environment)
* The client for campus' Pulse VPN: [Downloads](https://www.it.ucsb.edu/pulse-secure-campus-vpn/get-connected-vpn)
* A UCSB account configured with Duo MFA (Note that this is different than Google MFA on your UCSB account): [Self-service](https://duo-mgmt.identity.ucsb.edu/)
* A UCSB CoE account: [Self-service](https://ucsb-engr.atlassian.net/wiki/spaces/EPK/pages/575373516/New+UCSB+Community+Member+Information)

## Connect to campus
*Skip this step if you're connecting from a campus network.*

The lab computers are behind the campus firewall, and will not accept connections from outside the university.
Once you have Duo MFA set up, use it and the Pulse VPN client to get on campus.

## Connect to ECI cluster
Regardless of what RDP client you use, you'll probably need to input your CoE username, password, and the hostname you're connecting to.
`eci-linux-lab.engr.ucsb.edu` is a load-balancer; it may not always send to you the same compute server (though they all share a filesystem).
In the worst case, ModelSim in the old session will keep holding on to locks on its transient files, blocking the new session from running.
You may want to just pick your favorite number from 01 to 20, and connect directly to `ecellvm-xx.engr.ucsb.edu`.
Since you're now not trusting the load-balancer to give you a good server, occasionally use system monitor programs like `top` and `users` to check that you're not on an unreasonably loaded server.

## Starting ModelSim & Project Organization
If you're connected over RDP as described, you'll find a desktop shortcut (under the category ECE) for ModelSim's GUI.
ModelSim will encourage you to organize your project files into "projects", which need to be associated with a working directory.
Thus, it may be helpful to run `mkdir -p ece154a/lab1` in the shell to make a working directory where you can organize your first lab.

## Your First Module
The half-adder is one of the simplest digital circuits that still does some interesting logic.
For the 2 inputs, you can probably verify the \(2^2 =4\) logic cases in your head.
Here, we'll use that simplicity to prove to ourselves that the testbenching method we're using is actually valid.

The half-adder can be set up in Verilog as follows

```
module my_half_adder (a, b, s, cout);

input a,b;
output s,cout;

assign s = a ^ b;
assign cout = a & b;

endmodule
```

Note that within the ModelSim GUI's project view, you can create a new "Source File" to contain this module.
Using this option will automatically designate it for compilation, or you can write Verilog code in a plain file,
and add that to ModelSim's source list.
Quickly running "Compile" here, despite not having any "runtime" behavior verification, may help you catch syntax issues (like forgotten semicolons).

## Testbenches
Unlike the purely hardware-description focus of your main Verilog code,
a powerful way of thinking about testbenches is as part "test harnass" and part "imperative code" that feeds values into it.
The "test harnass" instantiates and wraps around the "Device Under Test" (the module you're verifying),
providing any necessary pre- and post-processing of signals to correctly run your device in a set of enlightening configurations.
The "imperative-like code" sets values in Verilog memories (`reg`'s) to define test cases, and reads relevant output values.
These imperative-like segments will be separated from other parts of a testbench module by `initial begin ... end` blocks.

As an example, let's make a quick-and-dirty testbench for the half-adder above
```
`timescale 1ns/100ps
module zz_my_half_adder();

reg a,b;
wire s,cout;

my_half_adder dut(a,b,s,cout);

initial begin
  #3;
  a = 0;
  b = 0;
  #1 $display("a=%d, b=%d, s=%d, cout=%d", a, b, s, cout);

  #3;
  a = 1;
  b = 0;
  #1 $display("a=%d, b=%d, s=%d, cout=%d", a, b, s, cout);

  #3;
  a = 0;
  b = 1;
  #1 $display("a=%d, b=%d, s=%d, cout=%d", a, b, s, cout);

  #3;
  a = 1;
  b = 1;
  #1 $display("a=%d, b=%d, s=%d, cout=%d", a, b, s, cout);
end

endmodule
```

Let's break down some of the syntax at work here.
In the very first line, we're setting the `timescale` of the simulation (the amount of time advanced by calling the `#1` operation) to 1 nanosecond,
with a simulation granularity of 100 picoseconds.
Setting granularity to a tenth of the timestep is a nice working default for a lot of situations,
but irrelevant to a simple combinatorial testbench like this (a later note may get into what goes into this setting).
Then, we create the "test harnass" for our circuit.
We need two memories, into which the test cases of $a$ and $b$ can be written, and two wires to connect the outputs of the DUT to (they're wires here because the half-adder outputs combinatorially, not synchronously).
Then we use the imperative block, stepping time with the `#n;` construct, to write test cases into memory and read the outputs.
The `$display(...)` procedure acts loosely follows string-building rules for `printf(...)` from C.
However, you'll note that time is stepped to allow updates to propogate through the logic of the DUT and display correctly.

Reading this, you should be skeptical of how you would write reasonable testbenches for a more complicated circuit.
Above, we used 4 lines to define a single test case, varying only a couple characters between them.
One possible solution might be to put test cases into a Verilog memory, and use the imperative-`for` construct
(remember that this is different from the `generate for` syntax construct you might use in a module!)
to step through them.

As a possible snippet, imagine we have a `reg [1:0] test_cases [3:0]` correctly filled with test cases.
We might replace the imperative block above with something like

```
initial begin
  for (i=0;i<4;i=i+1) begin
    #3;
    a = test_cases[i][1];
    b = test_cases[i][0];
    #1 $display("a=%d, b=%d, s=%d, cout=%d", a, b, s, cout);
  end
end
```

Quick note: Verilog, unlike modern C standrads, won't let you declare an iteration variable in-line.
To make this example work, there should be a declaration `integer i;` somewhere.

Here, the careful reader should be skeptical of me again.
Why did I elide how to make an array correctly filled with test cases?

Verilog, unlike its sibling SystemVerilog, lacks array assignment syntax.
Instead, you're encouraged to use the `$readmem[bh]` family of procedures to populate preset memories from text files.
These read lines of a plain text file (interpreted as binary or hexidecimal, respectively) into words in a Verilog memory array.
For example, if the file `zz_my_adder_testcases.bin` contained the text
```
0_0
0_1
1_0
1_1
```
then calling `$readmemb("zz_my_adder_testcases.bin", test_cases)` would perform the memory setup needed.
Be careful about the shape of the Verilog memory you're writing into!
The procedures have additional optional arguments specifying the start and end of the memory segment, which may be relevant to you later but shouldn't be for lab 1.
If you think you *do* need it, I encourage you to dig around online documentation or chat with a TA.

## Running your simulation
Within the ModelSim GUI, the menu option **Simulate > Start Simulation** will prompt you to attach your simulation to a compiled module
(remember to compile your testbench module!), and once you do, dump you to a blank wave-viewer screen.
Add traces from your testbench with the **Add Wave** context menu option.
Then run the simulation with the menu option **Simulate > Run > Run -all**.
You should see traces of the saves you added pop up on the viewer, but may need to zoom in.

## Collaboration Tools
We, your course staff, encourage you to work on the labs in groups of up to three.
Sharing with your peers, teaching each other lives at the "evaluate" & "create" levels of Bloom's taxonomy of learning, while directly working on labs may be closer to "analyze" or "apply".
Working together will foster a deeper understanding of the material (and it's less work to grade 30 assignments than 90 ;) ).

Don't overthink your collaboration tools, though!
Sometimes looking over a partner's sholder and pair-programming is more helpful than any technical solution.
However, your source files will mostly be plain text (Verilog code, initial memory configurations, etc), so you can version control it with `git`.]
Feel free to use private repos on gitlab or a similiar hosting service.
Since ModelSim likes to dump transient files from its compilation and simulation into your project directory, it may be helpful to use a `.gitignore` file like [this](https://github.com/github/gitignore/blob/master/Global/ModelSim.gitignore).

## Good luck! See you next time!
If you've been struggling with Lab 1, hopefully this note hits your specific problem.
Good luck!
