# HCSN Complete Reference — Part 3
## The Rewrite Engine Deep Dive + Interview Guide

---

# PART 3: THE ENGINE — `rewrite_engine.rs`

This is the heart of HCSN. At 1912 lines, it is the most important file.

## Chapter 16: Key Enums — Modes of Operation

```rust
pub enum EmergenceMode {
    Control,   // No stability bonus, hard threshold. Used for falsification tests.
    Assisted,  // Honest inheritance + sigmoid threshold. DEFAULT production mode.
    Forced,    // Constant bonus + hard threshold. Experimental only.
}
```

**What these mean:**
- **Control**: The "unpatched vacuum" — no memory, no survival bonus. Tests whether particles emerge from pure topology. Used in the Aggressive Mode experiments.
- **Assisted**: The real engine. Stability memory accumulates in coherent structures, giving them a survival advantage. This is what produces the power-law lifetime distribution.
- **Forced**: Gives everything a constant boost. Too artificial. Not used in production.

```rust
pub enum ConservationMode {
    Baseline,          // No conservation enforcement
    Pairwise,          // Symmetric momentum correction on each interaction
    StabilityScaled,   // Inertial cooling proportional to stability
    FluxCompensated,   // Momentum reservoir that leaks and re-absorbs
    MassCoupled,       // Soft Newtonian correction (30% of error)
    TimeSymmetry,      // Stochastic undo of high-error steps
    Hybrid,            // Pairwise + FluxCompensated. PRODUCTION DEFAULT.
}
```

These represent the "Hypotheses A–E" for emergent conservation. Each was tested separately. `Hybrid` is the combination that works best.

---

## Chapter 17: `RewriteEngine` State — What the Engine Remembers

```rust
pub struct RewriteEngine {
    // The Universe
    pub h: Hypergraph,              // The live causal network
    pub p_create: f64,              // Base probability of edge creation vs fusion
    pub mode: EmergenceMode,
    
    // Physics configuration
    pub params: PhysicsParams,
    pub gamma_time: f64,            // 0.1 — temporal decay coupling
    pub gamma_ext: f64,             // 0.05 — external coupling
    pub epsilon_label_violation: f64, // 0.08 — charge mismatch tolerance
    
    // ξ-field
    pub xi: HashMap<u64, f64>,      // Current ξ value per vertex
    pub prev_xi: HashMap<u64, f64>, // ξ from previous step (for delta tracking)
    pub xi_threshold: f64,          // 1e-6 — below this, ξ is ignored
    pub xi_decay: f64,              // 0.70 — ξ retention fraction per step
    pub xi_coupling: f64,           // 0.6 — propagation coupling
    
    // Geometry memory (two types)
    pub topo_distance_memory: HashMap<(String, usize, usize), f64>,
    pub xi_distance_memory: HashMap<(String, usize, usize), f64>,
    pub distance_memory_decay: f64,  // 0.9 — exponential memory fade
    pub geometry_stride: usize,      // 5 — update geometry every 5 steps
    
    // Particle tracking
    pub active_knots: HashMap<u64, TopologicalKnot>, // Currently alive particles
    pub dead_knots: Vec<TopologicalKnot>,            // Dissolved particles (last 200)
    pub interaction_events: Vec<InteractionEvent>,   // Completed interactions
    
    // Stability memory (the key to H8)
    pub stability: HashMap<u64, f64>,  // Per-vertex accumulated stability
    
    // Conservation
    pub conservation_mode: ConservationMode,
    pub momentum_reservoir: HashMap<u64, f64>, // For FluxCompensated mode
    
    // Bookkeeping
    pub time: usize,
    pub rewrite_history: Vec<UndoRecord>, // Last 200 rewrites (for undo)
    pub cached_inter: Option<HashMap<u64, FixedBitSet>>, // Cached interaction graph
    pub interaction_counts: HashMap<(u64, u64), u16>,   // How often pair interacted
}
```

**Key design insight:** The engine holds ALL state. The `Hypergraph` is just the raw data; the `RewriteEngine` wraps it with all the physics memory and tracking logic.

---

## Chapter 18: `step()` — One Tick of the Universe

This function is called once per simulation step. Let's walk through it completely.

### Phase 1: Bookkeeping

```rust
self.time += 1;

if self.time % 100 == 0 {
    self.h.scrub_ghost_bits();  // Clean up stale causal references
}
```

Every 100 steps: remove dead vertices from causal bitsets.

### Phase 2: Bootstrap (First Step Only)

```rust
if self.cached_inter.is_none() {
    let inter = worldline_interaction_graph(&self.h, 0.1);
    self.cached_inter = Some(inter);
    // Also populate interaction_counts from initial state
}
```

On step 1, build the interaction graph from scratch. After that, it is updated **incrementally** (delta updates) instead of recomputed from scratch — this is a major speed optimization.

### Phase 3: Time Symmetry Undo (Optional)

```rust
if conservation_mode == TimeSymmetry {
    // Check if any knot's momentum changed by >50%
    let high_error = active_knots.values()
        .any(|k| (k.momentum - k.prev_momentum).abs() / k.prev_momentum.abs() > 0.5);
    
    let p_undo = if high_error { 0.2 } else { 0.05 };
    
    if rng.gen::<f64>() < p_undo {
        // Pop the last rewrite and reverse it
        if let Some(record) = self.rewrite_history.pop() {
            self.undo_changes(record);
            // Still update physics even on undo
            return true;
        }
    }
}
```

This is Hypothesis E. If a rewrite caused a large momentum jump, there is a 20% chance of reversing it. This implements a statistical time-reversal symmetry.

### Phase 4: Vacuum Nucleation (Optional)

```rust
if !self.pure_mode && self.params.defect_injection > 0.0 {
    if rng.gen::<f64>() < self.params.defect_injection {
        // Create 4 vertices, fully connect them causally
        // This injects a ready-made coherent cluster (seed knot)
    }
}
```

When `defect_injection > 0`, the vacuum spontaneously produces new particle seeds. Used for studying interaction of separately-born particles.

### Phase 5: Propose a Rewrite

```rust
let undo_opt = self.propose_rewrite(&inter);
```

This is where `propose_rewrite()` is called. See Chapter 19 below.

### Phase 6: Apply or Reject

```rust
let accepted = rng.gen::<f64>() <= accept_prob;  // accept_prob = 1.0 always

if accepted {
    // Update interaction graph incrementally
    self.update_interaction_graph_delta(&undo, true);
    
    // Propagate ξ-field
    self.propagate_xi(&inter, &xi_clusters);
    
    // Update geometry memory every 5 steps
    if self.time % self.geometry_stride == 0 {
        self.update_topo_distance_memory(&inter, &xi_support);
    }
    
    // Update particles and physics
    self.update_topological_knots(&inter);
    self.update_stability(&inter);
    self.perform_kinematics_and_interactions(&inter);
} else {
    self.undo_changes(undo);
}
```

Currently `accept_prob = 1.0` (always accept). The rejection infrastructure exists for future Metropolis-Hastings style acceptance.

---

## Chapter 19: `propose_rewrite()` — The Suppression Logic

This is where HCSN's most important physics lives: the suppression mechanism that protects coherent structures from being randomly destroyed.

```
Algorithm:
1. If the hypergraph is empty: return None (nothing to rewrite)

2. Sample a random vertex as "anchor"

3. Compute local_density:
   local_density = local_clustering × (local_degree / avg_degree)
   
   where:
   - local_clustering = fraction of anchor's neighbors that are connected to each other
   - local_degree = number of hyperedges containing anchor
   - avg_degree = average hyperedge degree across all vertices

4. Compute local_coherence for the anchor's neighborhood:
   (internal_edges, boundary_edges) = compute_coherence_raw(neighborhood)
   coherence = internal / boundary  (or 10.0 if pure clique)

5. Compute alpha_eff — the suppression strength:
   alpha_eff = 2.0  (base — always suppressing somewhat)
   
   + coherence_boost:
     if neighborhood size ≥ 4 AND coherence > 1.0:
       alpha_eff += 0.5 × (coherence - 1.0)
       (More coherent = more protected)
   
   + memory_contribution:
     if anchor is in active knot:
       stability_val = mean stability of knot vertices (capped at 50)
       alpha_eff += mu × 50 × (stability_val/50)^gamma
       (= 0.3 × 50 × (s/50)^2.2)
       (More historical stability = more suppressed)

6. coupling_modifier:
   If anchor is in a deep-overlap (chi > 0.4) interaction:
     coupling_modifier = 0.2  (reduce protection during scattering!)
   else:
     coupling_modifier = 1.0

7. rewrite_prob = exp(-alpha_eff × coupling_modifier × local_density)

8. If rng.gen() > rewrite_prob:
   → This rewrite is SUPPRESSED (self.suppressed_rewrites += 1)
   → Return None

9. Decide: grow or contract?
   coherence_ratio = coherence / nucleation_threshold
   where nucleation_threshold ≈ 1.3 (the critical coherence for particle formation)
   
   If coherence_ratio > 1 (structure is coherent): bias toward growth
   If coherence_ratio < 1 (structure is disordered): equal chance

10. Call edge_creation_rule() or vertex_fusion_rule()
```

**Why `coupling_modifier = 0.2` during deep overlap?**

This is subtle and important. During a collision (χ > 0.4), we REDUCE the suppression. This allows the scattering to actually happen — the structures can be partially modified. Without this, two particles would simply pass through each other unchanged because the suppression would protect both of them completely.

This one line is responsible for the non-trivial scattering angles observed in Phase 12.

---

## Chapter 20: `update_stability()` — The Memory System

```rust
fn update_stability(&mut self, inter: &HashMap<u64, FixedBitSet>) {
    // Step 1: Decay ALL stability values
    for val in self.stability.values_mut() {
        *val *= self.params.stability_decay;  // × 0.975 every step
    }
    
    // Step 2: Accumulate for vertices in ACTIVE knots
    for knot in self.active_knots.values() {
        for &v_id in &knot.vertices {
            let s = self.stability.entry(v_id).or_insert(0.0);
            *s += 1.0;
            *s = s.min(50.0);  // Hard cap at 50
        }
    }
    
    // Step 3: Prune dead vertex entries
    let alive: HashSet<u64> = self.h.vertices.keys().cloned().collect();
    self.stability.retain(|k, _| alive.contains(k));
}
```

**How stability memory works:**

Each step:
1. Everyone loses 2.5% of their stability
2. Vertices currently in active knots gain +1.0
3. Maximum is 50.0

After k steps in a knot, a vertex has approximately:
```
stability ≈ sum_{i=0}^{k} 0.975^i ≈ 40 (at equilibrium)
```

A vertex not in a knot for 100 steps has: 50 × 0.975^100 ≈ 7.8

This means: **vertices that have been in stable structures for a long time are heavily protected**. The protection is history-dependent — the longer you've survived, the harder you are to destroy. This is what breaks the memoryless exponential decay and produces the observed power-law lifetime distribution.

---

## Chapter 21: `perform_kinematics_and_interactions()` — The Physics

This function computes all kinematic quantities for active knots and detects/records interactions.

### Computing Mass, Velocity, Momentum

```rust
for knot in active_knots.values_mut() {
    // Mass = size × coherence²
    knot.mass = knot.vertices.len() as f64 
                × knot.coherence.powi(2);
    
    // Velocity from position history
    // centroid_x = average vertex ID (proxy for position in interaction graph)
    if position_history.len() >= 2 {
        let (t1, x1, _) = history[history.len() - 2];
        let (t2, x2, _) = history[history.len() - 1];
        let dt = (t2 - t1) as f64;
        knot.velocity = (x2 - x1).abs() / dt;
        knot.velocity_avg = ((x2-x1)/dt, 0.0);  // Only x-component tracked
    }
    
    // Momentum and energy
    knot.prev_momentum = knot.momentum;
    knot.momentum = knot.mass * knot.velocity;
    knot.energy = 0.5 * knot.mass * knot.velocity.powi(2) 
                  + 0.02 * mean_stability;
}
```

### Interaction Detection

```rust
// For each pair of active knots (A, B):
let intersection: HashSet<u64> = knot_a.vertices
    .intersection(&knot_b.vertices).cloned().collect();

// χ = overlap fraction
let chi = intersection.len() as f64 
          / knot_a.vertices.len().min(knot_b.vertices.len()) as f64;

if chi > 0.015 {
    // Start or update an InteractionEvent
    if !active_interactions.contains_key(&(a_id, b_id)) {
        active_interactions.insert((a_id, b_id), InteractionEvent {
            start_time: self.time,
            overlap_depth: chi,  // Will track maximum
            // ... pre-state captured here
        });
    } else {
        // Update max overlap
        event.overlap_depth = event.overlap_depth.max(chi);
    }
}

if chi > 0.4 {
    // Mark vertices as "coupled" → reduce suppression
    coupled_vertices.extend(intersection);
}

if chi == 0.0 && interaction was active {
    // Interaction has ended: capture post-state, push to interaction_events
    event.end_time = Some(self.time);
    event.duration = self.time - event.start_time;
    // Capture post_a, post_b states
    self.interaction_events.push(finalized_event);
}
```

### The Force Law (Pairwise Conservation Mode)

```rust
// In Pairwise mode, during interactions:
let delta_p = (post_momentum_a + post_momentum_b) 
              - (pre_momentum_a + pre_momentum_b);

// Stability-ramped correction:
let stability_factor = mean_stability.min(20.0) / 20.0;
let k = stability_factor;  // Ramps from 0 to 1 as stability reaches 20

let correction = -k * 0.5 * delta_p;

// Apply correction to both knots
knot_a.momentum += correction;
knot_b.momentum += correction;
```

This is where the empirical coupling constant k = 182.1 comes from — it's the fitted strength of this correction across all interactions in Phase 12.

---

## Chapter 22: `main.rs` — How Everything Runs Together

```rust
fn main() {
    // Read configuration from environment
    let total_steps = env::var("HCSN_STEPS").unwrap_or("250000").parse().unwrap();
    let p_create = env::var("HCSN_P_CREATE").unwrap_or("0.58").parse().unwrap();
    let num_threads: usize = 2;  // Dual-core
    
    let steps_per_thread = total_steps / num_threads;
    let out_file = Persistence::generate_filename("main");
    
    // Create shared CSV writer (Arc<Mutex<>> for thread safety)
    let shared_writer = Arc::new(Mutex::new(header_writer));
    
    // Run both threads in parallel using Rayon
    (0..num_threads).into_par_iter().for_each(|tid| {
        // Each thread creates its OWN universe
        let mut h = Hypergraph::new();
        let v1 = h.add_vertex();
        let v2 = h.add_vertex();
        h.add_causal_relation(v1.id, v2.id);
        h.add_hyperedge(vec![v1.id, v2.id]);
        
        // Each thread creates its OWN engine
        let mut engine = RewriteEngine::new(h, p_create, None);
        engine.mode = EmergenceMode::Assisted;
        engine.conservation_mode = ConservationMode::Hybrid;
        engine.thread_id = Some(tid);
        engine.max_steps = steps_per_thread;
        engine.verbose = false;  // No console spam in parallel mode
        
        // Main loop
        for s in 1..=steps_per_thread {
            engine.step();
            
            // Every 2000 steps: flush interaction events to CSV
            if s % 2000 == 0 || s == steps_per_thread {
                let mut writer_lock = shared_writer.lock().unwrap();
                let events = std::mem::take(&mut engine.interaction_events);
                for event in events {
                    if let Some(row) = Persistence::format_event(&event, tid) {
                        writeln!(writer_lock, "{}", row).unwrap();
                    }
                }
            }
        }
    });
}
```

**Key design decisions:**
- **Separate universe per thread**: Each thread has its own `Hypergraph` and `RewriteEngine`. There is no shared simulation state — only shared output.
- **`Arc<Mutex<>>`**: The CSV writer is shared between threads using an atomic reference count (Arc) and a mutex lock. Each thread acquires the lock only when writing.
- **`std::mem::take()`**: Takes all interaction events out of the engine's vector in a single operation (O(1)) and gives ownership to the writing thread, leaving the engine's vector empty and ready for the next batch.

---

# PART 4: INTERVIEW GUIDE

## Chapter 23: Questions You Will Be Asked

### "What is HCSN in one sentence?"

> "HCSN is a computational framework that tests whether particles and spacetime can emerge spontaneously from stochastic rewriting of a causal hypergraph, without assuming they exist in advance."

### "Why is this original research and not just Wolfram physics?"

> "Wolfram Physics uses hypergraph rewriting similarly, but HCSN adds three things: (1) rigorous operational definitions — every concept like 'mass' and 'momentum' is defined as a measurement, not assumed; (2) explicit falsifiability criteria — the theory lists what results would disprove it; (3) empirical discovery — the threshold-gated force law (χ > 0.14) and the coupling constant k = 182.1 were discovered from simulation data, not imposed."

### "What have you actually proven?"

> "Proven is too strong a word — this is simulation science. What we have validated: (1) persistent topological structures emerge without injection, (2) they have mass-like and momentum-like properties that are measurable, (3) they interact via a threshold-gated law, (4) momentum conservation emerges statistically with Spearman ρ ≈ −0.47. These are reproducible across multiple random seeds and parameter variations."

### "Why do you need a workstation?"

> "The fundamental bottleneck is simulation scale. To study the asymptotic behavior of the theory — whether the effective dimension stabilizes, whether exact conservation laws emerge at large N — we need runs of order 10⁶ to 10⁷ steps. A Phase 12 run of 250,000 steps on a dual-core laptop takes ~15 minutes. A 2-million step run would take 2 hours on laptop hardware. On a 32-core workstation with 64GB RAM, we could run 16 parallel seeds simultaneously in the same 2 hours — a 16× experiment throughput improvement. This would let us complete the dimensional characterization, the many-body competition study, and the criticality sweep within months instead of years."

### "What do you mean by 'fragile conservation'?"

> "In standard physics, conservation laws are exact — momentum is always conserved. In HCSN, we found that momentum conservation is statistical and approximate. Across 9 independent simulations (unpatched), the Spearman correlation between worldline persistence and momentum drift is ρ = −0.47 ± 0.16. This means: longer-lived structures have less momentum drift on average, but it's not exact. The implication is that conservation laws in HCSN are not axioms — they are *emergent statistical phenomena* that become more reliable as structures become more stable. This is actually a stronger claim than exact conservation: it suggests conservation laws arise from dynamics, not from symmetry assumptions."

### "What is a Topological Knot?"

> "A topological knot in HCSN is a persistent, localized subgraph with high structural coherence — meaning more internal connections than boundary connections. It is tracked through time using overlap continuity: if two snapshots share more than 30% of their vertices, they are the same knot. The knot's identity does not depend on which specific vertices it contains — the vertices can completely turn over, and the knot persists as a pattern. This is analogous to how a whirlpool persists in a river even though the specific water molecules change completely."

### "Why Rust over Python?"

> "The simulation needs to run 250,000 steps, each involving O(N²) causal lookups, bitset union operations, and hash table access patterns. Rust compiles to native machine code with zero runtime overhead, no garbage collector pauses, and enables true SIMD-accelerated bitset operations. The migration gave roughly 50-100× speedup. More importantly, Rust's ownership system catches entire categories of bugs at compile time — in a complex simulation with mutable state shared across functions, this prevented hundreds of subtle bugs that would have been silent errors in Python."

### "What does `cargo run --release` do?"

> "Without `--release`, Rust compiles with debug symbols and no optimizations — code runs slowly but is easy to debug. `--release` enables full LLVM optimization: loop unrolling, inlining, SIMD vectorization, dead code elimination. For the bitset operations that dominate HCSN's compute time, this gives 3-10× additional speedup over the debug build. The `--release` flag is essential for any run beyond 10,000 steps."

### "What is the ξ field?"

> "ξ (xi) is an internal accounting variable — not a physical field. It is a scalar assigned to each vertex that propagates through the network according to a diffusion-like rule: 70% self-retention, 15% spread to neighbors per step, with exponential decay. It serves as a marker for which regions of the network are 'active' — where structural dynamics are concentrated. Knot detection uses ξ support to identify candidate particle regions. Critically, ξ does NOT feed back into the rewrite dynamics — it has no causal influence on the simulation. It is a pure measurement instrument."

---

## Chapter 24: The Speed Comparison — Python vs Rust

| Metric | Python (hcsn-sim) | Rust (hcsn-rust) |
|---|---|---|
| Language | Interpreted | Compiled (LLVM) |
| GC | Yes (reference counting) | None |
| Causal lookup | O(N) dict search | O(1) bitset read |
| Causal intersection | O(N²) set operations | Bitwise AND |
| Memory per step | High (Python objects) | Minimal (raw integers) |
| Parallelism | GIL prevents true parallel | Rayon data parallelism |
| 50k steps | ~30 minutes | ~2 minutes |
| 250k steps | ~150 minutes | ~12 minutes |

The key algorithmic improvement was the `FixedBitSet`-based transitive closure. In Python, checking "is vertex 5000 reachable from vertex 3?" required traversing a chain of dictionary lookups. In Rust with bitsets, it is a single bit read: `causal_future[3].contains(5000)`.

---

## Chapter 25: The Complete Flow — What Happens When You Run

```
cargo run --release
↓
main.rs:main()
  → Creates 2 threads via Rayon
  → Each thread:
    → Creates minimal Hypergraph (2 vertices, 1 edge, 1 causal link)
    → Creates RewriteEngine with Assisted + Hybrid modes
    → Loops 125,000 times:
      → engine.step()
        → time++
        → Every 100: scrub_ghost_bits()
        → Step 1: bootstrap interaction graph cache
        → propose_rewrite():
          → Pick random vertex
          → Compute local_density (clustering × degree_ratio)
          → Compute coherence of neighborhood
          → alpha_eff = 2.0 + coherence_boost + memory_term
          → If suppressed: return None
          → Else: call edge_creation_rule() or vertex_fusion_rule()
        → If undo returned: apply delta to interaction graph
        → propagate_xi(): diffuse ξ through network
        → Every 5 steps: update geometry memory
        → update_topological_knots(): 
          → Run detect_candidate_knot_neighborhoods()
          → Match new candidates to existing knots (overlap continuity)
          → Kill knots that lost overlap continuity
          → Record knot births/deaths
        → update_stability(): decay all, accumulate for active knots
        → perform_kinematics_and_interactions():
          → Compute mass, velocity, momentum, energy for each knot
          → Detect knot overlaps (χ)
          → Start/update/finalize InteractionEvents
          → Apply conservation correction (Hybrid mode)
      → Every 2000 steps: acquire mutex, flush InteractionEvents to CSV
  → Final flush on completion
→ CSV file written to exports/
→ Analysis with scripts/
```

---

## Chapter 26: The Numbers That Matter

Memorize these for your interview:

| Number | Context |
|---|---|
| **0.14** | Interaction threshold χ_c — below this, zero force |
| **182.1** | Empirical coupling constant k |
| **71.5°** | Mean back-scattering deflection angle |
| **1.7–2.0** | Power-law exponent α for lifetime distribution |
| **58%** | Hazard rate decrease over knot lifetime |
| **−0.47** | Spearman ρ for fragile momentum conservation |
| **0.64** | Critical p_create for power-law criticality |
| **2.2** | Critical γ (nonlinear_coupling) |
| **1.1** | Critical Ω value for phase transition |
| **0.975** | Stability decay rate ν per step |
| **0.70** | ξ self-retention fraction per step |
| **50** | Maximum stability cap |
| **524,288** | Initial bitset capacity (2^19) |
| **2,000** | Steps between CSV flushes |
| **100** | Steps between ghost bit scrubs |
| **250,000** | Default total steps per run |
| **Phase 12** | Current phase — Full Interaction Theory |

---

*End of HCSN Complete Reference (3 parts)*

**Files:**
- `HCSN_COMPLETE_REFERENCE_PART1.md` — Theory, Axioms, Concepts
- `HCSN_COMPLETE_REFERENCE_PART2.md` — Code Architecture, File Walkthroughs
- `HCSN_COMPLETE_REFERENCE_PART3.md` — Engine Deep Dive, Interview Guide
