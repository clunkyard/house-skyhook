# Cadence language specification

Cadence organizes a bot as named behavioural states. The active state can run
one-time entry work, issue controls every tick, keep variables, and move to a
new state when a condition or timer is ready.

This model suits tactics with explicit phases: patrol until the scanner finds
something, track it while contact holds, then break away for a fixed interval.
The state and memory remain available across decisions without a separate
protocol adapter.

## First bot

```cadence
bot First Cadence
modes 2d 3d
initial search

state search
  each status-text "SEARCH"
  each status-value tick
  each drive 0.7
  each turn 0.25
  each scan-heading sweep(8)
  each scan-pitch wave(30, 22)
  each scan-arc 58
  when contact -> attack
end

state attack
  each status-text "TRACK"
  each status-value range
  each aim-heading bearing
  each aim-pitch elevation
  each scan-heading bearing
  each scan-pitch elevation
  each fire cooldown == 0 and abs(aim-error) < 3 and battery >= shot-cost-base + shot-cost-per-power + 5
  when not contact for 8 -> search
end
```

The example is [minimal.cadence](minimal.cadence). Run it from the repository:

```sh
node src/cli.js match --mode 3d --ticks 300 --seed 4201 \
  --bot 'First Cadence=node src/dsl/runner.js cadence bots/dsl/cadence/minimal.cadence' \
  --bot dsl-glyph
```

Add `--serve` to watch. Add a neighboring `bot.json` to make a new source
available in browser setup; the [DSL
guide](../../../docs/dsls.md#register-your-source) contains the manifest
shape.

This bot fires at power 1 once the turret, which turns at 220° per second, is
within 3° of the target and the battery can pay for the shot with 5 to spare.
The new-robot starter,
[src/public/starters/cadence/bot.cadence](../../../src/public/starters/cadence/bot.cadence),
also picks shot power by range and keeps a larger reserve for driving.

## State timing

Cadence starts in the state named by `initial`. A decision tick runs in this
order:

1. Run the state's `enter` lines if this is its first tick.
2. Run every `each` line in source order.
3. Check transitions in source order and choose the first ready transition.
4. If a transition is chosen, mark its target to enter on the next tick.

The action sent on a transition tick still comes from the old state. Entry
actions for the new state run on the following tick.

`state-time` starts at zero. It increments after a tick with no transition.
`after 20 -> break` therefore changes state after the current state has
reached 20 on its timer. `when contact for 3 -> track` requires three
consecutive true checks; a false check resets that transition's count.

[skyhook.cadence](skyhook.cadence) combines sensor loss holds and timed
breakaways. [ridge-hover.cadence](ridge-hover.cadence) uses states to acquire
and maintain terrain-relative altitude in gravity.

[patient-lock.cadence](patient-lock.cadence) confirms a contact, chooses an
orbit side once on entry, tolerates brief scanner gaps and waits for sustained
charge in a recovery state. The [workshop](../../../docs/dsl-workshop.md#cadence-make-commitment-and-patience-explicit)
explains the resulting motion and gives a real local match command.

## File structure

```text
bot DISPLAY NAME
modes 2d 3d
rulesets classic inertial-2d gravity-flat gravity-hills zero-g
initial STATE

state STATE
  enter ACTION EXPRESSION
  enter set NAME EXPRESSION
  each ACTION EXPRESSION
  each set NAME EXPRESSION
  when EXPRESSION -> STATE
  when EXPRESSION for NUMBER -> STATE
  after EXPRESSION -> STATE
end
```

- `bot`, `modes`, and `rulesets` have the same meaning as in Vane.
- `initial` is required and must name a declared state.
- State names and transition targets are case-sensitive.
- A duplicate state or unknown transition target is a compile error.
- `#` begins a comment outside a quoted string.

The manifest and source declarations must support the selected match. A
mismatch is reported during the protocol handshake.

## Actions and memory

`enter` and `each` accept every Arena action, plus `set`:

```cadence
enter set passes 0
each set passes passes + 1
each status-value passes
```

Memory names are case-insensitive and do not need a separate declaration. A
write is immediately visible to later lines and transition expressions in the
same tick. Values may be finite numbers, strings, or booleans.

Actions run in source order. If `enter` and `each` write the same action on an
entry tick, the later `each` value is sent. Cadence does not use Vane's
priority arbitration.

| Action | Request |
| --- | --- |
| `drive`, `turn`, `pitch` | Classic forward, yaw, and pitch controls |
| `thrust`, `strafe`, `lift` | Physics linear forces |
| `yaw-torque`, `pitch-torque`, `roll-torque` | Physics angular forces |
| `brake` | Physics braking assist, 0 to 1 |
| `aim-heading`, `aim-pitch` | absolute bearing and elevation the turret turns toward |
| `scan-heading`, `scan-pitch`, `scan-arc` | scanner direction and cone width |
| `fire` | fire this tick if `cooldown` is 0 and the battery can pay; a bare `fire` means true |
| `power` | shot power, 0.5 to 3 |
| `status-text`, `status-value` | spectator text and finite numeric value |

Each tick defaults to zero movement, no shot and shot power 1. Aim and scanner
targets keep their current values until a state writes new targets. The
scanner turns at once; the turret turns at up to 220° per second. A shot the
battery cannot pay for is refused at no cost.

`each brake` takes a strength: a number is clamped to 0 through 1, and a
condition brakes fully when it holds, so `each brake speed > 70` and `each
brake 0.3` both work.

## Expressions and sensors

Cadence uses Vane's expression language. It supports finite numbers, quoted
strings, `true`, `false`, parentheses, unary `-` and `not`, arithmetic,
comparisons, `and`, and `or`.

Functions are `min`, `max`, `abs`, `clamp`, `sin`, `cos`, `wave(period,
amplitude)`, `sweep(degreesPerTick)`, and `angle(target, current)`. Angles are
degrees. Boolean `and` and `or` short-circuit.

| Area | Values |
| --- | --- |
| Match | `tick`, `time`, `remaining`, `mode3`, `ruleset`, `state-time` |
| Own robot | `health`, `cooldown`, `x`, `y`, `z`, `heading`, `pitch`, `aim`, `aim-pitch` |
| Battery | `battery`, `battery-capacity`, `battery-fraction` |
| Scanner | `scan-heading`, `scan-pitch`, `scan-arc`, `scan-range`, `contact`, `range`, `bearing`, `elevation`, `aim-error` |
| Shots | `power-min`, `power-max`, `shot-speed-base`, `shot-speed-per-power`, `shot-cost-base`, `shot-cost-per-power` |
| Safe zone | `zone-active`, `zone-radius`, `zone-distance`, `zone-bearing`, `zone-elevation` |
| Match mechanics | `storm-active`, `storm-distance`, `storm-bearing`, `storm-heading`, `storm-radius`, `cue-present`, `cue-bearing`, `cue-elevation`, `cue-range`, `cue-error`, `cue-age`, `cue-muzzle`, `pickup-present`, `pickup-bearing`, `pickup-elevation`, `pickup-range`, `hill-active`, `on-hill`, `hill-held`, `hill-bearing`, `hill-elevation`, `hill-range`, `hill-radius`, `hill-time`, `hill-to-win`, `score`, `score-best` |
| Arena | `width`, `height`, `depth`, `terrain-kind` |
| Physics flags | `physics-mode`, `inertial`, `gravity`, `zero-g`, `hills`, `grounded` |
| Motion | `vx`, `vy`, `vz`, `speed`, `yaw-rate`, `pitch-rate`, `roll-rate` |
| Ground and gravity | `altitude`, `terrain-height`, `gravity-x`, `gravity-y`, `gravity-z` |
| Attitude basis | `forward-x/y/z`, `right-x/y/z`, `up-x/y/z` |

`contact`, `range`, `bearing`, and `elevation` describe only the nearest
current scanner echo. No opponent state is added to memory automatically.
`aim-error` is the signed heading difference from where the turret points now
to that echo, and 0 without one. In 3D, also compare `elevation` with
`aim-pitch`.

`cooldown` is 0 when the gun can fire this tick. `aim` and `aim-pitch` report
where the turret points now. `scan-range` is the reach of the current scan
arc. A shot costs `shot-cost-base + shot-cost-per-power * power` battery and flies at
`shot-speed-base + shot-speed-per-power * power`. Before sudden death,
`zone-active` is false, the zone distance and radius are 0, and the zone
bearing is the robot heading. In classic, `speed` is the smoothed speed that
aim spread uses. The [DSL
guide](../../../docs/dsls.md#shared-observations-and-actions) lists every
value, and the [protocol guide](../../../docs/protocol.md) defines each one.

The match mechanics' values read as inactive while their mechanic is off or
not yet in play: flags false, ranges and distances a very large number,
bearings the robot heading. `storm-distance` is to the storm's edge and
negative inside it. `cue-*` describe the best passive cue, remembered for 60
ticks where it placed a robot and seen from where the robot is now. The [DSL
guide](../../../docs/dsls.md#shared-observations-and-actions) defines each
one.

## Transitions

```text
when CONDITION -> target
when CONDITION for TICKS -> target
after DURATION -> target
```

`for` takes a numeric literal, floors it to an integer, and requires at least
one tick. `after` evaluates its expression every tick and requires a finite,
non-negative result. When several transitions are ready, the first one in the
state wins. Put emergency transitions before routine timeouts.

Changing state resets `state-time` and all transition hold counters. Memory
survives the transition.

## Work limits and errors

Without a supplied budget, the local Cadence interpreter allows 640 operations
per decision. Hosted DSL decisions and browser practice use 1024. The local
CLI with `--resources dsl` defaults to 768; add `--dsl-ops 1024` to use the
hosted operation limit. A supplied `dslOpsPerTurn` replaces the interpreter
default. Entry work, per-tick actions, expressions and transition checks all
consume operations. The reported `dslOps` value is host-metered.

Errors include `file:line:column`. Missing `end`, invalid states, unknown
actions or values, division by zero, non-finite results, stack-independent
work exhaustion, and invalid transition durations forfeit the bot cleanly.

## Compile to WASM

The build needs Clang, a C compiler and the Wasmtime C API; see the
[WebAssembly guide](../../../docs/sdk/wasm.md). Run from the repository root:

```sh
scripts/build-dsl-wasm.sh cadence
cat > build/cadence-wasm.json <<'JSON'
{
  "name": "Wasm Cadence",
  "language": "cadence-wasm",
  "protocol": 2,
  "supportedModes": ["2d", "3d"],
  "resourceModes": ["wasm"],
  "command": ["build/clunkyard-wasm-host", "build/wasm-skyhook-cadence.wasm"]
}
JSON
node src/cli.js match --resources wasm \
  --bot @build/cadence-wasm.json --bot c --ticks 300
```

The generated module retains the current state, state timer, memory values and
transition hold counters in its linear memory. Wasmtime meters it like the C
and Rust modules. The interpreter remains available as `dsl-cadence`. The
module speaks WebAssembly ABI 4: it reads every value through the robot core's
`dsl_fill_v1` import once a turn and sets each control through the core's
setters. Numeric `status-value` works in the module. Text
`status-text` produces a located compile error; use the interpreter when a
program needs text.

## Grammar

```text
program      := directive* "initial" state-name state+
directive    := "bot" words
              | "modes" ("2d" | "3d")+
              | "rulesets" ruleset+
state        := "state" state-name newline statement* "end"
statement    := ("enter" | "each") (action | set) newline
              | "when" expression ("for" number)? "->" state-name newline
              | "after" expression "->" state-name newline
set          := "set" word expression
action       := action-name expression | "fire"
ruleset      := "classic" | "inertial-2d" | "gravity-flat"
              | "gravity-hills" | "zero-g"
```
