# Steam Workshop Description — Draft (v0.28)

Draft text for the Workshop page. Written to be pasted in and lightly
reformatted for Steam's BBCode (bold/headers/bullets translate directly;
adjust as needed). Update the version number and the "known issues" list
whenever a new build actually changes what's in this document — don't let
it drift out of sync with README.md the way earlier in-code comments did.

---

## Title

Male Pregnancy (MPreg) — Beta

## Short description (workshop card)

The first working male pregnancy mod for CK3. Men can be flagged as
carriers, conceive, visibly grow, and give birth — with real vanilla
same-sex marriage support built in. Early beta, actively developed, has
known issues — read before installing.

## Full description

### What this actually does

This mod lets a male character carry and give birth to a child — not as a
reskinned decision, but wired into CK3's real systems as closely as
vanilla allows:

- **Same-sex marriage is on by default** for every faith, including ones
  that would normally forbid it. Faiths that consider it shunned or
  illegal don't block the marriage outright, but do apply a real
  consequence through vanilla's own secret/blackmail system.
- **A mixed-sex couple can redirect the pregnancy onto the husband.** If
  he's opted in as a "Carrier," a real, mechanically normal pregnancy on
  his wife visibly plays out on him instead — belly, stages, and all —
  while the actual conception, health risk, and birth stay 100% vanilla
  and unmodified.
- **Two men can conceive, get pregnant, and give birth to a real child**
  with both of them recorded as parents, using a custom conception system
  built to mirror vanilla's real fertility math (age, health, and traits
  all matter, not a flat coin flip).
- **AI uses all of this too** — AI characters can become carriers on
  their own (more likely when younger), and the whole system runs in the
  background without player input required.
- **Force or Forbid another character from being a carrier** — a first,
  simple version of an interaction for liege/vassal or family authority
  over this; a fuller version (using hooks, gold, or influence to persuade
  rather than just order) is planned.

### Beta status — please read before installing

This is an early beta, not a finished mod. Specifically:

- **Clothing doesn't fully adjust.** A pregnant male character is forced
  into a state resembling vanilla's own "no clothes" look, but legwear
  currently stays on. There is no dedicated maternity clothing yet — this
  is a known, deliberate trade-off for now, not an oversight.
- **This is actively being developed and will have bugs.** Several real
  ones have already been found and fixed by testing in a live game; more
  are likely, especially around edge cases (a character in two
  overlapping relationships, a parent dying mid-pregnancy, and similar).
  If something looks broken, it probably is — please report it.
- **Newborn parentage display is under investigation.** The underlying
  data may be correct while the family-tree UI — which has never had to
  display two male parents before, since vanilla itself can't produce
  that combination — may not show it properly. Being actively looked into.
- **Save compatibility across updates is not guaranteed** during the beta.
  A conversion mechanism exists for known cases, but you should expect
  some rough edges if you update a mod version mid-save.

### Compatibility notes

- Enabling this mod turns on the vanilla `accepted_same_sex_marriage` game
  rule by default, which **disables achievements** for that game rule
  setting whether you use the pregnancy features or not.
- This mod fully overrides vanilla's male body asset file to add the
  pregnant belly. **It will conflict with any other mod that also
  modifies the male body model.**
- Contains a nudity-adjacent outfit for pregnant characters (vanilla's own
  existing "fig leaf" content, not new art) — be aware if that matters to
  you or the people you play with.

### Credits / disclaimer

The mod's script/code was written with AI assistance (Claude). **All
custom art assets — the belly sculpt/mesh and any other custom models —
are handmade**, not AI-generated.

This is a passion project made because, as far as we're aware, no other
CK3 mod has gotten male pregnancy actually working end-to-end — real
conception, real stages, a real birth with a real child. Feedback and bug
reports are genuinely welcome; this is early and there's a lot more
planned (flavor events, decisions, and more).
