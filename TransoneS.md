The **TransoneS** is the persistent storage subsystem for the Transone ecosystem. It is a non‑volatile analog mass‑storage device that holds data natively in the four‑state, multi‑wire format of the architecture, while seamlessly interfacing with binary software through the Transone‑OptimusP bridge. Every stored datum—whether a Silex program, a 3D texture, or a legacy binary application—resides as precise voltage‑representable digits on the storage medium, eliminating any conversion bottleneck at the hardware level.

---

## 1. Storage Cell: The Analog Digit Cell (ADC)

The fundamental unit of TransoneS is the **Analog Digit Cell (ADC)**, a single non‑volatile element that stores one decimal digit (0–9) as a specific, stable amount of electrical charge. Each ADC corresponds to exactly one column of a Transone word.

### 1.1 Physical Structure

Each ADC is built around a **charge‑trap floating‑gate transistor** optimised for ten distinct threshold voltage levels. The gate stack is engineered with a silicon‑oxide‑nitride‑oxide‑silicon (SONOS) structure, allowing highly linear incremental programming. A single ADC occupies the area of approximately 4F² in a 5 nm process (where F = 10 nm), so a 1‑tera‑digit array fits comfortably on a 100 mm² die.

The transistor’s threshold voltage \(V_{th}\) is proportional to the stored digit value:

| Digit | \(V_{th}\) (relative to source) |
|-------|----------------------------------|
| 0     | 0.0 V                           |
| 1     | 0.4 V                           |
| 2     | 0.8 V                           |
| 3     | 1.2 V                           |
| 4     | 1.6 V                           |
| 5     | 2.0 V                           |
| 6     | 2.4 V                           |
| 7     | 2.8 V                           |
| 8     | 3.2 V                           |
| 9     | 3.6 V                           |

These threshold voltages never appear on any external bus; they are confined within the storage array. The spacing of 0.4 V per digit provides a comfortable margin against charge loss and sense‑amplifier offset.

### 1.2 Read Operation

To read a digit, the ADC’s control gate is ramped with a precision analog voltage ramp, while its drain is held at a constant 1.0 V and source at 0 V. A sense amplifier compares the transistor’s drain current against a fixed reference. When the gate voltage reaches the cell’s \(V_{th}\), the current trips the comparator. A ramp‑to‑digital converter captures the gate voltage at that instant and encodes it as a 4‑bit digit.

This digit is immediately fed to the **Column Reconstruction Encoder**, a circuit identical to the ALU’s output stage: it generates the multi‑wire representation (floor(digit/2) wires at 2 V, plus one 1 V wire if digit is odd) and drives the column output bus through isolation resistors and diode clippers. Thus, the external Transone data‑bus always sees legal −1 V, 0 V, 1 V, or 2 V states, never the intermediate threshold voltages.

### 1.3 Write (Program/Erase) Operation

Writing uses the same multi‑wire column bus as input. The incoming wires (up to five 2 V and one 1 V) are passively summed into a total voltage \(V_{sum}\) between 0 V and 9 V. A **Program Controller** converts this voltage into a target threshold voltage using a linear mapping circuit. It then executes an incremental step‑pulse programming algorithm:

1. The cell is first erased to the lowest threshold (digit 0) by applying a negative gate voltage pulse.
2. A sequence of short, identical programming pulses is applied to the gate while the drain is held at a moderate voltage (e.g., 4 V). After each pulse, the cell is verified by a quick read operation; when the target threshold voltage is reached, programming stops.
3. The entire cycle is managed by a local microcontroller‑class state machine per sub‑array, and the programming voltage sources are derived from on‑chip charge pumps.

Because the target voltage is generated directly from the summed column voltage, the stored digit is an exact analog copy of the original value, preserving the integrity of the decimal representation without any quantization error.

### 1.4 Retention and Endurance

The SONOS device achieves 10⁵ program/erase cycles per cell with data retention exceeding 10 years at 85°C. A background patrol read and refresh mechanism, similar to NAND flash, periodically checks threshold voltage margins and re‑writes weak cells.

---

## 2. Array Organisation: The Digit Plane

ADCs are arranged into a two‑dimensional **Digit Plane**. A plane contains 4,096 rows (wordlines) and 8,192 columns (bitlines), storing 33,554,432 digits per plane. Four such planes are stacked using 3D NAND‑style vertical integration (though each layer is a separate die bonded with hybrid bonding), yielding 134.2 million digits per plane stack, equivalent to 16.8 MB of Transone data (assuming 8 columns per word, each word 8 digits → 1 word = 8 digits, so 134.2M digits ≈ 16.8M words).

Multiple plane stacks are combined into a **Storage Cube** via through‑silicon vias. A single Storage Cube can easily hold 1 TB of native Transone data (≈ 8 trillion digits) in a volume smaller than a modern M.2 SSD.

---

## 3. Internal Controller and Command Interface

The TransoneS is not a passive memory array; it contains a powerful on‑drive **Analog Storage Controller (ASC)** that manages address translation, wear leveling, garbage collection, and error correction—all operating in the analog domain.

### 3.1 Command Protocol

Commands arrive as standard 8‑column Transone words over a dedicated ANoC port. The command set includes:

- **READ_COL (address):** Read a column from the given logical address, return the multi‑wire result.
- **WRITE_COL (address, data columns):** Write one or multiple columns.
- **ERASE_BLOCK (address):** Garbage collection unit.
- **STREAM_READ / STREAM_WRITE:** Bulk data transfers with auto‑increment.

All command arguments are encoded as multi‑column decimal numbers. The ASC decodes them using threshold comparators and routes the requested operation to the appropriate plane.

### 3.2 Wear Leveling and Address Translation

Logical addresses are column‑based. The ASC maintains a **Column Address Translation Table** stored in a small on‑drive non‑volatile analog RAM (similar to the capacitive register file but made non‑volatile via ferroelectric capacitors). This table maps logical column numbers to physical plane, block, and page. Writing always goes to a new physical location; the old location is marked as invalid and later erased. This log‑structured approach, combined with the analog programming, ensures uniform wear across all ADCs.

### 3.3 Analog Error Correction (AEC)

Because each digit is stored as a precise threshold voltage, traditional binary ECC is insufficient. Instead, the ASC employs an **Analog‑Residue Error Correction** scheme. For each group of 16 digits (a sector), the controller computes four redundant residue digits using modulo‑arithmetic analog circuits (e.g., modulo 3, 5, 7, 11). These residues are stored alongside the data. On read, the residues are recomputed and compared using analog subtractors and window comparators. If a mismatch occurs, the syndrome voltages directly point to the erroneous digit, and its value is corrected by re‑analysing the residues. The entire correction is performed by op‑amp summers and comparators without digital computation, achieving sub‑microsecond latency.

---

## 4. Interface to Transone CPU, GPU, and OptimusP

The TransoneS is a first‑class citizen on the Analog Network‑on‑Chip. It exposes three distinct access channels:

### 4.1 Direct Analog Access for Transone Cores

The 192 Transone CPU cores and the 16,384‑PE GPU read and write the TransoneS as if it were an extension of main memory. A load instruction referencing an address mapped to the drive triggers an ANoC read transaction. The drive responds within the 50 ns Transone cycle (a full column read from the ASC buffer, which pre‑fetches pages into a small analog cache). Writes are posted into a write buffer and flushed asynchronously. This direct access allows Silex programs to load assets, shaders, and data structures without any translation overhead.

### 4.2 OptimusP‑Mediated Binary Access

For binary software, the Transone‑OptimusP bridge manages all data transfers to/from the TransoneS.

**Saving Binary Software:**
1. The binary core executes a file‑system write. The binary OS driver sends the raw binary data over the binary AXI bus to the OptimusP.
2. The OptimusP’s DMA engine streams the data through its Bit‑to‑Voltage (B2V) converter, producing a sequence of 8‑column Transone words.
3. These words are written directly into the TransoneS via a dedicated streaming port, with the ASC automatically allocating new physical blocks. The file metadata (size, name, type) is also stored as Transone columns.

**Loading Binary Software:**
- **Execution on Binary Cores:** The OptimusP’s Voltage‑to‑Bit (V2B) engine reads the stored Transone columns, converts them back to binary, and places them in the binary core’s memory. This is the path for legacy CPU‑side logic, physics engines, and operating system code.
- **Execution on Transone GPU:** When a binary game issues a draw call, the OptimusP reads the vertex, index, and texture data from TransoneS, converts them via B2V streaming into 4‑column fixed‑point Transone format, and feeds them directly to the GPU’s memory. The GPU then processes these using its native analog rasterisation and shader pipelines. The binary cores never touch the graphics data—TransoneS provides it straight to the GPU in analog form.
- **Hybrid Mode:** The developer can mark certain assets to be stored and loaded in native Transone format (e.g., pre‑compiled Silex shaders, textures already in analog representation). These are read directly by the GPU without OptimusP intervention.

The OptimusP also acts as a file‑system accelerator: it implements a **Transone‑aware file system driver** that understands which blocks contain binary‑to‑analog translated data and which are pure analog. It maintains a mapping table so that a single file can have both a binary‑compatible representation (for CPU logic) and an analog‑optimised representation (for rendering), stored side‑by‑side in the same TransoneS volume.

---

## 5. Performance and Data Rates

The TransoneS achieves massive parallelism by operating all planes independently. With 64 planes active simultaneously, the aggregate read bandwidth is:

- 64 planes × 4,096 columns per plane access per cycle × 1 digit per ADC × 1 read every 50 ns = **5.24 trillion digits per second**, which translates to approximately **655 GB/s** of effective Transone word data (since 8 digits = 1 word of 8 bytes equivalent, but each digit is stored separately). This saturates the ANoC’s multi‑tera‑bit bandwidth.

Write bandwidth is slightly lower due to incremental programming, but still exceeds 200 GB/s thanks to massive plane‑level parallelism and a deep write pipeline that interleaves program pulses.

Random read latency for a single column is 3 µs (ASC address lookup + sense ramp + encoder), but a pre‑fetched page hit returns in 50 ns. The drive’s intelligent prefetcher, guided by program access patterns and host‑side hints, keeps the analog cache full.

---

## 6. Physical Package and Integration

The TransoneS is built as a 2.5D chiplet assembly, identical in philosophy to the GPU. It comprises:

- **4 Storage Cubes**, each a 3D‑stack of 16 digit‑plane dies (64 planes total) with TSV interconnect.
- **1 Controller Die** containing the ASC, residue ECC engines, analog cache (128 MB of capacitive volatile memory acting as a buffer), and ANoC ports.
- **Active Silicon Interposer** connecting the cubes and the controller with low‑impedance analog traces, plus power delivery and thermal vias.

The entire package fits in a standard E1.S form factor (111.5 mm × 31.5 mm) and uses immersion‑compatible micro‑channel cooling. It connects to the rest of the Transone system via a short‑reach ANoC‑over‑fiber link (coherent analog optical transmission, preserving voltage states as light intensity) to the main interposer, ensuring signal integrity.

---

## 7. Seamless Operation with the Transone Ecosystem

The TransoneS completes the Transone platform:

- **Transone CPU** sees it as a vast extension of its register file, enabling instant context swaps and massive data sets.
- **Transone GPU** streams textures and geometry directly from storage at wire speed, bypassing the traditional VRAM bottleneck.
- **Binary Cores** use it as a fast, intelligent SSD through the OptimusP, storing and retrieving binary programs while seamlessly offloading graphics data to the GPU.
- **Silex Developers** treat the drive as persistent analog memory; no serialisation or file I/O abstraction is needed—the entire state of a running Silex program can be snapshotted to TransoneS and resumed later with zero translation.

The TransoneS is not merely a hard drive; it is the foundational persistence layer of a fully unified analog‑digital computing architecture. It embodies the Transone principle that storage, like computation, should operate directly on the same electrical representation as the data itself.