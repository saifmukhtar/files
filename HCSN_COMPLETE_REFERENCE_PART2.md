# HCSN Complete Reference Guide — Part 2
## Code Architecture + File-by-File Walkthrough

---

# PART 2: THE CODE ARCHITECTURE

## Chapter 8: Why Rust? Why Not Python?

The original simulation was written in Python. It was migrated to Rust for reasons that are core to the science.

### The Python Problem

Python is an interpreted language. Every line of code goes through a Python interpreter before reaching the CPU. For a simulation that needs to run **250,000 steps**, each step involving:
- O(N²) causal relationship checks
- Bitset union operations across thousands of vertices
- Hash table lookups at every step

...Python became a bottleneck. A simulation that takes 1 second in Rust might take 100 seconds in Python.

### Why Rust Specifically

| Feature | Benefit for HCSN |
|---|---|
| **Zero-cost abstractions** | High-level code compiles to machine code with no runtime overhead |
| **No garbage collector** | Memory is managed at compile time — no GC pauses during simulation |
| **Ownership system** | Eliminates entire classes of bugs (use-after-free, data races) |
| **Rayon parallelism** | Trivially parallelize across CPU cores |
| **FixedBitSet** | Hardware-native bitwise operations for causal set intersection |
| **`--release` flag** | Enables full compiler optimizations (LLVM) |

### The Speed Improvement

The migration from Python to Rust gave roughly **50-100× speedup** for the same simulation. This is not just a performance tweak — it makes previously impossible experiments feasible. A 250,000-step run that took hours in Python takes minutes in Rust.

---

## Chapter 9: Project Structure — What Each File Does

```
hcsn-rust/
│
├── Cargo.toml         ← Build manifest. Lists all dependencies and binary targets.
├── src/
│   ├── lib.rs         ← Gateway: declares all 6 modules
│   ├── main.rs        ← Default entry point: dual-core parallel run
│   ├── hypergraph.rs  ← The Universe: data structures for the causal network
│   ├── physics_params.rs ← Configuration: all tunable parameters
│   ├── rules.rs       ← The Laws: the two fundamental rewrite operations
│   ├── observables.rs ← The Instruments: everything we measure
│   ├── rewrite_engine.rs ← The Engine: the main simulation loop (1912 lines)
│   ├── persistence.rs ← The Recorder: saves data to CSV files
│   └── bin/           ← Experiment binaries (run_simulation, force_law_*, etc.)
```

---

## Chapter 10: `lib.rs` — The Gateway (7 lines)

```rust
pub mod hypergraph;
pub mod observables;
pub mod persistence;
pub mod physics_params;
pub mod rewrite_engine;
pub mod rules;
```

**What this does:** Declares all 6 modules as public, making them accessible from both `main.rs` and the `bin/` executables.

**In Rust, `pub mod X` means:** "Tell the compiler there is a module called X, and make all its public items accessible to the outside world."

**Why it matters:** Without this file, the individual source files are isolated. This file creates the "library" that all the binary executables can use. Think of it as the index of a book.

---

## Chapter 11: `hypergraph.rs` — The Universe (308 lines)

This file defines the **fundamental data structure** of HCSN. Everything else operates on what this file creates.

### 11.1 Global Counters

```rust
static VERTEX_ID_COUNTER: AtomicU64 = AtomicU64::new(0);
static EDGE_ID_COUNTER: AtomicU64 = AtomicU64::new(0);
```

**Line-by-line:**
- `static` = lives for the entire duration of the program
- `AtomicU64` = a 64-bit integer that can be safely incremented from multiple threads simultaneously without data races
- `AtomicU64::new(0)` = starts counting from 0

**Why atomic?** The simulation uses multiple threads (Rayon). If two threads create vertices at the same time and both read/write the same counter with a regular integer, you get a **race condition** where both get the same ID. `AtomicU64::fetch_add(1, Ordering::SeqCst)` is guaranteed to give each thread a unique, different number.

**Why important for the theory?** Every vertex in HCSN has a globally unique ID. This ID is permanent and never reused. This is what makes it possible to track causal relationships correctly.

### 11.2 The `Vertex` Struct

```rust
#[derive(Clone, Debug)]
pub struct Vertex {
    pub id: u64,        // Unique permanent identifier
    pub depth: usize,   // Causal depth — how far from the initial state
    pub label: i32,     // +1 or -1, a charge-like property
    pub parents: Vec<u64>,   // Vertices this one causally follows from
    pub children: Vec<u64>,  // Vertices that causally follow from this one
}
```

**What each field means:**

- **`id`**: The unique name of this event. Never changes, never reused.
- **`depth`**: How many causal steps from the "beginning." This is HCSN's definition of time — not a clock, but a depth in the causal DAG.
- **`label`**: Randomly assigned +1 or -1 at creation. This is a discrete "charge-like" property. Hyperedges where vertices have mixed labels are "frustrated" — this is how we measure defect density.
- **`parents` / `children`**: Direct 1-hop causal neighbors. Used for fast adjacency traversal.

**`impl Vertex::new()`:**
```rust
pub fn new() -> Self {
    let mut rng = rand::thread_rng();
    let label = if rng.gen_bool(0.5) { 1 } else { -1 };
    Self {
        id: VERTEX_ID_COUNTER.fetch_add(1, Ordering::SeqCst),
        depth: 1,
        label,
        parents: Vec::new(),
        children: Vec::new(),
    }
}
```
- Gets a new random number generator (thread-local, fast)
- Assigns label randomly (+1 or -1 with 50% probability each)
- Takes the next available ID atomically
- Starts at depth 1 (gets updated when causal relations are added)

### 11.3 The `Hyperedge` Struct

```rust
#[derive(Clone, Debug)]
pub struct Hyperedge {
    pub id: u64,
    pub vertices: Vec<u64>,  // IDs of the vertices this edge groups
}
```

A hyperedge is a **group of vertices** that are causally related to each other. Unlike regular graph edges (which connect exactly 2 nodes), hyperedges can connect any number.

**Theory connection:** In the axioms, hyperedges represent causal relations between events. A hyperedge {A, B, C} means events A, B, and C are causally entangled.

### 11.4 The `Hypergraph` Struct

```rust
#[derive(Clone, Debug)]
pub struct Hypergraph {
    pub vertices: HashMap<u64, Vertex>,
    pub hyperedges: HashMap<u64, Hyperedge>,
    pub causal_future: HashMap<u64, FixedBitSet>,  // J+(v) for each vertex
    pub causal_past: HashMap<u64, FixedBitSet>,    // J-(v) for each vertex
}
```

**The four fields:**

1. **`vertices`**: All vertices, keyed by ID. O(1) lookup.
2. **`hyperedges`**: All hyperedges, keyed by ID. O(1) lookup.
3. **`causal_future`**: For each vertex v, a bitset where bit `i` is set if vertex `i` is in J⁺(v) (reachable from v causally). This is the transitive causal closure.
4. **`causal_past`**: For each vertex v, a bitset where bit `i` is set if vertex `i` is in J⁻(v) (causally precedes v).

**Why bitsets?** A `FixedBitSet` stores membership using individual bits. Checking if vertex 5000 is in J⁺(v) is a single bit-read operation — O(1). Computing the intersection J⁺(u) ∩ J⁻(v) (needed for causal interval size) is a bitwise AND — blazing fast on modern CPUs.

**Initial capacity 524,288:** Each bitset starts pre-allocated with space for 524,288 vertices. This avoids repeated re-allocation as the graph grows.

### 11.5 `add_causal_relation(u_id, v_id)` — The Most Critical Method

This adds a directed causal link u → v. But the hard part is maintaining **transitive closure** — if A → B and B → C, then A must also reach C.

```rust
pub fn add_causal_relation(&mut self, u_id: u64, v_id: u64) {
    // Guard: don't add if already related
    if self.is_causally_related(u_id, v_id) { return; }

    // Step 1: Add direct 1-hop adjacency
    // u.children gets v, v.parents gets u

    // Step 2: Transitive closure update
    // Get J-(u) = all vertices that can reach u
    let past_u = self.causal_past.get(&u_id).cloned();
    // Get J+(v) = all vertices that v can reach
    let future_v = self.causal_future.get(&v_id).cloned();

    // For every p in J-(u): p can now also reach everything in J+(v)
    for p_idx in past_u.ones() {
        let p_future = self.causal_future.get_mut(&(p_idx as u64));
        p_future.union_with(&future_v);  // Bitwise OR
    }

    // For every f in J+(v): everything in J-(u) can now reach f
    for f_idx in future_v.ones() {
        let f_past = self.causal_past.get_mut(&(f_idx as u64));
        f_past.union_with(&past_u);  // Bitwise OR
    }

    // Step 3: Update depth
    // v.depth = max(v.depth, u.depth + 1)
}
```

**Why this is expensive but necessary:** Every new causal link can affect ALL ancestors of u and ALL descendants of v. In the worst case this is O(N²). This is why the bitset optimization is critical — union_with is a single CPU instruction per 64 bits.

### 11.6 `scrub_ghost_bits()` — Memory Safety

```rust
pub fn scrub_ghost_bits(&mut self) {
    // Build a mask of all VALID (alive) vertex IDs
    let max_id = *self.vertices.keys().max().unwrap_or(&0);
    let mut mask = FixedBitSet::with_capacity(max_id as usize + 1);
    for &id in self.vertices.keys() {
        mask.insert(id as usize);
    }
    
    // Remove bitsets for deleted vertices
    self.causal_future.retain(|k, _| self.vertices.contains_key(k));
    self.causal_past.retain(|k, _| self.vertices.contains_key(k));
    
    // Intersect all bitsets with the valid mask
    // This removes references to deleted vertices
    for bs in self.causal_future.values_mut() {
        bs.intersect_with(&mask);
    }
    for bs in self.causal_past.values_mut() {
        bs.intersect_with(&mask);
    }
}
```

Called every 100 steps. When vertices are deleted (by the fusion rule), their bit positions in other vertices' causal sets become "ghost bits" — references to non-existent vertices. This function removes them. Without this, the simulation would accumulate stale memory and eventually crash or give wrong results.

---

## Chapter 12: `physics_params.rs` — The Configuration (80 lines)

All physics parameters are loaded from **environment variables** at startup. This is a crucial design decision.

```rust
pub struct PhysicsParams {
    pub gamma_defect: f64,          // HCSN_GAMMA_DEFECT, default 0.15
    pub inertia_scale: f64,         // HCSN_INERTIA_SCALE, default 1.0
    pub interaction_boost: f64,     // HCSN_INTERACTION_BOOST, default 1.02
    pub stability_decay: f64,       // HCSN_NU, default 0.975
    pub nonlinear_coupling: f64,    // HCSN_GAMMA, default 2.2
    pub memory_coupling: f64,       // HCSN_MU, default 0.3
    pub noise_bias: f64,            // hardcoded 0.0
    pub defect_injection: f64,      // HCSN_DEFECT_INJECTION, default 0.0
    pub geometry_freeze: f64,       // hardcoded 0.9
    pub enable_conservation_patches: bool,  // HCSN_PATCHES, default true
    pub export_mechanisms: bool,    // HCSN_EXPORT_MECHANISMS, default false
}
```

**Parameter meanings:**

| Parameter | Symbol | Role |
|---|---|---|
| `stability_decay` | ν = 0.975 | Per-step decay of stability memory. 0.975 means stability loses 2.5% each step |
| `nonlinear_coupling` | γ = 2.2 | Exponent in stability→suppression formula. Higher γ = sharper transition |
| `memory_coupling` | μ = 0.3 | How strongly accumulated stability suppresses rewrites |
| `interaction_boost` | 1.02 | Amplifies interaction forces slightly |
| `defect_injection` | 0.0 | If > 0: spontaneous vacuum nucleation (creates 4-vertex cliques randomly) |
| `enable_conservation_patches` | True | Enables Hybrid conservation mode |

**Why environment variables?** This allows running experiments with different parameters WITHOUT recompiling:
```bash
HCSN_GAMMA=3.0 HCSN_NU=0.95 cargo run --release
```
You can sweep parameters in a shell script. This is how the Phase 12 criticality scans were done.

---

## Chapter 13: `rules.rs` — The Laws of Physics (207 lines)

This file contains the only two fundamental operations in the simulation. Everything in HCSN emerges from these two rules.

### 13.1 The `UndoRecord` Struct

```rust
pub struct UndoRecord {
    pub target: Vec<u64>,                              // The anchor vertices
    pub added_vertices: Vec<u64>,                      // New vertices created
    pub added_edges: Vec<u64>,                         // New edges created
    pub added_causal: Vec<(u64, u64)>,                 // New causal links added
    pub removed_vertex: Option<Vertex>,                // Vertex that was deleted
    pub kept_vertex: Option<u64>,                      // Vertex that survived fusion
    pub removed_edges: HashMap<u64, Hyperedge>,        // Edges that were deleted
    pub old_causal_future: HashMap<u64, FixedBitSet>,  // Causal state before change
    pub old_causal_past: HashMap<u64, FixedBitSet>,    // Causal state before change
    pub old_parents: HashMap<u64, Vec<u64>>,           // Adjacency before change
    pub old_children: HashMap<u64, Vec<u64>>,          // Adjacency before change
}
```

**What this is for:** Every rewrite step records **exactly what changed** in this struct. If the step needs to be undone (for the TimeSymmetry conservation mode), the `execute_undo_record()` method in `hypergraph.rs` restores everything perfectly.

**Theory connection:** This is the HCSN equivalent of a Metropolis-Hastings rejection step. The simulation can "undo" high-error rewrites to enforce statistical conservation.

### 13.2 Rule 1: `edge_creation_rule()` — Birth

This is the GROWTH rule. It creates new vertices and connects them.

```
Algorithm:
1. Select a random hyperedge from the graph
   (optionally anchored to a specific vertex, for targeted growth)

2. Loop Closure (with probability p_rule, if ≥3 vertices exist):
   - Pick 2 random existing vertices u, v
   - If they are NOT already causally related:
     → Add causal link u → v
     → Add hyperedge {u, v}
   This creates LOOPS in the interaction graph (not causal loops!),
   which is essential for forming closed motifs (knots)

3. Create a new vertex

4. Connect it causally to ALL vertices in the selected edge:
   old_edge.vertices → new_vertex (causal direction)

5. Causal thickening (with probability 0.3 for each ancestor):
   For each ancestor of the edge vertices,
   also add a causal link to the new vertex.
   This creates temporal "thickness" — the new vertex sees the history.

6. Create new hyperedge = old_edge.vertices + new_vertex

7. Return UndoRecord
```

**Theory connection:** This is "spacetime growing." Each step creates a new event (vertex) that is causally downstream of existing events. This is how the universe evolves.

### 13.3 Rule 2: `vertex_fusion_rule()` — Contraction

This is the CONTRACTION rule. It merges two vertices into one.

```
Algorithm:
1. Requires: ≥3 vertices, ≥1 edge with 3+ vertices
   (Safety: fusion on a tiny graph is meaningless)

2. Select an edge with ≥3 vertices
3. Choose: v_keep (survives), v_remove (will be deleted)

4. Safety check: ensure v_remove is not the ONLY vertex in all edges
   (We can't delete the last copy of something)

5. Merge causal identity:
   → v_keep inherits ALL of v_remove's causal future/past (bitset OR)

6. Redirect all adjacency:
   → For each parent of v_remove: redirect to v_keep
   → For each child of v_remove: redirect to v_keep

7. Remove all edges containing v_remove

8. Delete v_remove from the graph

9. Return UndoRecord
```

**Theory connection:** This is "spacetime contracting." Two formerly distinct events become a single event. This creates the complex, non-tree topology that allows persistent knots to form.

---

## Chapter 14: `observables.rs` — The Instruments (580 lines)

This file computes everything we measure. It is the scientific instrument panel.

### 14.1 Key Measurement Functions

**`compute_omega(inter)`:**
```rust
// For each vertex v with degree ≥ 2:
//   Count edges between v's neighbors (internal)
//   Count possible edges = k*(k-1)/2
//   local_C = internal / possible
// Ω = mean(local_C)
```

**`myrheim_meyer_dimension(h, samples, min_interval)`:**
This estimates the **emergent spacetime dimension** from causal statistics.

Formula: `D = 2 × ln(N) / ln(avg_interval_size)`

This comes from causal set theory. If you randomly pick two causally related events, the volume between them (the causal interval) scales as `N^(D/2)`. By measuring interval sizes statistically, you can infer D.

**`worldline_interaction_graph(h, fraction)`:**
Builds the interaction graph by:
1. Finding all "deep" vertices (depth ≥ fraction × max_depth)
2. For each hyperedge, finding which of its vertices are deep
3. Connecting all pairs of deep vertices in that edge
This represents: "which long-lived worldlines have been in the same causal neighborhood?"

### 14.2 Knot Detection: `detect_candidate_knot_neighborhoods()`

This is the **particle detector**. Here is the complete algorithm:

```
Step 1: For each vertex v in the graph:
  - Take v and all its 1-hop neighbors as a "neighborhood" N(v)
  - If |N(v)| < 3: skip (too small to be interesting)
  - Compute coherence = internal_edges(N(v)) / boundary_edges(N(v))
    where:
      internal_edges = edges where BOTH endpoints are in N(v)
      boundary_edges = edges where ONE endpoint is in N(v), one is outside
  - Compute compactness = internal / (internal + boundary)
  - If coherence > min_coherence AND compactness > 0.6:
    → This neighborhood is a "seed" — a candidate particle

Step 2: Union-Find merge:
  - Build a map: vertex → list of seeds it belongs to
  - For each pair of seeds sharing any vertex:
    - Compute overlap = |seed_i ∩ seed_j| / min(|seed_i|, |seed_j|)
    - If overlap > 0.3: merge them (they are the same particle)
  - Union-Find with path compression for efficiency

Step 3: Return all merged groups
```

**Why Union-Find?** Merging overlapping sets naively is O(N² × set_size). Union-Find with path compression is nearly O(N). With thousands of vertices, this makes the difference between a responsive simulation and one that freezes.

### 14.3 The `TopologicalKnot` Struct

```rust
pub struct TopologicalKnot {
    pub id: u64,
    pub vertices: HashSet<u64>,
    pub age: usize,
    pub max_size: usize,
    pub min_size: usize,
    pub radius: f64,
    pub coherence: f64,
    pub velocity: f64,
    pub velocity_avg: (f64, f64),   // (dx, dy) — directional
    pub mass: f64,
    pub momentum: f64,
    pub energy: f64,
    pub prev_mass: f64,
    pub prev_momentum: f64,
    pub position_history: Vec<(usize, f64, f64)>,  // (time, centroid_x, coherence)
}
```

**The `position_history`** is particularly important — it records where the knot was at each time step, allowing computation of velocity and momentum history.

### 14.4 The `InteractionEvent` Struct

When two knots interact, an `InteractionEvent` is recorded:

```rust
pub struct InteractionEvent {
    pub start_time: usize,
    pub end_time: Option<usize>,
    pub duration: usize,
    pub knot_a: u64,
    pub knot_b: u64,
    pub overlap_size: usize,
    pub overlap_depth: f64,   // χ = max overlap fraction reached
    pub resonance: f64,       // A = 2*Ca*Cb / (Ca² + Cb²), harmonic mean
    // Full kinematic state before and after:
    pub pre_a:  (m, v, p, (vx,vy), coherence, stability, radius, size, boundary_ratio, energy, age)
    pub pre_b:  same
    pub post_a: Option<same>  // None if knot was destroyed
    pub post_b: Option<same>
}
```

This is what gets written to the CSV files. Each row is one complete interaction event with full before/after state. These are what allow the force law analysis.

---

## Chapter 15: `persistence.rs` — The Recorder (91 lines)

```rust
pub struct Persistence;

impl Persistence {
    // Generate filename: exports/hcsn_main_2026-05-04_14-30-00.csv
    pub fn generate_filename(prefix: &str) -> String { ... }
    
    // Write CSV header (21 columns)
    pub fn write_header(writer: &mut dyn Write) -> Result<()> { ... }
    
    // Convert InteractionEvent → CSV row
    pub fn format_event(event: &InteractionEvent, tid: usize) -> Option<String> { ... }
    
    // Open file in append mode with BufWriter
    pub fn open_writer(filename: &str) -> BufWriter<File> { ... }
}
```

**`format_event()` details:**
- Skips events with `duration < 1` (too short, noise)
- Computes pre/post momenta: `p = mass × velocity_x`
  - `mass = size × coherence²`
  - `velocity_x = velocity_avg.0.clamp(-10.0, 10.0)` (clamped for numerical stability)
- Runs numerical integrity check: if any value is NaN or Inf, the event is dropped
- Returns a formatted 21-column CSV string

**Why BufWriter?** Writing directly to a file byte-by-byte is slow. `BufWriter` accumulates writes in memory and flushes in large chunks. For 250,000 steps with writes every 2,000 steps, this matters.

**Why append mode?** The simulation uses multiple threads. Each thread writes to the same file. Append mode ensures writes don't overwrite each other (though a mutex in main.rs also protects concurrent access).

---

*Continue reading: HCSN_COMPLETE_REFERENCE_PART3.md*
