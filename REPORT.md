# OSTEP Virtual Memory Lab Report Draft

## Setup Tasks
### Command(s)
- `python --version`
- `mkdir -p logs`
- `ls vm-paging`
- `ls vm-segmentation`

### Output snippet (short)
- `Python 3.10.19`
- `vm-paging: README.md, paging-linear-translate.py`
- `vm-segmentation: README.md, segmentation.py`
- Full log: `logs/setup_20260215_220106.txt`

### Interpretation
Python is available and both required simulators exist in the expected directories.

### Key takeaway
Environment is ready; all later claims are based on real simulator runs captured in `logs/`.

---

## Part 1 — Paging (`vm-paging`)

## Task 1 — Page Table Size Analysis (vary address space size)
### Command(s)
1. `python paging-linear-translate.py -P 1k -a 1m -p 512m -v -n 0`
2. `python paging-linear-translate.py -P 1k -a 2m -p 512m -v -n 0`
3. `python paging-linear-translate.py -P 1k -a 4m -p 512m -v -n 0`
- Full log: `logs/paging_task1_20260215_220113.txt`

### Output snippet (short)
- Run 1 shows 1024 page-table entries (VPN 0..1023).
- Run 2 shows 2048 page-table entries (VPN 0..2047).
- Run 3 shows 4096 page-table entries (VPN 0..4095).

### Interpretation
Using linear page tables:
- `#VPNs = address_space_size / page_size`
- `page_table_size = #VPNs * PTE_size`
- Assuming 4-byte PTE:
  - 1 MiB / 1 KiB = 1024 VPNs → `1024 * 4 = 4096 B = 4 KiB`
  - 2 MiB / 1 KiB = 2048 VPNs → `2048 * 4 = 8192 B = 8 KiB`
  - 4 MiB / 1 KiB = 4096 VPNs → `4096 * 4 = 16384 B = 16 KiB`

This directly reflects the paging chapter concept: linear tables reserve one PTE per virtual page, so table size scales with virtual-space pages, not with how many pages are actually used.

### Results analysis
Doubling address space doubled the VPN count and table size each time (4 KiB → 8 KiB → 16 KiB). This demonstrates why linear page tables become expensive for large sparse spaces: allocation sparsity does not reduce table footprint.

### Key takeaway
Linear page table memory overhead is driven by `(virtual address space / page size)`, even for unmapped pages.

---

## Task 2 — Page Allocation Experiment (vary `-u`)
### Command(s)
1. `python paging-linear-translate.py -P 1k -a 16k -p 32k -v -u 0`
2. `python paging-linear-translate.py -P 1k -a 16k -p 32k -v -u 25`
3. `python paging-linear-translate.py -P 1k -a 16k -p 32k -v -u 50`
4. `python paging-linear-translate.py -P 1k -a 16k -p 32k -v -u 100`
5. (for computed outcomes) same commands with `-c`
- Full logs: `logs/paging_task2_20260215_220122.txt`, `logs/paging_task2_answers_20260215_220134.txt`

### Output snippet (short)
From computed runs (`-c`), with 16 VPN total:
- `u=0`: valid PTEs = 0/16; hits = 0, invalid = 5
- `u=25`: valid PTEs = 6/16; hits = 1, invalid = 4
- `u=50`: valid PTEs = 9/16; hits = 3, invalid = 2
- `u=100`: valid PTEs = 16/16; hits = 5, invalid = 0

### Interpretation
Higher `-u` marks more PTEs valid, so random VA traces are more likely to hit mapped VPNs. At `u=0`, every lookup faults (`VPN not valid`). At `u=100`, all trace addresses translate successfully.

### Results analysis
The valid bit is the key sparsity mechanism: many entries can exist but be invalid. This preserves fixed table size while allowing sparse mappings. As occupancy rises, translation success rises monotonically.

### Key takeaway
`-u` changes mapping density (valid bits), not table size; sparse tables still consume full linear-table memory.

---

## Task 3 — Random Seed Experiments (realistic vs unrealistic)
### Command(s)
1. `python paging-linear-translate.py -P 8 -a 32 -p 1024 -v -s 1 -c`
2. `python paging-linear-translate.py -P 8k -a 32k -p 1m -v -s 2 -c`
3. `python paging-linear-translate.py -P 1m -a 256m -p 512m -v -s 3 -c`
- Full log: `logs/paging_task3_20260215_220143.txt`

### Output snippet (short)
- Case 1: 4 VPN total (page size 8 bytes), mostly invalid trace results.
- Case 2: 4 VPN total (8 KiB pages in 32 KiB VA), coarse but plausible for toy examples.
- Case 3: 256 VPN total with 1 MiB pages; mixed valid/invalid as expected.

### Interpretation
Realism assessment:
- `P=8 bytes`: unrealistic in modern systems (page granularity far too tiny; metadata and translation overhead would explode).
- `P=8 KiB`: reasonable teaching-scale parameter.
- `P=1 MiB`: generally unrealistic as a default page size (internal fragmentation risk, coarse mapping), though large pages exist as optional features.

### Results analysis
The simulator clearly shows mechanics are unchanged across scales (valid bit + PFN lookup), but engineering practicality changes dramatically with page size due to fragmentation and metadata trade-offs.

### Key takeaway
Same translation logic works for all parameters, but practical systems pick page sizes balancing table overhead, TLB reach, and fragmentation.

---

## Task 4 — Challenge: Explore Paging Limits
### Command(s)
1. `python paging-linear-translate.py -P 4k -a 64m -p 16m -v -s 4 -c`
2. `python paging-linear-translate.py -P 1 -a 64k -p 1m -v -s 5 -n 3 -c`
3. `python paging-linear-translate.py -P 16m -a 64m -p 1g -v -s 6 -c`
4. `python paging-linear-translate.py -P 4k -a 4g -p 512m -v -s 7 -n 2 -c`
- Full log: `logs/paging_task4_20260215_220152.txt`

### Output snippet (short)
- Run 1: `Error: physical memory size must be GREATER than address space size (for this simulation)`
- Run 2: extremely large page table printed (65,536 entries for 64 KiB VA with 1-byte pages).
- Run 3: `Error: must use smaller sizes (less than 1 GB) for this simulation.`
- Run 4: same physical-vs-virtual constraint error as Run 1.

### Interpretation
What becomes impractical/breaks:
- Address space > physical memory is rejected by this simulator model (even though real demand-paged OSes allow it).
- Tiny page size (1 byte) causes absurd linear page table overhead: 65,536 PTEs for only 64 KiB VA; with 4-byte PTEs that is ~256 KiB table for a 64 KiB space.
- Very large parameter values hit simulator guardrails (<1 GiB constraints).

### Results analysis
The extremes confirm two limits: (1) modeling limits in this specific simulator and (2) fundamental linear-page-table scaling costs for tiny pages / large VPN counts. Translation remains conceptually simple, but metadata cost dominates quickly.

### Key takeaway
Paging design is a balancing act: very small pages inflate metadata, very large configurations hit implementation/practical limits.

---

## Part 2 — Segmentation (`vm-segmentation`)

## Task 1 — Address Translation with Segmentation
### Command(s)
1. `python segmentation.py -a 128 -p 512 -b 0 -l 20 -B 512 -L 20 -s 0`
2. `python segmentation.py -a 128 -p 512 -b 0 -l 20 -B 512 -L 20 -s 1`
3. `python segmentation.py -a 128 -p 512 -b 0 -l 20 -B 512 -L 20 -s 2`
4. (for computed answers) same commands with `-c`
- Full logs: `logs/seg_task1_20260215_220203.txt`, `logs/seg_task1_answers_20260215_220209.txt`

### Output snippet (short)
- Example valid SEG0: `VA 0x11 -> VALID in SEG0: PA 0x11`
- Example valid SEG1: `VA 0x6c -> VALID in SEG1: PA 0x1ec`
- Several other VAs produce `SEGMENTATION VIOLATION`.

### Interpretation
With 128-byte VA space, top bit selects segment:
- `VA 0..63` → SEG0, offset = `VA`
- `VA 64..127` → SEG1, negative-growth offset effectively `VA - 64` against SEG1 limit; PA computed from base downward.
Validity requires offset < segment limit. If valid: PA = base ± offset (direction depends on segment growth).

### Results analysis
Small limits (`20`) make most random addresses illegal. This clearly illustrates segmentation’s explicit bounds checking unlike pure linear-page-table lookup.

### Key takeaway
Segmentation adds protection via per-segment base/limit checks before translation.

---

## Task 2 — Segmentation Address Space Limits + Verification
### Command(s)
- Boundary computation from parameters (`a=128, l0=20, l1=20`)
- Verification command:
  - `python segmentation.py -a 128 -p 512 -b 0 -l 20 -B 512 -L 20 -A 19,108,20,107 -c`
- Full log: `logs/seg_task2_20260215_220219.txt`

### Output snippet (short)
- `19 -> VALID in SEG0`
- `108 -> VALID in SEG1`
- `20 -> SEGMENTATION VIOLATION (SEG0)`
- `107 -> SEGMENTATION VIOLATION (SEG1)`

### Interpretation
Computed boundaries:
- Highest legal VA in SEG0: **19**
- Lowest legal VA in SEG1: **108**
- Lowest illegal VA overall: **20**
- Highest illegal VA overall: **107**

### Results analysis
Verification mode confirms precise boundary behavior: one-off changes across limits immediately switch valid ↔ invalid.

### Key takeaway
Segmentation legality is interval-based and sharply discontinuous at segment bounds.

---

## Task 3 — “No valid addresses” scenario
### Command(s)
- `python segmentation.py -a 128 -p 512 -b 0 -l 0 -B 512 -L 0 -A 0,10,63,64,100,127 -c`
- Full log: `logs/seg_task3_20260215_220224.txt`

### Output snippet (short)
All tested addresses report `SEGMENTATION VIOLATION`.

### Interpretation
Setting both limits to 0 (`l0=0`, `l1=0`) yields empty valid ranges for both segments; no code edits needed.

### Results analysis
This demonstrates segmentation can represent an intentionally inaccessible process address space through register configuration alone.

### Key takeaway
Base/limit parameters are sufficient to create total denial of valid virtual addresses.

---

## Screenshot Checklist (what to capture)
1. **Setup proof**: terminal showing setup commands and `logs/setup_...txt` in file tree.
2. **Paging Task 1**: terminal at start of each run + visible page-table range (show first/last entry indices) and the `paging_task1_...txt` log file highlighted.
3. **Paging Task 2**: terminal showing each `-u` case with computed (`-c`) outputs that include both valid translations and invalid VPNs.
4. **Paging Task 3**: terminal showing all three parameter sets and outputs, especially tiny-page and huge-page-size examples.
5. **Paging Task 4**: terminal showing simulator error lines for extreme invalid configs and one huge-output run (`-P 1`) proving massive table.
6. **Seg Task 1**: terminal showing segmentation traces for seeds 0/1/2 (prefer `-c` screen for explicit validity).
7. **Seg Task 2**: terminal showing boundary verification command (`-A 19,108,20,107 -c`) and results.
8. **Seg Task 3**: terminal showing zero-limit configuration and all violations.
9. **Final artifacts**: file tree screenshot of `logs/` and `REPORT.md` together.
