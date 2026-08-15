# Male Pregnancy QoL (MPreg Test) — Developer/AI Context

*This is the technical development log — root-cause debugging history,
hard-won engine lessons, and the current open-items/TODO list. It's kept
for future dev sessions (human or AI) so nobody has to relearn the same
lessons the hard way. For the player-facing overview (what the mod does,
how to install it, known issues), see [README.md](README.md).*

**Status as of this version:** the male character correctly shows a growing
pregnant belly in three stages, correctly deforms, has correct
material/skeleton, and is wired into the real vanilla gene system. This took
a long debugging journey (documented below) and every lesson learned is
recorded here so future work — by you, by Claude, by anyone — doesn't have
to relearn it the hard way.

Also as of this version: same-sex marriage is on by default (any faith,
including shunned/illegal ones — see Lesson 12), a real conception in a
mixed-sex couple can redirect onto a male spouse who has opted in via the
carrier decisions, and male-male conception/pregnancy/birth is implemented
end to end (Lesson 14) and **confirmed working in a live game** — a
same-sex couple has actually produced a child. Getting the underlying
crash bug (a stored character-reference variable that could go stale over
long chains) genuinely fixed took **three** attempts across Lessons
16/18/20 — each one confidently declared fixed and the first two were
wrong; recorded honestly rather than cleaned up in hindsight, because the
progression of wrong theories is itself the useful part for next time.

**v0.23 was beta-prep** (Lesson 17): debug decisions hidden (kept
functional for future dev use, not deleted), stage tracking moved off
separate traits entirely (CK3 has no real "hidden trait" field — confirmed
from `common/traits/_traits.info` — so `mp_pregnant_stage_1/2/3` were
removed in favor of the `mpreg_months` variable, which already carried the
same information), real vanilla trait icons (`pregnant.dds`/`fecund.dds`).

**v0.24–0.28 added, on top of the confirmed-working core**: AI now
actually uses the carrier decisions (Lesson 19 — they were hard-locked out
entirely before this), a real vanilla birth-naming widget (Lesson 21), and
Force/Forbid Carrier interactions for ordering another character
(liege→vassal/courtier or close family) to become or be barred from
becoming a carrier (Lesson 22). **None of these three are confirmed live
yet** — see the top-priority test list at the start of Section 5 before
doing anything else, including flavor content.

Still open, not solved yet: clothing clipping (partially mitigated), the
mixed-couple/male-male trait collision (Lesson 16/23, mitigated not
fixed), whether AI ever forms same-sex marriages/relations on its own
(unconfirmed either way), and whether reported newborn parentage is a real
data bug or a display-only limitation of a UI that's never had to show two
male parents before (Lesson 23 — verification tool added, not yet run).
See Section 4 and Section 5.

---

## 1. What this mod actually does

Vanilla CK3 hard-restricts pregnancy to female characters at the engine
level — the `make_pregnant` effect refuses anything that isn't female, no
exceptions, confirmed directly from error.log during early development.
That single restriction shaped this entire project's architecture.

**The core idea:** don't fight the restriction. Let the real, mechanical
pregnancy stay exactly where CK3 insists it has to be — on a female
character — untouched, so vanilla's own birth event, health effects, and
parentage all keep working correctly with zero risk of breaking them.
Layer a **purely cosmetic presentation** on top: when a female character in
an eligible relationship gets pregnant, redirect the *visual* presentation
(trait + growing belly) onto her male partner instead, while marking her to
hide her own belly for the duration. Only he visibly shows it.

This means, mechanically:
- Conception, health risk, and the birth event are 100% vanilla, untouched.
- The child's real parents are correctly recorded by vanilla itself.
- Everything this mod adds is additive/cosmetic on top of that.

## 2. Current file structure and what each piece does

```
common/traits/mpreg_traits.txt
    mp_pregnant, mp_carrier — purely cosmetic marker traits, no mechanical
    effects, real vanilla icons (pregnant.dds / fecund.dds). The real
    pregnancy's real effects stay on her. Stage (early/mid/late) is NOT a
    trait — it's read from var:mpreg_months directly; see Lesson 17.

common/genes/mpreg_genes.txt
    Extends vanilla's REAL gene_bs_pregnant gene (not a custom gene — see
    Lesson 3 below) with a "male" setting block, so the same body-shape
    attribute vanilla already uses for the female belly can also apply to
    a male character.

common/decisions/mpreg_debug_decisions.txt
    Dev/testing decisions: status report, force redirect, force tick,
    force stage jumps. Hidden (is_shown = { always = no }) as of the
    beta, not deleted — none of this is meant for normal play.
    mpreg_debug_status_decision is TEMPORARILY set back to always = yes
    (Lesson 23) to verify real newborn parentage data — revert once
    confirmed, see Section 5's top-priority test list.

common/traits/mpreg_trait_conversion.lookup
    Maps the now-deleted mp_pregnant_stage_1/2/3 trait keys to mp_pregnant
    for save-compatibility with pre-0.23 saves — see Lesson 17.

gfx/portraits/portrait_modifiers/mpreg_pregnancy.txt
    Three weighted stage entries applying the belly gene value (0.33 /
    0.66 / 1.0) based on var:mpreg_months (1/4/7 — see Lesson 17). Also
    contains an experimental cloak_offset clearance entry (see Section 4),
    and mpreg_hide_female_pregnancy (priority 1, overrides vanilla's own
    special_pregnancy group) which zeroes a real mother's own belly gene
    while her mp_carrier-flagged male spouse is showing the redirected one.

gfx/portraits/portrait_modifiers/mpreg_clothes_test.txt
    Forces the "leaf" outfit onto pregnant characters via a dedicated,
    mod-owned portrait modifier group (mpreg_clothes_special, priority 7)
    rather than reopening vanilla's clothes_special group. See Lesson 11
    below for why the original version silently did nothing.

gfx/models/portraits/male_body/blendshapes/male_bs_body_pregnant_1.mesh
    The actual sculpted belly blend-shape mesh. Vertex-count-matched to
    the real male_body_average base mesh (12,824 skin bone-index entries,
    confirmed identical). See Lesson 4/5 for what this required.

gfx/models/portraits/male_body/male_body.asset
    A FULL OVERRIDE of vanilla's real male_body.asset (verbatim copy, only
    two lines added: a blend_shape registration and an attribute mapping).
    This is the actual missing piece that made the belly render at all —
    see Lesson 6. Because this is a full-file override, it will conflict
    with any other mod that also overrides male_body.asset.

localization/english/mpreg_l_english.yml
    All user-facing text. See Lesson 7 for a real syntax gotcha here.

common/game_rules/mpreg_game_rules.txt
    Full redeclaration of vanilla's real same_sex_marriage game rule with
    default = accepted_same_sex_marriage. Deliberately does NOT touch the
    separate same_sex_relations game rule — see Lesson 12.

common/scripted_triggers/mpreg_marriage_triggers.txt
    Full redeclaration of vanilla's real allowed_to_marry_same_sex_trigger,
    dropping the doctrine block so shunned/illegal faiths can still marry
    same-sex couples (consequences are handled separately — see Lesson 12).

common/decisions/mpreg_carrier_decisions.txt
    Player-facing (not debug) decisions to add/remove the mp_carrier trait
    on yourself — the persistent "I'm eligible to carry a pregnancy"
    marker, independent of sex. Also gated on the mpreg_carrier_forbidden
    flag (see the interactions file below — Lesson 22).

common/character_interactions/mpreg_carrier_interactions.txt
    Force/Forbid Carrier — order another character (liege→vassal/courtier
    or close family) to become, or be barred from becoming, a carrier.
    First pass, unconditional (auto_accept), no acceptance mechanic yet —
    see Lesson 22.

common/on_action/mpreg_on_actions.txt
    Reopens vanilla's real on_marriage, on_pregnancy_mother, on_birth_mother,
    and on_set_relation_lover on_actions, appending our own events to their
    events lists only (the one database in this project confirmed to be
    safely cross-file additive — see Lesson 12).

common/script_values/mpreg_script_values.txt
    mpreg_ss_conception_chance — the male-male monthly conception odds,
    mirroring vanilla's own real FERTILITY_CHANCE_MULTIPLIER formula
    verbatim (see Lesson 14).

events/mpreg_events.txt
    mpreg.0001/0003 (mixed-couple real-pregnancy redirect and cleanup — see
    Lesson 13). mpreg.0002 (fired via on_marriage): grants vanilla's real
    secret_homosexual to both spouses of a male-male marriage, or a real
    stress consequence to both wives of a female-female one (vanilla has
    no female equivalent secret — see Lesson 24). mpreg.0004/0005/0006/
    0009/0007: the full male-male conception → pregnancy → birth chain —
    see Lesson 14, and Lessons 18/20 for the variable-reference crash
    fixes. mpreg.0008: the visible birth notification, fired one day after
    birth (not instantly — Lesson 23) with vanilla's real child-naming
    widget attached (Lesson 21).
```

## 3. Hard-won lessons (read this before touching genes/assets again)

These are the mistakes that cost the most time. Recorded so they don't
happen twice.

**Lesson 1 — `make_pregnant` is female-only, no exceptions.**
Confirmed via a real, repeated engine error: `make_pregnant effect [ Mother
must be female. '...' is not. ]`. This is not a bug, not moddable around.
It's why the whole project is built around redirecting *presentation*
rather than the mechanical pregnancy itself.

**Lesson 2 — reopening a specific named entry can silently wipe fields.**
When we first extended `gene_bs_pregnant`'s `pregnant` template with only a
`male` block, it silently dropped vanilla's `index` and `female` block —
breaking vanilla's own female pregnancy display as collateral damage.
Reopening a *container* key (like `on_actions`, or `clothes_special`) to
add a new, differently-named *sibling* entry is safe and additive. Reopening
and partially redeclaring an *existing specific entry* is not — it replaces
that entry wholesale. Always redeclare the *complete* entry, verbatim
original content included, when extending something specific.

**Lesson 3 — CK3 doesn't support new top-level gene categories.**
An attempt to declare a wholly new `gene_bs_male_mpreg` gene failed
immediately with `Unexpected token`, because all morph genes must live
nested inside a single top-level `morph_genes` wrapper (and some, like
pregnancy, inside `special_genes → morph_genes`). The fix that actually
worked: don't invent a new gene. Extend vanilla's real `gene_bs_pregnant`
with an added `male` setting block, reusing the confirmed real attribute
name `bs_body_pregnant_1`.

**Lesson 4 — the belly is not a Blender Shape Key.**
Confirmed by directly checking: vanilla's female pregnant body has *no*
Shape Key data at all. The deformation is a proper skeletal blend shape —
a real skinning skeleton (72 joints, matching vanilla's standard humanoid
rig naming: `ground_joint → body_root → bn_sp_lumbar → ...`) with the
belly geometry weight-painted to it, exported via io_pdx_mesh. No shape
key hunting needed; get the skeleton and weights right instead.

**Lesson 5 — vertex count must match the base body exactly.**
A `.mesh` built with Dyntopo/subdivision active produced 65,832 skin
bone-index entries versus vanilla's 12,824 — a ~5x mismatch. This alone is
enough to break rendering even with a perfect material and skeleton.
Sculpt on the base topology directly, no subdivision, no added geometry.
Confirmed working meshes match 12,824 exactly.

**Lesson 6 — blend shapes register in the BASE mesh's own .asset file, not
a standalone one.**
This was the single longest-running bug. A standalone `.asset` declaring
its own `pdxmesh` entry for the belly mesh will *always* throw `Failed to
create material`, no matter how correct the mesh itself is — because
that's structurally the wrong place to register it. The real mechanism:
`male_body.asset` (the main body's own asset file) contains a `blend_shape
= { id = "..." type = "blendshapes/....mesh" }` entry inside its `pdxmesh`
block, and a matching `attribute = { name = "..." blend_shape = "..." }`
entry inside its `entity` block. Confirmed by reading vanilla's actual
male_body.asset directly. Our mod now ships a full verbatim override of
that file with exactly these two lines added.

**Lesson 7 — CK3 loc syntax uses `#`, not `§`, and reserves `[...]` for
function calls.**
Two separate real bugs: using `§Y text §!` (wrong symbol) instead of `#Y
text#!`, and naming a decision `[DEBUG] ...` — square brackets are parsed
as an attempt to call a scripted loc function, producing `Could not find
data system function 'DEBUG'`. Both fixed; avoid both patterns going
forward.

**Lesson 8 — portrait modifier entries need their own loc key.**
Format: `PORTRAIT_MODIFIER_<group>_<entry>`. Missing it throws a loc
warning; unconfirmed whether it can also cause silent exclusion from
selection, but cheap to always add.

**Lesson 9 — `no_clothes` is not literal nudity, it's the leaf outfit.**
The gene template named `no_clothes` (in the `clothes` gene) resolves to
`male_clothes_secular_western_nudity_01` — the actual leaf/fig-leaf
outfit, not bare skin. This was a real misunderstanding early on; the
"nudity" naming is misleading.

**Lesson 10 — always verify file changes with real byte-level or
diff-based checks, not by eye, especially on large files.**
Multiple bugs (double-BOM, brace mismatches, unintended field loss) were
only caught by actually running Python-level checks (brace counting,
`difflib.unified_diff`, binary header inspection) rather than assuming an
edit was correct. Keep doing this for every future change.

**Lesson 11 — forcing clothing: don't reopen a vanilla portrait_modifiers
group, own a new one with an explicit priority instead.**
The original clothes test reopened vanilla's real `clothes_special` group
from a second file and just boosted a sibling entry's weight to 100000.
That mechanism was never actually confirmed to work for portrait_modifiers
specifically — every other confirmed-working piece of this mod
(`mpreg_visual_pregnancy`, `mpreg_cloak_clearance`) is its own brand-new
top-level group, never a reopen. Two things confirmed directly from
vanilla source while investigating:
- `should_show_nudity` is not a hard content-safety gate. It's an ordinary
  script trigger (synced from the client's "Show Nudity" interface
  setting) that only vanilla's own `no_clothes` entry happens to reference
  in its own weight math. It does nothing to entries that don't reference
  it. Confirmed via `localization/english/settings_l_english.yml`
  (`SETTING_SHOW_NUDITY`) and the accessory definition in
  `gfx/portraits/accessories/clothes.txt`, which carries no nudity/content
  flag at all. The original "blocked by a content-safety gate" theory was
  wrong.
- Every real vanilla clothes group has a fixed priority, and priority
  determines override order ("the higher the priority, the later the
  modifier is applied, and overwrites previous ones" —
  `_portrait_modifiers.info`): `clothes_base = 1`, `clothes_religious = 3`,
  `clothes_sickness = 3`, `clothes_armor = 4`, `clothes_situational = 5`,
  `clothes_special = 6`. A new group that doesn't set `priority` defaults
  to 0 and loses to all of them — meaning even a perfectly-selected entry
  in a new group would get silently overwritten by whatever normal
  clothes_base/situational/special picked afterward.

The fix: `mpreg_clothes_special` is now its own group with
`priority = 7` (one above vanilla's highest clothing priority, so it
always wins when it applies) and `ignore_outfit_tags = yes` (matching
every other working entry in this mod, removing any dependency on
outfit-tag/event context a normal character-window portrait may or may
not supply). Its single entry, weight-gated on `has_trait = mp_pregnant`
with `add = 100000`, mirrors vanilla's own real pattern for trait-forced
clothing almost exactly — see `charioteer_blue` / `pope_larper` /
`cardinal_larper` in `gfx/portraits/portrait_modifiers/05_clothes_situational.txt`,
which force specific outfits off a plain `has_trait`/`has_character_flag`
check with weight ~200–1000, no outfit_tags, no reopening — proof this
shape of entry reliably forces clothing in the live game.
Not yet re-tested in-game after this fix — see Section 5.

**Lesson 12 — same-sex marriage and a mixed-couple male carrier were
mostly a matter of finding what vanilla already has, not building new
systems.** Three real findings, all confirmed by reading vanilla source
directly before writing anything:
- Same-sex marriage is already a real vanilla game rule,
  `same_sex_marriage` (`common/game_rules/00_game_rules.txt:320-335`,
  options `default_same_sex_marriage` / `accepted_same_sex_marriage`), and
  a real per-faith doctrine split (`homosexuality_accepted` /
  `_shunned` / `_illegal`, `common/religion/doctrine_types/20_doctrines.txt:146-189`).
  We didn't need a custom toggle system — just override the game rule's
  `default` (full redeclaration, `mpreg_game_rules.txt`, same reasoning as
  Lesson 2). There's a second, separate rule, `same_sex_relations`, whose
  own effect (`game_rule_accepted_same_sex_relations_effect`,
  `common/scripted_effects/00_game_rule_effects.txt:765`) strips the
  homosexuality doctrine from every faith in the world at game start —
  the wrong tool here, since it destroys the shunned/illegal distinction
  the design wants to keep. Left untouched.
- Vanilla's `allowed_to_marry_same_sex_trigger` currently hard-blocks
  marriage entirely for shunned/illegal faiths (not just penalizes it) —
  confirmed by reading it directly, `00_marriage_triggers.txt:63-71`.
  Per design decision, neither should fully block marriage; shunned
  should penalize, illegal should criminalize. Fixed by fully
  redeclaring the trigger (`mpreg_marriage_triggers.txt`) with the
  doctrine check removed — scripted triggers are flat named macros with
  no cross-file merge, so reopening one with the same name is a full,
  deliberate replace (same behavior as Lesson 2, confirmed for this
  database too).
- Vanilla already has a real, deep secret/crime mechanic exactly matching
  what "penalize shunned / criminalize illegal" needs:
  `secret_homosexual` (`common/secret_types/00_secret_types.txt:45-79`),
  whose `is_shunned`/`is_criminal` checks are already wired to the exact
  same two doctrine parameters
  (`00_secret_type_triggers.txt:45-61`), and which plugs into vanilla's
  existing blackmail interaction (`00_blackmail_interactions.txt:352`)
  and dozens of exposure/opinion events. Rather than build a parallel
  penalty system, `mpreg.0002` (fired via `on_marriage`) just calls
  vanilla's own `give_homosexual_secret_or_nothing_effect`
  (`00_secret_effects.txt:279-294`) on both spouses of a same-sex
  marriage — that effect already self-checks adult/male/not-already-secret/
  shunned-or-criminal-in-faith, so it's safe to call unconditionally and
  reliably grants the secret instead of waiting on random flavor events.
- Confirmed real conception mechanics for the mixed-couple carrier
  redirect too: vanilla's actual pregnancy roll is `had_sex_with_effect`
  (`common/scripted_effects/00_romance_effects.txt:361-579`, called from
  many flavor events, not a monthly tick), a flat 30%
  (`pregnancy_chance`, `common/script_values/00_basic_values.txt:703`)
  gated by `possible_pregnancy_after_sex_with_character_trigger`
  (`00_romance_and_seduction_triggers.txt:726-755`), which checks
  `is_visibly_fertile` + `fertility >= 0.1` + `is_pregnant = no` — **on
  both partners regardless of sex already**, since men need fertility to
  count as fathers. There's no separate male-carrying-child branch
  though, and `make_pregnant` is still hard female-only (Lesson 1) — so
  for a mixed couple the *mechanical* pregnancy still has to happen on
  the woman; we only redirect the *cosmetic* presentation, via a new
  `on_pregnancy_mother` hook (`child_birth_on_actions.txt:1169`, fires
  once a real pregnancy reaches "revealed" status) that checks for an
  `mp_carrier`-flagged male spouse and, if found, applies the same
  trait/stage/variable logic the debug decisions already used, at a stage
  computed from the real, native `pregnancy_days` value (also used by
  vanilla's own belly morph calc, `99_special.txt:166-212`) instead of
  always restarting at stage 1. Her own belly is suppressed by a new,
  higher-priority portrait modifier group rather than any new trait on
  her — see the `mpreg_pregnancy.txt` file entry above.
- One thing confirmed *safe to rely on*: `on_action` `events = { }` lists
  (used for both `mpreg.0001` and `mpreg.0002`) are the one database in
  this whole project confirmed to be genuinely, safely additive across
  files/mods — unlike scripted triggers, gene templates, or
  portrait_modifiers groups, which all fully replace on reopen.

**Lesson 13 — a redirect that only fires once needs its own upkeep and its
own cleanup; don't assume the hook that starts something also ends it.**
Caught on review, not in-game: the first version of `mpreg.0001` (Lesson 12)
correctly redirected onto a carrier at the moment a real pregnancy became
"revealed," but `on_pregnancy_mother` only ever fires *once* per pregnancy
(confirmed — nothing in script calls it a second time; vanilla's own
`pregnancy.0001` schedules the eventual birth event itself via
`trigger_event`, it doesn't rely on `on_pregnancy_mother` firing again). Two
real consequences that would only have shown up hours into a real playthrough:
the carrier's stage would freeze at whatever it was on reveal day and never
advance toward stage_2/stage_3, and nothing ever removed `mp_pregnant` from
him after birth, leaving him permanently "pregnant." Fixed two ways, both
using only hook names confirmed real by directly reading vanilla source
first: `mpreg.0001` now reschedules itself every 30 days via
`trigger_event = { id = mpreg.0001 days = 30 }` (exact syntax confirmed
against vanilla's own use of the same effect, `events/pregnancy_events.txt:62-65`)
for as long as `root = { is_pregnant = yes }`, re-syncing the carrier's stage
each time; and a new `mpreg.0003`, hooked on `on_birth_mother`
(`child_birth_on_actions.txt:8`, confirmed to fire for both the normal-birth
and mother-dies cases), immediately clears the carrier's trait/stage/variable
the moment birth actually resolves, rather than waiting up to 30 days for the
reschedule chain to notice. Root stays the mother through this entire chain,
deliberately never scope-switched to the carrier, matching vanilla's own
`pregnancy.0001 → pregnancy.1001/2050` pattern — sidesteps any question of
whether saved scopes survive a `trigger_event` call.

One related, *not* fixed, inherited-from-vanilla limitation, worth knowing
about rather than assuming is a bug: vanilla's own `give_homosexual_secret_or_nothing_effect`
(`00_secret_effects.txt:279-294`) requires `is_male = yes` internally, so
`mpreg.0002`'s "grant the secret to both spouses" only ever actually grants
anything for male-male marriages — a female-female marriage under this mod's
now-always-on same-sex marriage rule gets no shunned/criminal consequence at
all, purely because vanilla itself never built a female equivalent of that
secret. Not addressed; flagged here in case symmetric treatment is wanted
later.

**Lesson 14 — male-male conception/pregnancy/birth, built from a real,
console-generated `effects.log`, not guesswork.** The user ran
`-debug_mode` → console `script_docs` and shared the result
(`logs/effects.log`, generated fresh, ~548KB). One practical gotcha hit
immediately: the file has non-UTF-8 bytes (cp1252 em-dashes), which makes
`grep`'s binary-file heuristic silently skip it with zero matches and zero
error — looked exactly like the file was empty of "character"/"father"
content until read with `encoding='cp1252'` in Python instead. Worth
remembering for any future CK3 log inspection, not just this file.

What that log actually confirmed:
- `create_character` takes `mother =` and `father =` **directly as
  parameters** — no separate two-step effect needed to create a child with
  specific parents (contradicts this project's own earlier "unconfirmed
  guesswork" citation of a `create_character` + `set_father`/`set_mother`
  two-step pattern; that pattern *also* turned out to be real —
  `set_father`/`set_mother` exist too, confirmed in the same log, but for
  reassigning parentage on an *already-existing* character, not creation).
  Real vanilla usage of both `mother =`/`father =` together, verbatim,
  found in `common/scripted_effects/00_pregnancy_effects.txt` (Tokihito's
  birth) — this project's implementation copies that example's field
  choices (`dynasty_house`/`faith`/`culture` from the father,
  `create_character_memory` afterward) directly.
- `mother`/`father` are each independently optional (real vanilla example,
  `10_dlc_tgp_japan_scripted_effects.txt`, creates several children with
  only `father =` set, no mother at all) — nothing in the real
  documentation enforces a sex check on either slot, unlike `make_pregnant`
  (Lesson 1). Per design decision, the carrier fills the `mother =` slot
  even though male, reusing vanilla's existing DNA/ethnicity/inheritance
  blending exactly as-is rather than risking an unconfirmed alternate path
  through `real_father`.
- Vanilla's real monthly conception formula, found in
  `common/defines/00_defines.txt:372-374` (not something exposed via a
  script trigger — this is an engine define, only discoverable by reading
  the actual defines file): `FERTILITY_CHANCE_MULTIPLIER = 4.75`,
  `MIN_FERTILITY_CHANCE = 1`, `MAX_FERTILITY_CHANCE = 25`, checked *every
  month* — `chance% = clamp(avg(mother_fertility, father_fertility) ×
  4.75, 1, 25)`. This directly superseded an earlier, much rougher
  research estimate (a fork's static analysis of `had_sex_with_effect`'s
  ~60 flavor-event call sites, which could only bound the aggregate
  organic rate to roughly 1–5%/year without finding this formula) — real
  defines beat estimated pool weights every time they're available.
  `mpreg_ss_conception_chance` (`common/script_values/mpreg_script_values.txt`)
  reuses this formula verbatim rather than any invented number, satisfying
  the design ask ("keep variables like fertility into account") with zero
  new balancing logic.
- `set_variable`'s value field accepts "any event target" (confirmed
  directly from the log's own `set_variable` doc entry, not inferred) —
  meaning a character reference can be persisted in a variable and
  survive indefinitely, which is what makes a self-rescheduling chain
  spanning months of game-time possible without relying on temporary
  scopes (whose survival across separate `trigger_event` firings was never
  confirmed one way or the other — sidestepped entirely by using
  `var:mpreg_ss_partner` / `var:mpreg_ss_other_father` instead of saved
  scopes for anything that has to outlive a single event).
- No bulk "iterate my lovers" scope exists in vanilla (checked: zero real
  usage of any `every_relation_lover`/`any_relation_lover`/
  `random_relation_lover` anywhere) — only the single-target
  `has_relation_lover = CHARACTER` trigger. This is *why* the lover
  partner has to be captured into a variable at the moment
  `on_set_relation_lover` fires, rather than re-derived by iteration each
  month the way the mixed-couple case re-derives via `any_spouse`
  (spouses, unlike lovers, do have a real bulk iterator).

The resulting chain (`mpreg.0004`/`0005` start it on marriage/becoming
lovers → `mpreg.0006` self-reschedules monthly, rolling
`mpreg_ss_conception_chance` whenever eligible and nobody's already
pregnant → on success, `mpreg.0009` self-reschedules monthly tracking a
custom day-counter (no engine `pregnancy_days` exists for a custom
pregnancy) through the same 120/210/280-day stage thresholds already used
elsewhere → `mpreg.0007` fires the real `create_character` call at day 280
→ `mpreg.0008` is a simple visible notification, with a full
`birth.0001`-style vanilla-flavor event explicitly deferred, per design
decision, not forgotten) is detailed in `events/mpreg_events.txt`'s own
inline comments.

One simplification made deliberately, not silently: the earlier locked-in
design decision for the "both partners flagged carrier" edge case was
*weighted* random by fertility. The actual implementation uses a plain
higher-fertility-wins comparison instead — a true weighted roll would need
a `random_list` entry weight computed from a live character value, and
while `random_list`'s real documented syntax (confirmed from the same
`effects.log`) supports `modifier` blocks adjusting a base weight, whether
`add` inside that modifier accepts a nested computed `value = {}` block
the way `weight =` blocks elsewhere in this project do was never
confirmed against a real example. Given this only matters for a narrow
edge case, the safer, unambiguous comparison was used instead. Revisit
with real confirmation if it turns out to matter in practice.

**Not yet tested in a live game.** Every syntax choice above is grounded
in either a real, quoted vanilla example or the real, console-generated
effects documentation — but the chain's *runtime behavior* (does the
36-months-plus self-reschedule chain actually keep firing reliably across
that much game-time, does the monthly roll feel right, does birth actually
produce a healthy, correctly-parented child) is unverified. This is the
top item in Section 5.

**Lesson 15 — real error.log is directly readable from the filesystem;
check it before theorizing, and check it early.** First live test of the
male-male system (v0.20, 6 in-game years): marriage worked, many
`mp_carrier` characters conceived, but all got stuck in early pregnancy for
years, and the one that reached late pregnancy never gave birth. Rather
than guess at engine-reliability theories, the actual game logs
(`~/.../Crusader Kings III/logs/error.log`, `debug.log`) were read
directly — they're plain files on disk, no need to ask the user to paste
anything. That surfaced two real, confirmed bugs immediately, plus one
false lead worth recording so it isn't repeated:

- **`any_spouse` used as an *effect*, not a trigger — real error:
  `"Unknown effect: any_spouse"`.** `any_spouse` is trigger-only (boolean
  "does any spouse match X"); the effect-context equivalent for "iterate
  and act" is `random_spouse` (or `every_spouse`). This mistake was in two
  places: `mpreg.0001`'s "already redirected" fallback branch, and all of
  `mpreg.0003`. The real damage was much bigger than one wrong keyword,
  though — this single error **cascaded**, corrupting the parser's
  position for everything declared later in the same file: 8 separate
  `"Unexpected token: mpreg.000N"` errors followed (one for every event
  declared after `mpreg.0001`), and 3 of those events (`mpreg.0002`,
  `mpreg.0003`, `mpreg.0005`) ended up completely unregistered
  (`"Invalid event id"` at their `on_action` hookups) — meaning the
  same-sex-marriage secret grant, the mixed-couple birth cleanup, and the
  lover-relation conception starter **silently never ran at all** in that
  test, despite loading without any visible crash. Fixed by correcting
  both `any_spouse` → `random_spouse` occurrences. **Takeaway: brace
  balance (Lesson 10) proves a file's braces match; it does NOT prove
  every keyword is being used in a context the engine actually accepts.
  Only the real error.log (or the engine's own parser) can catch a
  trigger-used-as-effect mistake — verify against it, not just structure,
  for any construct without a directly-confirmed real usage example.**
- **`mode = replace` is invalid for special/morph genes — real error:
  `"modes modify, modify_multiply and replace only work for regular
  genes"`.** `mpreg_hide_female_pregnancy` (the entry meant to zero out a
  redirected mother's own belly) used `mode = replace value = 0.0` on
  `gene_bs_pregnant`, a special gene — silently never actually suppressed
  her belly (though this is unrelated to the male-male symptoms just
  reported; it affects the mixed-couple case instead). Fixed by switching
  to `mode = add value = -2.0`, relying on the template's own declared
  `min = 0.0` to floor the accumulated total regardless of what vanilla's
  `special_pregnancy` group already contributed (itself capped at 1.0).
- **False lead, corrected before reporting:** the log also showed 7
  `"Invalid location (not a province, or not provided)"` errors from
  `create_character`, which looked at first glance like exactly what
  would explain a stalled birth (`mpreg.0007` used
  `employer = root.employer`, and a landed carrier generally has no
  `employer`). Tracing each error's actual `Script location:` line (not
  just the error text) showed all 7 came from vanilla's own
  `00_accolades_scripted_effects.txt` (the unrelated Accolades
  squire-creation system) — **none from this mod.** `employer =` was
  still switched to `location = root.location` anyway (confirmed real,
  exact match to vanilla's own `create_character` usage,
  `common/on_action/yearly_on_actions.txt:1199`), since it's strictly
  safer regardless — but this was not the confirmed cause of anything
  observed, and is recorded here specifically as a caught near-miss:
  **trace the actual script location of an error before attributing it to
  your own code, don't pattern-match on "mentions an effect I also
  used."**
- **What this does *not* fully explain:** neither `debug.log` nor
  `error.log` contains any trace of `mpreg.0007`/`mpreg.0009` ever firing
  successfully either — meaning it's still genuinely unknown whether the
  "stuck in early pregnancy for years" characters were failing to
  reschedule, or simply hadn't had enough elapsed in-game time yet since
  their own individual conception (pregnancies started at different
  points across 6 years would legitimately show different current
  stages). The `any_spouse` fix removes the one *confirmed* source of
  corruption; whether it's sufficient on its own needs a fresh test, with
  `error.log` checked again afterward the same direct way.
- Also fixed while in here: all mpreg `.txt` files were missing their
  UTF-8 BOM (non-fatal `"should be in utf8-bom encoding"` warnings for
  every one of them) — added, verified single (not double, Lesson 10) on
  every file.

**Lesson 16 — trigger `limit` blocks in CK3 do NOT short-circuit; an
`exists`/`has_variable` guard earlier in the same flat list does not
protect a later reference to that same variable.** Second real test (v0.21,
10 years): birth actually worked this time, but `error.log` showed the
same variable, `mpreg_ss_partner`, throwing `"Failed to fetch variable ...
due to not being set"`, `"Event target link 'var' returned an unset
scope"`, and `"Invalid right side during comparison 'var'"` — **100
times**, all traced to one line (`events/mpreg_events.txt:236`,
`any_spouse = { this = var:mpreg_ss_partner }`, inside `mpreg.0006`'s outer
`limit`). That line sits in the *same flat AND-list* as an earlier
`exists = var:mpreg_ss_partner` check that was supposed to guard it — the
assumption that a failed `exists` check would prevent evaluation of a
later condition in the same list was wrong. Every condition in a flat
`limit = { A B C }` gets evaluated regardless of whether an earlier one
already failed; only *nested* `if = { limit = { A } if = { limit = { B } } }`
actually gates B on A. This matches the file's other real, confirmed
lesson about short-circuiting-that-isn't (see the earlier note on scope
survival across `trigger_event`) — CK3 script is a declarative rule
system, not sequential imperative code, and should be read that way even
when a construct *looks* like an early-exit guard.

Most likely real-world trigger for this, given it fired 100 times across
a single 10-year test with many couples: a stored partner reference
becoming invalid mid-chain because the referenced character died and CK3
scrubbed the now-dangling reference — not something that would show up in
a short test, only in exactly the kind of long, multi-couple simulation
that was actually run. Fixed by restructuring `mpreg.0006` into properly
nested `if`s: outer gate is `is_alive = yes` + `has_variable =
mpreg_ss_partner` (the same safe existence check already used elsewhere
in this mod, `mpreg_debug_status_decision`), and nothing inside ever
touches `var:mpreg_ss_partner` unless that outer gate already confirmed it
exists. The same defensive `has_variable` guard was added pre-emptively to
`mpreg.0009` before it dereferences `var:mpreg_ss_other_father` to fire
birth — same category of risk (the other father could die during the
~9-month gestation window), not yet observed failing, but now guarded the
same way rather than waiting to find out. Known residual gap: if that
guard trips (father died), the pregnancy currently just stalls
indefinitely at day ≥280 rather than resolving some other way — documented
as a known limitation, not fixed further, since a graceful fallback (birth
without a recorded father? cancel the pregnancy?) is a real design
decision, not a bug fix, and hasn't come up in practice.

Separately reported in the same test: with a mixed (male+female) couple
where the man had *also* independently become pregnant via the male-male
system with a different partner, his birth (via that unrelated
relationship) appeared to affect his wife's own separate pregnancy. Root
cause not fully confirmed yet — both systems share the same `mp_pregnant`/
stage traits to drive the same portrait modifiers, and
`mpreg_hide_female_pregnancy`'s suppression check
(`any_spouse = { is_male = yes has_trait = mp_pregnant }`) can't
distinguish "my spouse's `mp_pregnant` represents *my* redirected
pregnancy" from "my spouse's `mp_pregnant` represents an unrelated
pregnancy with someone else." Tightened with an added `is_pregnant = yes`
(her own real state) requirement, which narrows the false-positive window
but doesn't structurally eliminate it — the real fix would need the
redirect to carry an explicit "this mp_pregnant belongs to *this specific*
relationship" marker rather than relying on the trait alone, which hasn't
been designed yet. Flagged in Section 4.

**Lesson 17 — CK3 traits have no real "hidden from character screen"
field; if something needs to be truly invisible, track it as a variable,
not a trait.** Beta prep asked for the stage traits
(`mp_pregnant_stage_1/2/3`) to be hidden from the player. Checked the
authoritative source first (`common/traits/_traits.info`, the same real
documentation file used for every other trait-field question in this
project) rather than guessing a `hidden = yes` field exists by analogy to
other systems (portrait modifiers, decisions, etc. all have their own
visibility fields, but traits don't share that pattern) — it documents
exactly two visibility toggles, `shown_in_encyclopedia` and
`shown_in_ruler_designer`, neither of which controls the character's own
trait list/tooltip. There is no field that hides an active trait from
where it actually shows up. Since `var:mpreg_months` (1/4/7) already
carried the exact same information as the three stage traits, the real
fix was to delete the stage traits entirely and drive the portrait
modifier weights (`gfx/portraits/portrait_modifiers/mpreg_pregnancy.txt`)
directly off `var:mpreg_months = 1/4/7` instead of `has_trait =
mp_pregnant_stage_N` — genuinely invisible, not just visually
suppressed, and it deleted code rather than adding a workaround (every
`add_trait`/`remove_trait` stage_N call across `mpreg_events.txt` and
`mpreg_debug_decisions.txt` was redundant with the `set_variable
mpreg_months` call already sitting right next to it, so removing the
trait calls actually shrank the file). `mp_pregnant` and `mp_carrier`
stay as real, visible traits (per design decision — they're the whole
point, players need to see them), now with real vanilla icons
(`pregnant.dds`, `fecund.dds` — `icon = "<file>.dds"` confirmed as a real
trait field from the same `_traits.info`, several real vanilla traits use
it to point at a different file than their own name). Debug decisions
(`mpreg_debug_decisions.txt`) were hidden via `is_shown = { always = no }`
rather than deleted, so they stay available for future dev sessions.

**Lesson 18 — `has_variable` was the wrong fix for Lesson 16's crash; got
it wrong once, caught it from a fresh error.log, fixed it correctly the
second time.** A live test (10 years, real ruler) reproduced the exact
same `"Failed to fetch variable for 'mpreg_ss_partner' due to not being
set"` crash Lesson 16 claimed to have fixed — same file, same line,
still ~250 occurrences. Investigated instead of re-guessing: pulled the
real, console-generated `triggers.log` and compared the two candidates'
actual documented behavior. `has_variable` — "Checks whether the current
scope has any of the specified variables set." `exists` — "Checks whether
the specified scope target exists (check for not being the null object)."
These are different questions. A character-valued variable can have its
*slot* still set (`has_variable` = true) while the *character it points
to* no longer resolves to anything (fully removed from the game, not
just dead) — `has_variable` cannot see that difference, `exists` can.
Lesson 16's fix used the wrong one. Re-fixed properly: `exists =
var:mpreg_ss_partner`, alone in its own outer `limit` (per Lesson 16's
still-correct finding that CK3 `limit` blocks don't short-circuit — a
second reference to the same variable in the same flat limit throws
regardless of an earlier `exists` check), with every subsequent reference
nested one level deeper each time a *new* var-dependent condition is
introduced, not just once at the top. The original fix only nested one
level, then put two different references to the same variable back in
one flat inner limit — same mistake, one level down. `mpreg.0009`
(day/birth tracker) had the identical `has_variable` mistake for
`var:mpreg_ss_other_father`, and it's the direct, near-certain explanation
for the "pregnant for over two years" ruler reported in the same test:
`has_variable` doesn't check the father's continued existence, so
`create_character` would fire on a stale reference, `create_character`
would fail, and everything after it in that effect block — including the
`remove_trait = mp_pregnant` cleanup — would never run, leaving him stuck
pregnant permanently with no further reschedule. Rather than just gate
this one too, `mpreg.0007` (birth) was made structurally resilient
instead: it now branches on `exists = var:mpreg_ss_other_father` and
completes the birth either way, falling back to the carrier's own
dynasty/faith/culture if the father is genuinely gone, so a missing
father can never again leave a pregnancy stuck forever.

**Lesson 19 — AI never used the carrier decisions at all; `ai_potential =
{ always = no }` meant exactly that, literally.** Reported: no
naturally-occurring same-sex marriages, and the player's own carrier
attempt found nothing after years of trying. The second is very plausibly
just the Lesson 18 crash cutting the "trying" chain short before it ever
got a fair number of roll attempts — worth retesting now that it's fixed
before assuming pure bad luck. The first is a separate, real, structural
gap: `mpreg_carrier_decisions.txt` had AI locked out of both decisions
entirely since they were first written (`ai_potential = { always = no }`),
so AI characters could never become carriers on their own, regardless of
their marriage/relationship situation — this is upstream of and
independent from same-sex marriage/AI-pairing behavior, which remains
separately unconfirmed (see Section 4). Fixed with real, vanilla-confirmed
`ai_will_do`/`modifier` syntax (`common/decisions/00_fp3_decisions.txt`
pattern: `base` + additive `modifier` blocks), gated to `is_male = yes`
for AI specifically (not `is_valid`, which stays open to any sex for the
player) since `mp_carrier` is only ever functionally consumed by a male
character today (both `mpreg.0001` and `mpreg.0006` filter candidates on
`is_male = yes`) — flagging it on a female AI character would be
cosmetic-only clutter. Age thresholds (30/40, favoring younger characters
becoming carriers and older ones stopping) are direct design values from
the user, not derived from anything vanilla.

**Started: researching vanilla's real birth-naming mechanism**, for the
"use vanilla's birth event to choose a name" TODO. Confirmed real and
complete, not guessed: vanilla's actual child-naming UI is a `widgets = {
widget = { ... } }` block at the event's top level (`events/birth_events.txt`,
`birth.1001` and others), not a scripted option/decision:
```
widgets = {
	widget = {
		is_shown = {
			allow_naming_on_birth_of_child_trigger = { CHILD = scope:child }
		}
		gui = "event_window_widget_name_child"
		container = "dynamic_birth_name"
		controller = name_character
		setup_scope = { scope:child = { save_scope_as = name_character_target } }
	}
}
```
`allow_naming_on_birth_of_child_trigger` (`common/scripted_triggers/00_birth_triggers.txt:13`)
requires (among other things) `is_close_family_of = $CHILD$` from the
character it's evaluated on — should hold naturally for our case since
`create_character`'s `mother = root` already establishes that family
relationship the same way vanilla's own birth flow does. Not yet
implemented — needs the full real `option`/twin-handling pattern from
`birth.1001` worked out completely before writing anything, same rule as
everywhere else in this file. See Section 5.

**Lesson 20 — third attempt at the `mpreg_ss_partner` crash; Lesson 18's
fix was also wrong, and the real fix needed a structural change, not
deeper nesting.** A third live test (10 more years, AI carriers now
working well, lots of pregnancies observed) still showed the identical
crash — 526 occurrences, same file, same error text, this time at
`events/mpreg_events.txt:243` (`any_spouse = { this = var:mpreg_ss_partner }`)
— despite that exact line sitting *two* "if" levels below an already-passed
`exists = var:mpreg_ss_partner` check, which by Lesson 18's own (reasonable
at the time) logic should have been safe. It wasn't. Revised theory,
consistent with three real data points now: CK3 appears to revalidate a
bare `var:X` reference every time it's *textually* used, independent of
how deeply it's nested under an existence check earlier in the same
event — nesting protects which *effects* run, but does not appear to
protect repeated *variable lookups* from being independently
re-attempted. (Not fully proven — no official documentation confirms
this mechanism directly — but it's the simplest explanation that fits
all three failures and the fix built on it worked structurally, see
below.) The actual fix: stop trying to gate repeated `var:X` references
with nesting at all. Instead, dereference `var:mpreg_ss_partner` exactly
**once**, immediately after `exists` confirms it's safe, convert it to
`scope:mpreg_partner` via `save_scope_as`, and never write
`var:mpreg_ss_partner` again anywhere below that point in the event — a
saved scope behaves like any other scope reference already used safely
throughout this file (`scope:spouse`, `scope:carrier`, etc.) and can be
referenced repeatedly without issue; only the repeated *variable* lookup
was ever the problem. Applied the same pattern to `mpreg.0007`'s
`var:mpreg_ss_other_father` pre-emptively (converted to
`scope:mpreg_other_father` before use in `create_character`), even though
that one hadn't been observed crashing — same shape of risk, fixed before
it needed a fourth attempt rather than after. **This is genuinely
unconfirmed until the next real test** — said honestly, not claimed as
fixed pre-verification, given the last two attempts were each confidently
declared fixed and both were wrong.

**Lesson 20 confirmed working**: next live test after the scope-conversion
fix — pregnancy succeeded, birth fired, the visible notification appeared
correctly ("A Child is Born"). Male-male conception→pregnancy→birth is now
confirmed working end to end with the crash fixed, not just "probably."

**Lesson 21 — birth-naming widget implemented, using the real mechanism
researched in Lesson 19.** Added to `mpreg.0008` verbatim from vanilla's
own `birth.1001` (`gui = "event_window_widget_name_child"`, `container =
"dynamic_birth_name"`, `controller = name_character`, `setup_scope`
pattern) — only the scope name changed (`scope:mpreg_child`, matching this
file's existing convention, instead of vanilla's `scope:child`). No twin
handling needed (this system only ever produces one child per birth).
Confirmed real and complete before writing: `allow_naming_on_birth_of_child_trigger`
(gating the widget's `is_shown`) already requires `is_ai = no` and
`is_close_family_of = $CHILD$` from whoever it's evaluated on — root here
is the carrier, and `mpreg.0007`'s `create_character mother = root`
establishes that family relationship the same way a real birth does, so
no extra guard was needed. **Not yet live-tested** — the field should
appear on next birth; confirm before considering this fully done.

**Lesson 22 — Force/Forbid Carrier interactions added
(`common/character_interactions/mpreg_carrier_interactions.txt`), first
pass per direct design input.** Full field reference read from the real
`_character_interactions.info` before writing anything (same discipline
as every other new construct in this project). Design decisions locked in
directly by the user: `auto_accept = yes` for both (no
acceptance/hooks/gold/influence mechanic yet — explicitly deferred, "maybe
later"); targeting restricted to liege-over-vassal/courtier or close
family, using the real `?=` soft-comparison operator (confirmed from
vanilla's own `grant_court_position`, `liege ?= scope:actor`, which
doesn't error for a recipient with no liege) so independent rulers and
non-employed characters don't throw errors when checked; no opinion
penalty for Force in this pass. "Forbid" is a persistent block —
deliberately a character **flag** (`mpreg_carrier_forbidden`), not a
trait, to avoid Lesson 17's "no real hidden trait" problem (flags don't
appear anywhere in the character's trait list; `add_character_flag`/
`has_character_flag`/`remove_character_flag` already confirmed real
elsewhere in this project). Wired into `mpreg_carrier_decisions.txt`'s
Become a Carrier decision (`is_shown`/`is_valid`/`ai_potential` all gated
on `NOT has_character_flag = mpreg_carrier_forbidden`), so it blocks both
player and AI re-adoption. Force deliberately clears the forbidden flag —
there's no separate "lift the forbid" action, Force already serves that
purpose. Forbid is blocked on an already-pregnant target (excluded via
`is_valid_showing_failures_only`) — forcibly ending an active pregnancy is
a materially bigger, separate design decision not attempted here. No
perfect vanilla icon exists for "forbid" specifically (checked the real
icon folder directly rather than guessing); `demand_obedience.dds` reused
for both since it reads reasonably for either direction. **Not yet
live-tested** — the interaction should appear when right-clicking an
eligible male vassal/courtier/family member; confirm both directions
actually apply/clear the trait and flag correctly, and that the decision
correctly disappears for a forbidden character.

**Lesson 23 — naming widget silently didn't appear, and reported parentage
looked wrong; investigated both, fixed one, gave a real way to verify the
other rather than guessing at it.** Live test: interactions (Lesson 22)
worked, but the naming field never showed (just the plain "Wonderful"
popup), and checking the newborn's family showed only the husband as a
parent, not the carrier (the player's own character).

- **Naming widget**: `character_event.gui` genuinely contains a
  `dynamic_birth_name` container in the same base window `type =
  character_event` already uses (confirmed by reading the actual GUI file,
  not assumed) — so the earlier fear from Lesson 21 (wrong window type)
  wasn't it. Leading theory instead: `mpreg.0008` fired in the *exact same
  script tick* as `create_character`, with zero settling time, while
  vanilla's own version of this widget only ever fires from
  `on_birth_mother` — an engine-native lifecycle point reached well after
  a real birth's family graph has already settled. The widget's own
  `is_shown` gate (`allow_naming_on_birth_of_child_trigger`) checks
  `is_close_family_of`, which may not yet recognize a `create_character`
  relationship established in the same instant. Fixed by delaying
  `mpreg.0008` by one day (`trigger_event = { id = mpreg.0008 days = 1 }`)
  instead of firing it bare/immediately — which meant the child reference
  could no longer be carried as `scope:mpreg_child` (temporary scopes are
  not confirmed to survive a delayed `trigger_event`, per Lesson 13/16's
  standing caution), so it's now carried as `var:mpreg_ss_child` and
  dereferenced once into a scope in `mpreg.0008`'s own `immediate` block
  instead (same one-time-dereference pattern as Lesson 20). **Not yet
  live-tested** — this is a reasoned theory, not a confirmed fix.
- **Parentage**: no `create_character` error appeared in `error.log` for
  our own event (the two "Invalid location" hits present were confirmed,
  again, to be the same unrelated vanilla Accolades bug from Lesson 15) —
  meaning the underlying `mother =`/`father =` assignment most likely
  *did* execute without a script-level failure. Rather than guess whether
  this is a real data bug or a display-only limitation (CK3's own family
  tree UI has never had to render two male parents before — this mod is
  the only thing that can produce that combination, since vanilla's
  `make_pregnant` is hard female-only per Lesson 1), added real,
  ground-truth verification instead: `mpreg_debug_status_decision` now
  also reports `exists = mother` / `father` / `real_father` (all confirmed
  real trigger scope-links, `common/scripted_triggers/00_bastard_triggers.txt`
  and `00_birth_triggers.txt`) with the linked character's name via
  `GetMother`/`GetFather`/`GetRealFather` (all confirmed real from actual
  vanilla loc files, not guessed). **Temporarily set back to `is_shown =
  { always = yes }`** so it can be used directly on the child to check
  what the engine actually recorded — revert to `always = no` once
  confirmed. This will cleanly separate "our script got the parents wrong"
  from "the UI just doesn't know how to show this," which determines
  whether the next fix is a script change or an accepted, documented
  engine limitation.

**Lesson 24 — female-female marriages now get a real consequence too.**
Confirmed (checked, not assumed): vanilla has no female equivalent of
`secret_homosexual` anywhere in `common/secret_types/` — no "sapphist" or
similar entry exists. Building a whole new secret type + trait just to
mirror the male path was more than a "minor feature" deserved, so
`mpreg.0002` now applies a direct consequence instead, reusing the real
`homosexuality_shunned`/`homosexuality_illegal` doctrine parameters
already established in Lesson 12: `medium_stress_impact_gain` (40) for
shunned, `major_stress_impact_gain` (80) for illegal, both real vanilla
constants (`common/script_values/00_stress_values.txt:27-29`), applied to
both wives via the real bare `add_stress = X` effect. Deliberately a
smaller consequence than the male path (an immediate stress hit vs. a
standing, blackmailable secret) — not attempting to fully replicate a
system vanilla itself never built for women. **Not yet live-tested.**

**Lesson 25 — the parentage bug is real (confirmed via heir eligibility,
not just UI), the naming widget failure was very likely the *same* root
cause, and the obvious-looking fix (`real_father`) would have made things
worse.** Live test results: the child born to a male-male couple counted
the other father as a parent but not the carrier, and consequently
**could not be designated as the carrier's heir** — a mechanical
consequence, not a display quirk, which rules out Lesson 23's "UI has
never had to show two male parents" theory. Separately, the naming widget
still didn't appear even with Lesson 23's one-day delay fix. These are
very likely the *same* bug: the naming widget's gate,
`allow_naming_on_birth_of_child_trigger`, requires `is_close_family_of`,
which almost certainly depends on the exact parent-child relationship that
appears to be missing — meaning Lesson 23's delay theory was solving the
wrong problem (there was never a settling-time issue; the relationship
was likely never established at all, delay or no delay).

Investigated properly before touching anything, since this project has
already been burned three times this session by confidently-declared
fixes that weren't (Lessons 16/18/20):
- Confirmed real, from the actual console-generated `triggers.log`:
  `any_parent`/`is_child_of` are documented as generic, sex-agnostic
  relationship checks ("iterate through all (both) parents") — so if the
  underlying data were correct, sex shouldn't matter to these checks. This
  doesn't prove the data *is* correct, only that these specific triggers
  aren't the reason it wouldn't be recognized if it were.
- Seriously considered switching to `father =` + `real_father =` (both
  male, no `mother =` slot) as Lesson 14 had originally floated. **Ruled
  out as actively harmful**: `real_father != father` is literally vanilla's
  own real bastardy-detection condition
  (`common/scripted_triggers/00_bastard_triggers.txt:76`) — using this
  pattern would likely flag the child illegitimate, which would make
  "doesn't count as heir" worse, not better, since bastards are normally
  excluded from inheritance entirely.
- Considered vanilla's adoption mechanic
  (`common/character_interactions/00_adoption.txt`) as a way to establish
  the second parent cleanly. Ruled out: it's designed for a childless or
  low-fertility ruler adopting an *unrelated* heir, gated on traits like
  `compassionate` and `any_child = { count < 1 }` — the wrong tool
  entirely for co-parenting a newborn, and doesn't clearly establish real
  genealogical parentage either.
- Applied the one change that's genuinely low-risk regardless of the exact
  root cause: an explicit `set_mother = root` effect call
  (`common/effect_localization/00_character_effects.txt:2004`, confirmed
  real, distinct code path from the `mother =` parameter inside
  `create_character`) added immediately after character creation in both
  `mpreg.0007` branches. If the parameter form silently drops a male
  character but the separate effect form doesn't, this fixes it for free;
  if the parameter form already worked, this is a harmless no-op.

**Root cause still not confirmed — this is a reasoned experiment, not a
verified fix.** If the child still doesn't recognize the carrier as a
parent after this, the next diagnostic step is checking whether the
carrier appears in the child's family tree UI **at all**: completely
absent would mean the relationship is never established at the data level
(needs a different approach entirely, possibly requiring accepting this as
a genuine engine limitation); present but mislabeled would point back to a
narrower, display-only issue.

**Lesson 26 — the `set_mother` experiment failed too (2 for 2 on "mother
= male character"), so switched to `father` + `real_father` instead, per
the debugging discipline's own "2+ failures means stop and question the
approach" rule.** Live test: identical result to before — child recorded
the other father, not the carrier, still not eligible as heir. Two
independently-different code paths (the `mother =` create_character
parameter, and the standalone `set_mother` effect) both failing the exact
same way is strong evidence this isn't a code-path quirk — CK3 most
likely does not allow a male character to be recognized as "mother" at
all, the same category of hard restriction as `make_pregnant`'s
female-only requirement (Lesson 1), just failing silently instead of
throwing a script error.

Per the systematic-debugging skill's own rule (2+ failed attempts on the
same bug = stop and question the approach, don't just try a third
variant of the same idea) — paused and re-researched properties of
`father`/`real_father` before touching code again, rather than guessing a
third time. Found real, useful information that changes the earlier
Lesson 25 risk assessment: `real_father != father`
(`common/scripted_triggers/00_bastard_triggers.txt:76`) does NOT
automatically flag a child a bastard — reading the trigger's actual
context (`secret_disputed_heritage_is_valid_trigger`, lines 70-90) shows
it only gates whether an optional, discoverable "disputed heritage"
*secret* can exist, explicitly excluding characters who already have a
real `bastard`/`wild_oat`/`legitimized_bastard` **trait** — meaning
bastardy itself is a trait applied by vanilla's own scripted birth-flavor
events, not an automatic engine computation from the father/real_father
mismatch alone. Since this custom-created child never passes through
those vanilla events, that trait is very unlikely to ever get applied
here. Lesson 25's rejection of this approach was based on an overstated
fear.

What's still genuinely unconfirmed: whether `real_father` grants the
*other* father the same "real parent" recognition that `father` does, or
whether he simply inherits the identical problem the carrier has now.
Asked the user directly given this genuine uncertainty and the project's
track record this session (systematic-debugging: 2+ failures →
architectural question, not another guess) rather than picking silently;
they chose to proceed with `father = <carrier>` / `real_father = <other
father>`, explicitly prioritizing the carrier's own recognition (their
stated complaint) over solving both sides symmetrically in one pass.
**Superseded by Lesson 28 below** — `real_father` was later removed
entirely once a real overwrite risk in its actual vanilla usage was
found, so this specific question (whether it grants full recognition)
was never actually tested and is no longer relevant.
Implemented in `mpreg.0007`: both branches now use `father = root`
(dropping `mother =` and the `set_mother` experiment entirely),
`real_father = scope:mpreg_other_father` where he still exists,
`dynasty_house`/`faith`/`culture` now all derive from `root` (the
carrier) instead of the other father, matching the slot swap. (This
`real_father` usage was later removed — see Lesson 28.)

**Lesson 26 confirmed working**: `father = <carrier>` fixed the carrier's
own recognition — a live test showed the carrier correctly listed as a
parent and able to use the naming widget. The naming widget not appearing
(Lesson 23/25) really was downstream of the same parent-recognition
problem, exactly as suspected — no separate fix was needed for it once
the underlying relationship was established correctly.

**Lesson 27 — which father gets the confirmed-working "father" slot is
no longer hardcoded to "whoever carried."** Real user feedback: if the
*other* partner carries in some future pregnancy, the "main" character
(politically/narratively) should still be able to name the child and have
it count as their heir — carrying and inheritance-line standing are
unrelated concerns. Design decision: whichever of the two fathers holds
the higher-tier title gets `father` (the real inheritance slot); the other
gets `real_father`. Uses `highest_held_title_tier`, not `primary_title.tier`
— confirmed real and more robust (accounts for all titles held, not just
the primary one), and confirmed that comparing it directly against another
character's dynamic value (not just a fixed constant like `tier_county`)
is valid real syntax
(`common/scripted_triggers/00_secret_type_triggers.txt:250`:
`any_close_or_extended_family_member = { highest_held_title_tier >=
$OWNER$.primary_title.tier }`). Ties, including landless-vs-landless,
default to the carrier via `>=`. **Confirmed working in the one test run
so far — the carrier happened to also be the stronger-titled partner in
that test**, so the "carrier is NOT the more powerful partner" branch is
still specifically unconfirmed (see Section 5 TOP PRIORITY).

**Lesson 28 — `real_father` removed entirely; the weaker parent is now
simply not recorded in genealogy at all.** User asked directly, before
any live test of the "carrier is the weaker partner" branch could even
happen: if the disputed-heritage secret (enabled by exactly the
`real_father != father` pattern Lesson 27 would create) ever got
discovered, wouldn't the recorded father get overwritten? Checked the
actual vanilla secret's exposure effect rather than guessing —
`common/secret_types/00_bastard_secrets.txt:405-410`, inside the
exposure/`hidden_effect` block — and confirmed the fear was correct:
```
scope:child = {
    set_father = scope:real_father
}
```
This runs on discovery, which can happen at any point after birth,
possibly generations later, entirely outside this mod's control. It
would silently replace the correctly-assigned, title-tier-chosen
`father` with whoever was in `real_father` — undoing Lesson 27's whole
point and corrupting a real save's succession data with no warning.
Confirmed unacceptable.

Considered "make the other father a grandparent instead" (user's own
suggestion) — rejected: CK3 genealogy has no side-channel for this.
Making someone a grandparent of the child means editing one of the
child's *actual* parents' own recorded parent, which would corrupt a
completely different, unrelated relationship (that parent's real
mother/father) just to give the child a fabricated ancestor. Genealogy
only extends one hop per generation and only through real
mother/father/real_father fields — there's no way to attach a person
"sideways" without overwriting something that already means something
else.

Final fix: `create_character` in `mpreg.0007` now only ever sets
`father = scope:mpreg_stronger_parent` (Lesson 27's title-tier winner);
`real_father` is never set. The weaker of the two fathers is not
recorded as mother, father, or real_father — he has no genealogy link to
the child at all. He keeps his gameplay role throughout conception and
gestation (the `mp_carrier`/`mpreg_ss_other_father` mechanics are
unaffected), he's just invisible to the family-tree/inheritance system
for this specific child. A real, known consequence of this trade-off:
CK3's native `is_close_family_of` trigger (confirmed via `triggers.log`)
only recognizes parents/children/siblings/grandparents/grandchildren —
not spouses — so if the *carrier* ends up being the weaker-titled
parent, the carrier will have **no** recorded family link to their own
child either. Concretely: `mpreg.0008`'s naming widget (gated on
`allow_naming_on_birth_of_child_trigger`, which requires
`is_close_family_of = $CHILD$`) fires on `root` (the carrier)
unconditionally, so in that specific branch nobody will get the naming
prompt at all — the game will simply auto-name the child, the same
graceful fallback that happens whenever no eligible human parent is
present. This is a real, accepted gap, not a crash, and it's the direct
cost of closing the overwrite hole. Logged as a fresh open item in
Section 4 and the first item in Section 5's TOP PRIORITY test list — not
fixed further without the user weighing in on whether it's worth solving
(e.g. firing `mpreg.0008` on `scope:mpreg_stronger_parent` instead of
`root` would fix the naming prompt for that branch, at the cost of the
carrier no longer getting their own birth notification when they're the
weaker parent — a real trade-off, not an obviously-better default, so
left for a real decision rather than picked silently). A non-genealogy
custom relation (like vanilla's lover/rival relation, entirely separate
from mother/father/real_father) remains a floated, unimplemented future
idea for giving the weaker parent *some* in-game acknowledgment without
touching succession-critical fields at all.

## 4. Open items — not solved yet

**Carrier gets no naming prompt when the other father outranks them
(Lesson 28).** Since `real_father` was removed to close the discovery-
overwrite hole, the weaker of the two fathers has zero genealogy link to
the child. When that weaker parent is the *carrier* specifically, root
(who `mpreg.0008` always fires on) fails `is_close_family_of` and the
naming widget won't show — the child gets auto-named instead. Not a
crash, but a real gap: the person who physically gave birth can't name
their own child in that one branch. Possible fix (not implemented,
needs a decision, not picked silently): fire `mpreg.0008` on
`scope:mpreg_stronger_parent` instead of always on `root`, so whoever is
actually recorded as `father` gets the prompt — trade-off is the carrier
then wouldn't get their own birth notification in that branch either.
Needs a test where the carrier is the weaker-titled partner to actually
observe this in-game first (see Section 5 TOP PRIORITY item 1).

**Clothing clipping.** Two approaches attempted:
- `bs_cloak_offset` extension (already has a `male` setting in vanilla,
  low-risk to extend) — value tested at a conservative `0.5` add on top of
  vanilla's own baseline. Whether vanilla's baseline already saturates
  this at 1.0 (leaving no headroom) is untested.
- Forcing the leaf outfit (`mpreg_clothes_test.txt`) — the original
  approach (reopening vanilla's `clothes_special` group) had two real bugs,
  now fixed; see Lesson 11. Rewritten as its own `mpreg_clothes_special`
  group at `priority = 7` with `ignore_outfit_tags = yes`, matching a
  confirmed vanilla pattern for trait-forced clothing. Not yet re-tested
  in-game — this is now the top item in Section 5.
- For actual testing (not final gameplay), the game's own `portrait_editor`
  console command (requires `-debug_mode` launch flag) can force any
  clothing directly on a character, bypassing this whole question. Good
  for visual testing; not a real fix for normal play.
- The real, durable fix for close-fitting robes/tunics would be sculpting
  dedicated `bs_pregnant` blend shapes per garment, the way female
  garments already have them. Explicitly deferred — not attempting to
  reclothe the entire male wardrobe. A single dedicated "pregnancy robe"
  swapped in via decision was floated as a future-scope idea.

**Male-male conception and birth.** Implemented and confirmed working
end-to-end in a live game (v0.21/22 — birth actually produced a child).
See Lesson 14 for the full grounding, Lesson 16 for two real bugs found
and fixed from a live 10-year test. Known, deliberate simplifications: the
"both partners flagged carrier" tie-break is a plain higher-fertility
comparison rather than a true weighted roll (see Lesson 14's last
paragraph); a couple that is simultaneously married *and* has the lover
relation to each other would get two independent try-for-a-child chains
running at once (mildly more generous odds than intended, not a
correctness bug — not specially guarded against since it's a narrow edge
case); if the other father dies during gestation the pregnancy currently
stalls indefinitely rather than resolving (Lesson 16); and it's currently
unconfirmed whether the AI ever chooses same-sex marriage/lover relations
on its own (no naturally-occurring gay marriages observed in a 10-year
test) — unclear yet whether that's expected vanilla AI sexuality-weighting
behavior or something this mod should also address. Also unconfirmed:
whether the real per-month odds (as low as 1%) made the player's own
10-year, zero-success test statistically unsurprising (~30% chance of zero
successes at the floor rate over 120 months) or whether something is still
silently wrong — needs either a longer test, a temporarily-boosted test
value, or closer log inspection during an active attempt to distinguish
"unlucky" from "broken."

**Mixed-couple / male-male trait collision (Lesson 16).** A character who
is redirected from his wife's real pregnancy (`mpreg.0001`) and who is
*also* a carrier in an unrelated male-male pregnancy (`mpreg.0006`) shares
the exact same `mp_pregnant`/stage traits for both, because both systems
were built to reuse the same portrait-driving traits for simplicity. The
mutual-exclusion guards (`NOT has_trait mp_pregnant`) should prevent him
from being actively "claimed" by both at once, but a real test reported
his birth (from the unrelated relationship) appearing to affect his wife's
separate, real pregnancy. `mpreg_hide_female_pregnancy`'s suppression
trigger was tightened (now also requires her own `is_pregnant = yes`) but
this is a mitigation, not a structural fix — the actual fix would need the
redirect to carry an explicit "this belongs to *this* relationship"
marker instead of relying on the shared trait alone. Not designed yet;
needs the exact reproduction scenario confirmed first (see Section 5).

## 5. TODOs (next session)

### CONFIRMED (v0.31 test)

- **Force/Forbid Carrier interactions** (Lesson 22) — work as designed.
- Birth itself, and the visible birth notification, work.
- **`father = <carrier>` fixes the carrier's own recognition** (Lesson
  26): the carrier correctly showed as a parent and could use the naming
  widget. Confirms the naming-widget failure was downstream of the same
  root cause, not a separate bug.

### CONFIRMED FAILED (v0.30 test, do not retry these exact approaches)

- `mother = root` (create_character parameter) — child recorded the other
  father, not the carrier; carrier could not be designated heir.
- `set_mother = root` (separate effect, added on top of the above as an
  experiment) — identical failure. Do not attempt a third variant of
  "put a male character in the mother slot" — two independent code paths
  failing the same way is strong evidence this is a hard, non-scriptable
  restriction. See Lesson 26.

### TOP PRIORITY for next session

1. **Test the "carrier is NOT the more powerful partner" branch**
   (Lesson 27/28, v0.33) — the one successful parentage test so far
   happened to have the carrier as the stronger-titled parent too, so
   this specific branch has never been observed in-game. Expected,
   *not* a bug: the other (stronger) father should show as `father` and
   be the correct heir; the carrier should show as having **no**
   recorded parent relationship to the child at all (no mother, father,
   or real_father — that's the deliberate Lesson 28 trade-off, not a
   regression). Also specifically check whether the carrier gets the
   naming-widget prompt in this branch — per Lesson 28/Section 4, the
   expectation is **no**, the child should auto-name instead. If the
   carrier unexpectedly still gets prompted, or if anything crashes, that
   contradicts this session's analysis and needs to be re-investigated
   rather than assumed fine.
2. Re-confirm the naming field still works correctly in the **carrier-is-
   stronger** branch too (already confirmed once pre-Lesson-28, should be
   unaffected by removing `real_father`, but worth a quick re-check since
   the `create_character` call itself changed).
3. **Female-female marriage consequence** (Lesson 24) — explicitly
   low-priority per the user (2026-08-15: "dont care tho because thats not
   important to an mpreg mod at all"). Test eventually, not urgently.

### After that's confirmed

4. ~~Confirm AI now actually uses the carrier decisions~~ — **confirmed**
   (2026-08-15 test, 10 years: "lots of pregnancies" observed, AI carrier
   adoption "seems to work too"). Whether AI ever forms same-sex
   marriages/lover relations *on its own* is still separately unconfirmed
   (a different AI decision from becoming a carrier) and lower priority.
5. Decide on a real fix (not just the `is_pregnant = yes` mitigation) for
   the mixed-couple/male-male trait collision — Lesson 16's last section.
   Needs the exact reproduction scenario confirmed first.
6. Reload the existing pre-0.23 test save and check `error.log` to
   confirm `mpreg_trait_conversion.lookup` actually converts
   `mp_pregnant_stage_1/2/3` cleanly (mechanism confirmed real, this mod's
   specific use of it not yet live-tested).
7. **Brainstorm the flavor-content direction before implementing
   anything** — events, major decisions, nicknames, and "fanservice" are
   different scopes of work with different risk profiles (a nickname is
   cheap and low-risk; a full event chain is a real content commitment).
   Don't start writing content until this is actually decided with the
   user, per this project's own established process (clarifying questions
   → concrete approaches → approval → implementation, not guessing at
   scope). Per the user (2026-08-15): once the top-priority items above are
   confirmed working, the mod is considered close to release-ready, and
   this is the next phase.
8. ~~Write real Workshop-facing description/content-warning text~~ —
   **drafted**, `WORKSHOP_DESCRIPTION.md` (2026-08-15). Covers features,
   beta/known-issues status, the `blocks_achievements` flag, the
   male_body.asset conflict risk, content labeling, and the AI-code/
   handmade-assets disclaimer. Review before actually publishing — it was
   written once, not yet re-checked against whatever's shipped by then.
9. Lower priority, not blocking: clothing clipping (legwear stays on,
   Section 4); the "both flagged carrier" tie-break is a plain comparison
   not a true weighted roll (Lesson 14); `mpreg_hide_female_pregnancy`'s
   `-2.0` gene value logs a harmless but noisy "value clamped" warning on
   every portrait recompute (functionally fine, just log spam).
