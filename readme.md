Here's your MemSim — Dynamic Memory Management Visualizer. Here's what's packed in:
Three Memory Modes

Paging — frame-based allocation with configurable frame count and page size
Segmentation — visual segment map with custom-named segments, base/limit table
Virtual Memory — extended configuration for total memory size

Four Replacement Algorithms

FIFO — evicts the oldest loaded page
LRU — evicts the least recently used page
Optimal — theoretical best (looks ahead at future references)
Clock — second-chance algorithm with reference bits

Four Visualization Tabs

Memory View — animated frame grid, reference string tracker with hit/fault color coding, and a live event log
Page Table — full table with valid/dirty/referenced bits + TLB (Translation Lookaside Buffer) with 4 entries
Address Translation — enter any virtual address to see VPN/offset breakdown, TLB hit/miss, and physical address computation with a visual address space canvas
Analytics — fault-over-time chart, hit/fault donut ratio, algorithm comparison bar chart (all 4 algos run against the same string simultaneously), and per-process memory usage

Controls — step forward/back, auto-run with adjustable speed (1–10×), jump to end, random reference string generator, and multi-process support.Memory visualizerCode · HTML Downloadexplain the project completely2 AprI'll give you a complete walkthrough of the MemSim project — its architecture, features, algorithms, and how everything connects.

What MemSim Is
MemSim is a single-file, browser-based interactive simulator for three core operating system memory management techniques: Paging, Segmentation, and Virtual Memory. It lets you define your own memory configuration, feed in a page reference string, and then watch — step by step — how the OS would handle memory allocation, page faults, and page replacement.
Everything runs in pure HTML + CSS + JavaScript with zero external dependencies (just Google Fonts for typography).

The Three Memory Modes
1. Paging
This is the default and most feature-rich mode. Physical memory is divided into fixed-size frames, and logical memory is divided into equally-sized pages. The simulator maintains a page table that maps virtual page numbers (VPNs) to physical frame numbers (PFNs).
When a process references a page:

If it's already in a frame → Page Hit (green flash, logged)
If it's not → Page Fault (orange flash), and a replacement algorithm decides which frame to evict

2. Segmentation
Memory is divided into variable-sized named segments (Code, Data, Stack, etc.). You define each segment's name and size in bytes. The visualizer shows a proportional horizontal bar of the entire memory space, color-coded per segment, plus a segment table showing each segment's base address, limit, and size. This mode disables page replacement since segmentation doesn't use fixed-frame paging.
3. Virtual Memory
Extends the paging model by adding a configurable total memory size (in KB). This models the idea that a process's virtual address space can exceed physical RAM, with the OS managing which pages are currently resident.

The Four Replacement Algorithms
These only apply in Paging and Virtual Memory modes. When all frames are full and a new page must be loaded, one existing page must be evicted.
FIFO — First In, First Out
Maintains a queue of pages in the order they were loaded. The page at the front of the queue (the oldest resident page) is always evicted next. Simple but can suffer from Bélády's anomaly — adding more frames can actually increase faults.
Implementation: a queue[] array. On fault, queue.shift() gives the victim; the new page is pushed to the end.
LRU — Least Recently Used
Evicts the page that was accessed least recently. On every hit, the accessed page is moved to the back of the queue. On fault, the front of the queue (least recently used) is evicted. More accurate than FIFO in modeling real access patterns.
Implementation: on each hit, the page is removed from its current position in queue[] and re-appended to the end.
Optimal (OPT)
Theoretical best-case algorithm — not implementable in a real OS since it requires knowing the future reference string. It evicts the page whose next use is furthest in the future. Used as a benchmark to measure how close FIFO and LRU come to ideal behavior.
Implementation: a futureUse() function scans the remaining reference string for each resident page and returns the index of its next occurrence (or Infinity if it won't be used again). The page with the highest future index is evicted.
Clock (Second Chance)
A practical approximation of LRU used in real operating systems. Each frame has a reference bit. When a page is accessed, its bit is set to 1. On fault, a clock hand scans frames circularly — if a frame's bit is 1, it's cleared to 0 (second chance) and the hand moves on; if it's 0, that frame is evicted. This avoids the overhead of tracking full LRU ordering.
Implementation: clockPtr tracks the hand position; refBits[] array stores one bit per frame.

The Simulation Engine
The core is the buildSimSteps() function. It takes the full reference string, frame count, and algorithm, and precomputes every step upfront into an array called simState. Each step object contains:

step — index in the reference string
page — the page being referenced
hit — boolean, whether it was in memory
evicted — which page was kicked out (null if none)
targetFrame — which physical frame was used
memory — a snapshot of all frames at this moment
refBits / clockPtr — Clock algorithm state

Because all steps are precomputed, navigation is instant in both directions — stepping forward and backward is just reading from the array, not re-running the algorithm.

The Four Visualization Tabs
Memory View
The primary view. It has three parts:
Reference String Track — displays every page reference as a chip. The current step chip is highlighted and scaled up. Past chips are color-coded green (hit) or orange (fault), giving you a visual history at a glance.
Physical Memory Grid — shows one row per frame. Free frames are dark; occupied frames show the page number, the owning process ID, and the frame index. When a frame is written on fault, it flashes orange. On a hit, it flashes green. The grid adjusts its column layout automatically based on frame count.
Event Log — a reverse-chronological list of every event so far. Each entry shows the step number, HIT or FAULT badge, the page referenced, which frame was used, and which page was evicted if applicable. New entries animate in from the left.
Controls — Prev / Next step, Auto Run (with 1–10× speed slider), and Jump to End. Auto Run uses setInterval with a delay derived from the speed value.
Page Table
Shows the full software page table for all processes. Columns include Virtual Page Number, Frame Number, a Valid bit (green badge if in memory, red if not), a Dirty bit (yellow badge — randomly assigned on load to simulate write tracking), reference count, access count, and the step at which the page was last used.
Below the main page table is the TLB (Translation Lookaside Buffer) — a hardware cache of the 4 most recently translated pages. In real hardware the TLB is checked before the page table; here it's visualized as a 4-entry table that updates on every step.
Address Translation
Lets you manually enter any virtual address (decimal or hex) and see the full translation broken down:

VPN = floor(virtual address / page size)
Offset = virtual address mod page size
Physical address = frame number × page size + offset

The tool checks the TLB first (TLB Hit), then the page table (Page Table Hit), and reports a Page Fault if neither has a valid mapping. A canvas diagram on the right visually maps virtual pages to physical frames with a dotted arrow showing the active translation.
Analytics
Four charts rendered on HTML5 Canvas:
Page Faults Over Time — a line chart showing the cumulative fault count as steps progress. The area under the line is filled for readability.
Hit / Fault Ratio Donut — a ring chart showing the proportion of hits vs faults with exact counts and percentage in the center.
Algorithm Comparison Bar Chart — runs all four algorithms against the exact same reference string and frame count in real time, rendering a bar for each showing total fault count. The currently selected algorithm is highlighted with a dashed border. This is the most analytically useful view.
Process Memory Usage — a horizontal bar per process showing how many references it generated relative to the total, helping visualize workload distribution across processes.

The Data Structures
simState[] — array of precomputed step snapshots, the backbone of the whole simulation.
pageTableData — a nested object keyed by process ID, then by virtual page number. Each entry holds VPN, frame, valid, dirty, reference count, and last-used step.
tlbData[] — array of up to 4 TLB entries, each with VPN, PFN, valid bit, and process ID. Uses a simple recency eviction (most recent first, oldest dropped when full).
processes[] — array of process objects with an ID and a color. Pages are assigned to processes by page % processes.length, so with 2 processes, even pages go to P1 and odd to P2.
segments[] — array of segment definitions (name, size, color) used in Segmentation mode.

The Styling Architecture
The entire UI is built on CSS custom properties (variables) set on :root. The aesthetic is a dark terminal / cyberpunk theme with a green primary accent (#00f5c4), orange for faults (#ff6b35), purple for process IDs (#7c3aed), and yellow for dirty bits (#facc15). A CSS scanline overlay is applied via a fixed ::before pseudo-element with a repeating gradient to simulate a CRT monitor effect. Fonts are Orbitron (display/headers), Space Mono (body), and Share Tech Mono (data/monospace content).
