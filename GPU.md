Building a Transone GPU requires scaling the core architecture to thousands of processing elements, each a compact analogue computer, and adding graphics‑specific fixed‑function units that obey the same four‑state voltage law. The result is a fully parallel, decimal‑analogue GPU capable of vertex shading, rasterisation, pixel shading, and ray tracing—all without a single binary logic gate.

---

## 1. GPU Processing Element (PE) Design

Each PE is a miniature Transone processor optimised for graphics workloads. It retains the column‑sequential ALU but reduces the word width to 4 decimal columns (16 bits equivalent in 4‑state multi‑wire format), which handles fixed‑point coordinates, colours, and texture values with enough precision for high‑quality rendering. A 4‑column word can represent numbers 0–9999, or signed values using a sign‑magnitude column at the leftmost (e.g., column 3 carries 0 V for positive, −1 V for negative).

### PE Internals

- **Register File:** 16 four‑column registers, each stored as scaled charge in capacitive wells and read/written through analogue encoders/decoders that reconstruct the legal multi‑wire buses.
- **ALU:** A 4‑DPE chain (columns 0–3) with carry/borrow, as in the CPU. All operations—add, subtract, multiply, divide, AND, OR, NOT—are available. Multiply and divide are single‑column scalar analogue multipliers, so a full 4‑column multiply requires a sequence of digit‑by‑digit operations, microcoded in the control unit.
- **Special Graphics Instructions:**
  - **MADD (Multiply‑Add):** A fused operation that multiplies two 4‑column numbers and adds a third, crucial for matrix transformations.
  - **CLAMP:** Forces a result to lie between 0 and a given max using window comparators and clipping circuits built into the ALU’s output stage.
  - **INTERPOLATE (LERP):** Linear interpolation using the ALU’s multiply‑add with a fixed factor.
- **Control Unit:** A microcoded sequencer that fetches instructions from a tiny local instruction cache (also analogue, storing opcodes as multi‑column patterns). The sequencer drives the DPE column counter and the operation‑select lines.

Because a PE is smaller than the full 8‑column CPU core, its clock can be slightly higher—around 25 MHz (settling time scales with voltage swing, which remains at most 3 V, so the same 50 ns per step holds). We conservatively keep 20 MHz for uniformity.

---

## 2. Massive Many‑Core Array

To handle millions of pixels and vertices, the GPU contains **16,384 PEs** arranged in a 128×128 2D mesh. Each PE is an independent core with its own instruction stream (SIMT model: all PEs execute the same shader program on different data, but with the ability to mask inactive threads using a predicate flag line). The total aggregate performance is:

\[
16,384 \times 20,\!000,\!000 = 327.68 \text{ GIPS}
\]

Each instruction is a column‑wide operation, but one scalar arithmetic op (e.g., a 4‑digit add) is a single instruction. A 4‑digit multiply takes ~10 instructions (microcoded). Even so, this is enough for real‑time graphics.

### Analog Network‑on‑Chip (ANoC)

Cores communicate via the same spatial voltage‑multiplexed 2D mesh routers used in the 192‑core CPU. The ANoC carries full 4‑column words (up to 5 wires per column, so 20 wires per word) over analog pass‑transistor links. For graphics, a **vertex distributor** broadcasts transformed vertices from a set of vertex‑shader PEs to the rasteriser, and a **fragment collector** gathers shaded pixels to the framebuffer. The ANoC handles all traffic without digitisation.

---

## 3. Graphics Pipeline on Transone

The pipeline is a hybrid: programmable PEs run shaders, while fixed‑function analog blocks handle rasterisation, texture filtering, and ray intersection tests.

### 3.1 Vertex Shading (Fully Programmable)

A batch of PEs is allocated as vertex shaders. Each PE receives a vertex (position, normal, texture coordinates) stored in its registers as multi‑column words. The shader program—a sequence of Transone instructions—transforms the vertex by a 4×4 model‑view‑projection matrix. Matrix multiplication is decomposed into dot products:

\[
\text{new\_x} = m_{00} x + m_{01} y + m_{02} z + m_{03} w
\]

Each MADD is executed using the ALU’s multiplier and adder. The digit‑sequential nature means a single MADD takes roughly 8–10 clock cycles (column by column, with carry propagation). A full vertex transform (16 multiplications, 12 adds) is about 200 cycles per vertex, i.e., 10 µs at 20 MHz. With 4096 vertex‑shader PEs, the GPU can process 409.6 million vertices per second—more than enough.

### 3.2 Primitive Assembly & Clipping (Fixed‑Function)

After vertex shading, vertices are grouped into triangles. The clipping unit is a dedicated analog block that checks triangle vertices against the six view‑frustum planes. Each plane test is a dot product (normal·vertex + offset) computed by an analog summing node and comparator. If the voltage crosses a threshold, the clip logic uses linear interpolation (LERP) built from op‑amp summers and voltage dividers to generate new vertices. All operations use only states −1 V, 0 V, 1 V, 2 V.

### 3.3 Rasterisation (Analog Triangle Interpolator)

Rasterisation converts a triangle into pixel fragments. The Transone rasteriser is a novel analog parallel unit.

- **Edge Function Evaluator:** For each pixel in a tile (e.g., 8×8), three analog summing circuits compute the edge functions:

  \[
  E(x,y) = (x - x_0)(y_1 - y_0) - (y - y_0)(x_1 - x_0)
  \]

  This uses analog multipliers (Gilbert cells) and subtractors. A comparator chain checks if all three edge voltages are of the same sign (for a front‑facing triangle). The voltages are constrained by diode clippers to stay within legal ranges.

- **Barycentric Coordinate Generator:** For each fragment inside the triangle, another set of analog dividers and multipliers computes barycentric weights \( \alpha, \beta, \gamma \) as voltages. These weights are used to interpolate vertex attributes across the pixel.
- **Tile‑Based Parallelism:** The rasteriser processes many tiles simultaneously across multiple rasteriser pipelines. Each pipeline is a hard‑wired array of analog compute cells.

Because the rasteriser works in the voltage domain, it can process a 8×8 tile in a single clock cycle (once inputs settle). This provides massive pixel throughput.

### 3.4 Fragment Shading (Programmable PEs)

The rasteriser feeds fragments to an array of fragment‑shader PEs. Each PE runs a pixel shader program, which may sample textures, compute lighting, and apply colour operations. Texture sampling is supported by a **Texture Unit** that uses the same Transone memory cells to store texel arrays. To perform bilinear filtering, the texture unit computes weighted sums of four neighbouring texels using analog multipliers and adders—again, pure voltage arithmetic.

Fragment PEs share the same design as vertex PEs but are programmed with a different shader. The massive number of PEs (many thousands) ensures high pixel throughput.

### 3.5 Ray Tracing Acceleration (Analog Ray‑Tracing Core)

Ray tracing requires massive amounts of ray‑triangle intersection tests and bounding volume hierarchy (BVH) traversals. The Transone GPU integrates a special **Ray Acceleration Unit (RAU)** that leverages the parallel subtraction matrix and analog comparators to perform intersection tests directly in hardware.

#### Ray‑Triangle Intersection (Möller–Trumbore in Analog)

The Möller–Trumbore algorithm solves for barycentric coordinates \( t, u, v \) with three linear equations. The RAU implements this as a three‑stage analog pipeline:

1. **Edge Vector Computation:** Subtract vertex positions using the parallel subtraction matrix (which creates up to nine −1 V references). Differences are computed in a single cycle.
2. **Cross Product / Dot Product:** Cross and dot products are broken into multiplications and additions. Analog multipliers compute the products; op‑amp summers accumulate. The intermediate voltages are clipped to legal states at each step.
3. **Comparator Decision:** If the computed voltages satisfy \( 0 \le u \le 1, 0 \le v \le 1, u+v \le 1 \), and \( t > 0 \), a hit is registered. These checks are performed by window comparators that output a 1 V (hit) or 0 V (miss) signal.

Because the whole intersection is combinatorial analog (no clock cycles between multiply/add steps if pipelined appropriately with S&H stages), the RAU can test one ray against one triangle in about 200 ns (several op‑amp settling times). However, with 16,384 RAUs working in parallel (one per PE, or shared among a small cluster), the GPU can test billions of rays per second.

#### BVH Traversal

BVH traversal is managed by a simple state machine in the RAU, using the ALU’s logic operations to compare ray‑bounding box intersection distances. A bounding‑box test is a series of interval checks: for each axis, compute \( t_{min} \) and \( t_{max} \) using analog dividers and comparators. The traversal stack is stored in a small local memory (capacitive cells) within the RAU.

---

## 4. Memory Hierarchy

The GPU uses a unified memory architecture built from Transone capacitive DRAM cells, organised in independent banks to feed the parallel PEs.

- **Global Memory:** Many gigabytes of analog DRAM, where each cell stores a scaled charge representing a decimal digit. Readout regenerates the multi‑wire bus. Bandwidth is huge because each bank can output multiple columns in parallel over analog buses.
- **L2 Cache:** On‑chip capacitive storage, split into slices, connected via ANoC.
- **Texture Cache:** Read‑only, optimised for 2D locality, providing four texels per cycle to the texture units.
- **Framebuffer:** A special output region that converts the final pixel colour (multi‑column decimal RGB values) to display voltages via digital‑to‑analogue converters (but still within the 0–2 V range, compatible with display standards). The framebuffer is double‑buffered and can be read by the display controller without blocking the GPU.

---

## 5. Performance Estimate for 3D Graphics

Assume a target resolution of 1920×1080 (2M pixels) at 60 fps.

- **Vertex shading:** A typical scene has 200k vertices per frame. At 20 MHz, a PE takes ~10 µs per vertex (200 cycles for transform + lighting). Using 512 vertex PEs, total time = 200k / 512 * 10 µs = 3.9 ms, well under the 16.7 ms frame budget.
- **Rasterisation:** The analog rasteriser can process a tile of 8×8 pixels in ~50 ns (1 cycle). To cover 2M pixels, we need 2M / 64 = 31,250 tiles. With 128 rasteriser pipelines, that’s 31,250 / 128 * 50 ns = 12.2 µs. Rasterisation is effectively free.
- **Fragment shading:** Assuming an average of 4 fragments per pixel (oversampling) and a shader complexity of ~50 instructions per fragment, each instruction is ~50 ns (20 MHz). So per fragment time = 2.5 µs. With 16,384 fragment PEs, 8M fragments / 16,384 * 2.5 µs = 1.22 ms. Easily within budget.
- **Ray tracing (hybrid):** For ray‑traced shadows/reflections, the RAU can trace 1G rays per second (16,384 RAUs * 20 MHz * ~3 cycles per intersection). A 2M pixel image with 4 rays per pixel requires 8M rays, traced in ~8 ms. This allows real‑time ray tracing effects.

Thus, the Transone GPU comfortably achieves real‑time 3D graphics and ray tracing using a pure analog‑decimal architecture.

---

## 6. Physical Implementation

The 16,384‑PE GPU is laid out in a 128×128 grid, each PE occupying ~0.1 mm² (including register file, ALU, control). Total core area ≈ 1,638 mm², a large but feasible die on a modern process. Deep Trench Isolation surrounds each PE. The chip is packaged in a Flip‑Chip BGA with a copper heat spreader and direct voltage‑plane distribution for the four master rails. Power consumption is high due to static op‑amp bias, but power gating idle PEs (e.g., during rasterisation or when a shader is not fully occupied) reduces average draw.

---

## 7. Conclusion

The Transone GPU is a fully functional, massively parallel analogue graphics processor. It executes vertex and fragment shaders on thousands of tiny column‑sequential ALUs, rasterises triangles with pure voltage arithmetic, and accelerates ray tracing using dedicated analog intersection units. Every wire carries only −1 V, 0 V, 1 V, or 2 V, and all computation is performed by the physical summation of electrical pressures. The result is a GPU that renders 3D worlds without a single binary transistor, staying true to the Transone philosophy.

The Transone GPU, re-architected as an ultra-scaled chiplet module, maximises every physical dimension—yield, thermal stability, power integrity, and parallel throughput—while remaining faithful to the four‑state analog computation paradigm. The design escalates from four chiplets to a **tileable 16‑chiplet platform**, introduces 3D‑stacked analog cache, integrates an active silicon photonic interposer, and employs dynamic viewport slicing at the rasteriser level. This is the definitive high‑performance implementation.

---

## 1. Multi‑Chiplet Topology: 16 Compute Dies in a 4×4 Array

Instead of 4 dies of 4,096 PEs each, the GPU is composed of **16 identical Compute Chiplets** arranged in a 4×4 grid. Each chiplet contains a 32×32 grid of **1,024 Processing Elements**, for a total of **16,384 PEs** across the package. The smaller die size (approx. 200 mm² at 5 nm) pushes yield above 90%. Even if several chiplets per wafer are defective, the modular array is assembled from known‑good dies only—zero bad chiplets are placed.

**Quad‑chiplet redundancy:** For yield resilience, each 4×4 array includes **4 spare chiplets** (one per row) that remain dark during normal operation. If a chiplet fails post‑packaging, integrated self‑test hardware remaps its address space to the spare in under 100 ns, preserving the full 16,384‑PE logical GPU. The user never sees a defective unit.

---

## 2. Active Silicon Interposer with Integrated ANoC Routers

A passive interposer is replaced by an **Active Logic Interposer** fabricated on a mature 12 nm process. This interposer hosts:

- **ANoC Crossbar Switches:** At each chiplet boundary, dedicated analog pass‑transistor switch matrices sit inside the interposer. The long horizontal and vertical ANoC lines are broken into segments, and active repeaters (ultra‑fast unity‑gain buffers) refresh the voltage states every 2 mm, completely eliminating resistive droop. The four legal voltages (−1 V, 0 V, 1 V, 2 V) are buffered with precision chopper‑stabilised amplifiers that maintain accuracy within 0.05% across the entire interposer.
- **Distributed OptimusP Translators:** Rather than a single large bridge chip, each chiplet’s interposer footprint includes a **Micro‑OptimusP** block. These handle voltage‑to‑bit and bit‑to‑voltage conversion for their local PEs, enabling simultaneous binary‑to‑analog streaming at 16× bandwidth. The binary RISC‑V cores communicate with the GPU through an aggregated 4096‑bit parallel interface, all synchronised via a mesh‑synchronous clocking scheme.
- **Power Delivery Network (PDN):** The interposer contains a dense grid of through‑silicon vias (TSVs) and deep‑trench capacitors that deliver the four master rails directly under every chiplet. Each PE sees a local voltage regulation module that compensates transient loads in under 10 ns, eliminating voltage droop entirely.

---

## 3. 3D‑Stacked Analog Texture and Vertex Cache

Memory bandwidth is the ultimate bottleneck. The solution is **3D stacking of high‑density analog DRAM** directly on top of each Compute Chiplet using hybrid bonding.

- **Texture Cache Cube:** Each chiplet receives a dedicated 64 MB analog DRAM die, organised as 4,096 columns of capacitive storage. The DRAM cells store texel colours as multi‑wire voltage clusters (RGBA, each channel 0–9). A 128‑bit wide through‑silicon via bus reads 16 texels per cycle directly into the PE’s texture unit with sub‑3 ns latency.
- **Vertex/Constant Cache:** A separate 16 MB die holds vertex attributes and shader constants in the same multi‑column format. The analog rasteriser fetches transformed vertices from this cache without crossing the interposer, keeping localised data entirely within the chiplet stack.

The 3D integration effectively gives each chiplet over 80 MB of on‑chip analog storage, reducing external memory traffic by 90% and allowing the ANoC to be used exclusively for inter‑chiplet data coherence and screen‑space traversal.

---

## 4. Dynamic Viewport Partitioning and Load Balancing

The static quadrant assignment is replaced by a **micro‑tile dynamic scheduler**. The rasteriser is split into a two‑level hierarchy:

- **Global Command Processor:** Receives draw calls from the binary CPU via the OptimusP and decomposes them into 32×32‑pixel micro‑tiles. Each micro‑tile is tagged with a minimal bounding box and edge equations.
- **Tile Scheduler (on the interposer):** A low‑latency asynchronous arbiter assigns micro‑tiles to chiplets based on current load. It uses a tiny amount of binary logic (acceptable for control) to track each chiplet’s queue depth. The tile’s vertex data is then multicast only to the assigned chiplet via the interposer’s ANoC crossbar, ensuring no chiplet is starved or overwhelmed.

For ray tracing, the same scheduler distributes rays by screen tile, but for secondary rays (reflections/refractions), a coherence‑aware bucketing algorithm groups rays by direction and dispatches them to the same chiplet where the primary hit occurred, maximising BVH cache locality inside the 3D‑stacked memory.

---

## 5. Thermal Architecture: Immersion Cooling with Micro‑channel Interposer

The thermal challenge of 16,384 op‑amps is addressed with an **immersion‑compatible package**. The entire chiplet module is housed in a hermetically sealed ceramic lid with micro‑fluidic channels etched into the back of each die. A dielectric cooling fluid (e.g., 3M Fluorinert) flows directly over the backside of the compute chiplets and the interposer, removing heat at a rate of 1 kW/cm². The fluid is pumped through a closed loop with a heat exchanger.

Additionally, the interposer itself contains a network of tiny channels filled with the same coolant, placed strategically between the ANoC switch blocks. This dual‑sided cooling keeps junction temperature under 60°C, preventing the precision resistor drift that would threaten the 1 V and 2 V thresholds. The result: stable 20 MHz clock indefinitely under full load.

---

## 6. Enhanced Ray Acceleration Unit (RAU) Clusters

Within each chiplet, the 1,024 PEs are organised into **64 Ray Tracing Clusters** of 16 PEs each. Each cluster has a dedicated **BVH Traversal Engine** implemented as an analog state machine on the interposer, using a small content‑addressable memory stack built from capacitive cells. When a ray is scheduled, the cluster’s 16 PEs perform intersection tests on 16 different triangles simultaneously (one per PE) using the Möller–Trumbore analog pipelines. The traversal engine merges the “hit” voltages and decides the next BVH node to visit, all in under 100 ns.

With 16 chiplets × 64 clusters = 1,024 traversal engines, the GPU can test 1,024 rays against 16,384 triangles every 100 ns, yielding a raw intersection rate of over **1.6 trillion ray‑triangle tests per second**. Real‑time path tracing at 4K resolution becomes not only possible but comfortable.

---

## 7. Chiplet‑Level Power Gating and Clocking

Each chiplet can be independently power‑gated via the interposer’s power delivery network. When a viewport quadrant is entirely sky or a draw call only affects a subset of the screen, entire chiplets are powered off in less than one microsecond, eliminating static op‑amp current. The interposer’s active ANoC routers isolate the disabled chiplet’s bus lines to prevent leakage.

Clocking is distributed as a precise 20 MHz analog sine wave through the interposer, buffered by injection‑locked amplifiers at each chiplet. All 16 chiplets operate in perfect phase synchronisation, ensuring the ANoC voltage signals align exactly when crossing chiplet boundaries.

---

## 8. Manufacturing and Assembly

The 16‑chiplet GPU is manufactured as known‑good dies on 5 nm, tested at speed on a wafer‑level probe station that measures op‑amp settling times and comparator thresholds. Only dies passing a strict analog BIST (built‑in self‑test) are singulated. Assembly is performed by a chip‑on‑wafer process: the 16 compute dies are hybrid‑bonded to the active interposer wafer, then the 3D‑stacked analog DRAM dies are bonded on top. Finally, the interposer wafer is diced, and the complete module is attached to the ceramic substrate and fluidic heat spreader.

The result is a single socket‑compatible GPU package, measuring 65 mm × 65 mm, that contains 16,384 analog processing elements, 16 Micro‑OptimusP bridges, 1 GB of 3D‑stacked analog cache, and an active photonic‑ready interposer—all operating purely on the four voltage states −1 V, 0 V, 1 V, 2 V.

This maximised design removes every physical bottleneck of the original monolithic idea, delivering uncompromised yield, thermal headroom, and compute density while completely preserving the Transone philosophy. The chiplet GPU is no longer a compromise; it is the definitive physical form of the architecture.