# Bowling Simulation Engine — Written Explanation

## 1. Skill Model Design

The engine maps a single `skill` parameter in [0, 1] to bowling performance through three interconnected components: strike probability, spare probability, and a non-strike/spare pin-count distribution. The goal was a model that is simple, monotonic in skill, and produces realistic score distributions across the skill spectrum.

### Strike Probability

$$P_{\text{strike}} = 0.01 + 0.79 \cdot \text{skill}^2$$

The quadratic scaling reflects the fact that bowling improvement is nonlinear — the difference between a skill=0.3 and skill=0.5 bowler is much smaller than the gap between skill=0.8 and skill=1.0. Professionals achieve roughly 60–80% strike rates, while beginners rarely strike at all. Selected values:

| Skill | Strike % |
|-------|----------|
| 0.00  | 1%       |
| 0.25  | 6%       |
| 0.50  | 21%      |
| 0.75  | 45%      |
| 1.00  | 80%      |

### Spare Probability

$$P_{\text{spare}} = 0.05 + 0.70 \cdot \text{skill}^{1.5}$$

Spare conversion uses a gentler exponent (1.5 vs. 2) because spares are mechanically easier than strikes — you only need to hit the remaining pins, not achieve the precise pocket angle needed for a strike. The spare probability is applied uniformly regardless of how many pins remain. This is a deliberate simplification; in reality, certain leave patterns (e.g., a 7-10 split) are much harder than others. However, modeling pin-specific spare difficulty would add significant complexity without proportional benefit for a first-order simulation.

| Skill | Spare % |
|-------|---------|
| 0.00  | 5%      |
| 0.25  | 14%     |
| 0.50  | 30%     |
| 0.75  | 51%     |
| 1.00  | 75%     |

### Pin-Count Distribution (Non-Strike, Non-Spare Rolls)

When neither a strike nor a spare occurs, the number of pins knocked down is drawn from a power distribution:

$$\text{pins} = \lfloor r^{1/(1 + 3 \cdot \text{skill})} \cdot (\text{maxPins} + 1) \rfloor$$

where `r ~ Uniform(0, 1)` from the seeded PRNG and `maxPins` is the number of standing pins minus one (since a strike/spare has already been ruled out).

The exponent $1/(1 + 3 \cdot \text{skill})$ controls the distribution shape. At skill=0, the exponent is 1 and the distribution is uniform across [0, maxPins]. At skill=1, the exponent is 0.25, which compresses the distribution toward the high end — a skilled bowler who misses a strike still knocks down 7–9 pins most of the time. This creates the realistic pattern where high-skill bowlers rarely produce low pin counts.

### Score Distribution Validation

Running 5,000 simulated games at each skill level produces average scores that align with real-world expectations:

| Skill | Avg Score | Real-World Analog          |
|-------|-----------|----------------------------|
| 0.00  | ~73       | First-time bowler          |
| 0.25  | ~98       | Casual recreational bowler |
| 0.50  | ~131      | League-average bowler      |
| 0.75  | ~186      | Competitive amateur        |
| 1.00  | ~254      | Professional / touring pro |

The distribution is monotonically increasing in skill, and the variance is highest in the middle range (where randomness plays the largest role), which mirrors real bowling.


## 2. Randomness and Seed Implementation

### PRNG Choice: Mulberry32

The engine uses the Mulberry32 algorithm, a 32-bit multiply-with-carry PRNG. It was selected for three reasons: it produces high-quality uniform output (passes SmallCrush), it is fast and simple to implement in pure JavaScript, and it is fully deterministic given a seed.

Each call to the PRNG returns a float in [0, 1) by dividing the unsigned 32-bit state by 2^32.

### How the Seed Determines Behavior

The seed initializes the PRNG state once at the start of a simulation (per frame in `mode=frame`, per game in `mode=game`). All subsequent randomness — strike checks, spare checks, pin-count draws — comes from sequential calls to the same PRNG instance. This means:

- **Same seed → same RNG stream → identical rolls → identical output.** This satisfies the determinism requirement.
- **Different seed → different RNG stream → different rolls.** Even a seed change of 1 produces entirely unrelated output because Mulberry32 has no correlation between nearby seeds.
- **The RNG is never re-seeded within a game.** A common mistake would be to re-seed per frame, which would make all frames identical for a given skill level. Instead, the PRNG is seeded once and rolls are drawn sequentially across all 10 frames. Frame 5's outcome depends on the cumulative number of RNG calls consumed by frames 1–4, which depends on whether those frames had strikes (fewer calls per frame) or not.


## 3. Design Decisions

### Single-File Architecture

The engine is a single `index.html` file with all logic in embedded JavaScript. This makes deployment trivial (any static hosting platform works) and eliminates external dependencies. The same file serves JSON responses (for frame and game modes) and renders HTML (for viz mode), routed by the `mode` query parameter.

### Separation of Simulation and Scoring

Roll simulation and score computation are cleanly separated. The `simulateFrame` function only generates roll sequences; it has no knowledge of scoring. The `simulateGame` function first collects all rolls, then applies official scoring rules via a roll-index lookahead. This separation prevents bugs where scoring logic might inadvertently influence roll generation.

### 10th Frame Handling

The 10th frame follows official rules: a strike earns two bonus balls on fresh pins, a spare earns one bonus ball on fresh pins, and an open frame ends after two balls. After a strike, if the second ball is also a strike, the third ball is thrown on fresh pins; otherwise, the third ball targets the remaining pins from the second ball. This logic is encapsulated in the `isLastFrame` flag, which is an explicit parameter in frame mode and automatically set for frame 10 in game mode.

### Roll Simulation Structure

Each roll consumes either 1 or 2 PRNG calls: one for the strike/spare check, and (if the check fails) one for the pin-count distribution. The number of PRNG calls per frame varies depending on outcomes, which is standard for stochastic simulations. The key invariant is that the same seed always produces the same call sequence.


## 4. Assumptions and Simplifications

1. **Pin-pattern independence.** Spare probability does not depend on which specific pins remain standing. A 7-10 split and a single-pin leave have the same spare conversion rate for a given skill. This is the model's largest simplification relative to real bowling.

2. **No gutter tendency.** The pin distribution treats all non-strike/spare outcomes equally without modeling common failure modes like gutter balls or Brooklyn hits.

3. **Stateless skill.** A bowler's skill does not change during a game — there is no fatigue, momentum, or pressure modeling. Each roll is drawn from the same skill-parameterized distribution.

4. **No lane conditions.** Real bowling performance depends heavily on oil patterns, lane surface, and ball selection. The model abstracts all of this into the single `skill` parameter.

These simplifications are deliberate. The model prioritizes clarity, correctness, and determinism over physical realism. For a simulation engine that will be used for tournament comparison and statistical analysis, a coherent first-order model is more valuable than a complex model with poorly calibrated parameters.


## 5. Bonus Extension: Landing Page and Error Handling

As a small extension, the engine includes a landing page (shown when no query parameters are present) with usage documentation and clickable example links. It also performs input validation on all parameters and returns structured JSON error messages for invalid inputs (e.g., `skill` outside [0,1], missing `isLastFrame` in frame mode). This makes the API more robust for automated testing and tournament use.
