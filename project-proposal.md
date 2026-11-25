Degree level: Bachelor’s
# Project proposal

## 1. The problem and goal

(Explain the problem you want to solve and why it matters. Describe what you
want to improve, how you will measure improvement, and which limits or
requirements must be respected. A short example can help someone outside your
field understand the problem.)

**The problem and goal in my project:**

A digital chip is built from standard cells (logic gates and flip-flops). Most cell libraries offer each cell in several threshold-voltage (Vt) flavours that share the same function and footprint: a regular-Vt (RVT) cell leaks little but is slow, a low-Vt (LVT) cell is faster but leaks more, and a super-low-Vt (SLVT) cell is the fastest and leaks the most. Leakage is power burned even when the chip is idle, so it directly affects the battery life of phones and laptops.

During timing closure a design that fails its clock target is usually fixed by swapping some cells on the failing paths to a lower Vt. The problem is choosing which cells. Swapping everything on every failing path closes timing but wastes leakage. Swapping too few leaves violations. Swapping a cell that sits on many failing paths at once is worth far more than swapping a cell on just one. Lower Vt also makes short paths faster, which can create new hold violations.

Goal: starting from an all-RVT netlist that fails its clock target, find a Vt assignment that meets timing with the lowest possible total leakage, using as few timing-analysis runs as possible.

Requirements (hard limits):

Setup: worst negative slack (WNS) ≥ 0 and total negative slack (TNS) = 0.
Hold: worst hold slack ≥ 0, so no new hold violations.
Only Vt swaps are allowed. Cell function, drive strength and placement do not change, so the netlist stays logically identical.

How improvement is measured:

Total leakage power (reported by the timing tool) at the point where timing is met. This is the main metric.
The number of cells swapped and the number of timing runs used to get there.
Compared against three baselines: all-RVT (low leakage, fails timing), all-LVT or all-SLVT (meets timing, high leakage), and a greedy script that keeps swapping the cell with the best slack-gain-per-leakage on the current worst path.

Example: a path of 12 gates misses the clock by 40 ps. Swapping all 12 to LVT fixes it but multiplies their leakage. If 3 of those gates are also on 20 other failing paths, swapping just those 3 might fix most of the design at a fraction of the leakage cost.

## 2. The simulator

(Describe a program or model that could simulate your system and calculate what
happens under different conditions. Explain what inputs it takes and what
results it produces. State whether it can run on your computer and accept inputs
in a text file, such as JSON or YAML. If you have not chosen a simulator yet,
name an option you want to explore and explain what you need to check.)

**The simulator in my project:**

Tool: OpenSTA, the open-source static timing analysis engine (GPL-3.0). It is built into OpenROAD (BSD-3), which exposes it through Tcl and a Python interface (openroad -python). Static timing analysis is deterministic: the same netlist, libraries and constraints always give the same slack and leakage numbers.

Cell library: the ASAP7 predictive 7 nm PDK. It is open source and ships standard cells in SLVT, LVT and RVT flavours with matching footprints, and OpenROAD-flow-scripts already includes its Liberty files. (I first considered sky130, but its standard-cell libraries are not organised as same-footprint Vt variants, so ASAP7 is the better fit.)

Inputs (all text files):

A gate-level Verilog netlist. I will use small example designs from OpenROAD-flow-scripts, such as gcd or a small ALU or AES block, synthesized in RVT.
ASAP7 Liberty (.lib) files for RVT, LVT and SLVT.
An SDC constraints file with a clock period tight enough that the all-RVT design fails.
A run configuration in YAML: design, clock period, Vt flavours allowed and the run budget.

Outputs: setup WNS and TNS, hold WNS, the N worst paths (each with its cells, cell delays and slack), and total leakage power (report_power).

Runs on my computer: yes. OpenROAD runs on Linux, or on Windows through WSL or Docker. On a design with a few thousand cells, one swap plus an incremental timing update takes seconds, so dozens of iterations fit in a lab session.

MCP server (my code): a Python server that loads the design once and exposes:

get_timing_summary() returns setup WNS/TNS, hold WNS, leakage and the cell count per Vt.
get_worst_paths(n) returns the cells on each path with their delay, Vt and slack.
swap_vt(cells, flavour) applies the swaps and returns the new summary.
reset() returns to all-RVT.

Testability (pytest): on a tiny hand-built netlist (a few gates between two flip-flops) I can calculate slack and leakage by hand from the Liberty tables and assert that the server returns exactly those numbers. I also need to check that swapping a cell and then swapping it back restores identical results, and that two identical runs give identical output.

What I still need to check: whether OpenROAD's Python interface can do the swap directly (replacing an instance's master cell) or whether I must call the Tcl command from Python, and how long one incremental update takes on the largest design I plan to use.

## 3. The agent's role

(Explain what the language-model-based agent will decide or change, and how it
will use the simulator's results to choose its next action. If known, list the
variables it may change and their allowed ranges. Explain what you expect the
language model to contribute: could a simple program that tries different
options and picks the best result achieve the same goal? Include one question
you still need to answer.)

**The agent in my project:**
What it changes: the Vt flavour of each cell instance, chosen from {RVT, LVT, SLVT}. Nothing else (clock period, sizing, placement) is under its control.

The loop:

Read the timing summary and the worst failing paths.
Reason about which cells to change. For example: "these four cells appear on most of the failing paths, so swap them first," or "this path misses by only 3 ps, so one LVT swap is enough and SLVT is overkill."
Call swap_vt with a small batch of changes.
Read the new summary and check whether setup improved, how much leakage it cost, and whether a hold violation appeared.
Adjust. That may mean continuing, undoing a swap that turned out unnecessary (leakage recovery), or moving a cell back to RVT because it created a hold problem.
Stop when all limits are met and the last few attempts to recover leakage gave no gain, or when the run budget (for example 30 timing runs) is used up.

What the language model contributes: brute force is impossible, since a design with 2,000 cells and 3 flavours has 3^2000 combinations. The obvious script, a greedy sensitivity loop, only ever looks at the current worst path. It therefore over-swaps cells that do not help other paths, never undoes earlier choices, and does not anticipate hold problems. The agent can read several path reports at once, spot the shared cells, plan a batch of swaps, and go back to recover leakage. My claim is that it reaches lower leakage than the greedy script, or the same leakage in fewer timing runs. I will test this on several designs and clock targets, and report the result honestly even if greedy wins on some of them.

Question I still need to answer: does the language model actually beat a well-written greedy heuristic, or does it only match it? I also need to decide how much path detail to give the agent: too little and it cannot reason, too much and the reports do not fit its context.

## Explain your project to someone outside your field

(Now describe your whole project again, as if you were explaining it to someone
from another field who has no background in your topic. For example, if you
study chemical engineering, imagine explaining it to a physicist; if you study
aeronautical engineering, imagine explaining it to someone in medicine; if you
study data science, imagine explaining it to a materials engineer. In one or two
short paragraphs, connect all three parts: the problem you want to solve, how
the simulator helps you test ideas, and what the agent will decide or change.
Use everyday language, explain any technical terms you need, and avoid
unexplained abbreviations. Do not simply copy your answers above.)

**My project in simple words:**

A computer chip is made of millions of tiny switches wired into chains, and on every tick of the chip's internal clock a signal must travel through its chain before the next tick. Chip designers can choose between switch versions: a "fast" version that gets signals through quickly but constantly loses a little electricity, like a dripping tap, and a "slow" version that barely drips. If some chains are too slow the chip does not work, but making every switch fast would drain the battery even when the phone is doing nothing. The question is which few switches to upgrade so that every chain finishes in time while wasting as little electricity as possible. This is hard because chains share switches: upgrading one well-chosen switch can fix many chains at once, while a badly chosen one wastes power for little gain.

I will use an existing free program that calculates exactly how long every chain takes and how much electricity the chip leaks, for any choice of switches, in a few seconds. An AI agent will work with this program the way an engineer does. It looks at which chains are too slow, decides which switches to upgrade, asks the program to recalculate, and learns from the result, including undoing upgrades that turned out to be unnecessary. I will compare it with a simple automatic rule and with "upgrade everything", to see whether the agent's reasoning finds a working chip that wastes less electricity.
