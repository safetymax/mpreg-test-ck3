# MPREG - Male Pregnancy for Crusader Kings III

As far as we know, this is the first working male pregnancy mod for CK3. Men can be flagged as carriers, conceive, visibly grow, and give birth, with real vanilla same-sex marriage support built in. It's a first release and still actively being worked on, so there are some known issues. Please read through this before installing.

Current version: **1.0.1**

## What this actually does

This mod lets a male character carry and give birth to a child. Not as a reskinned decision, but wired into CK3's real systems as closely as vanilla allows:

- **Same-sex marriage is on by default**, for every faith, including ones that would normally forbid it. Faiths that consider it shunned or illegal don't block the marriage outright, but they do apply a real consequence through vanilla's own secret/blackmail system.
- **A mixed-sex couple can redirect the pregnancy onto the husband.** If he's opted in as a "Carrier," a real, mechanically normal pregnancy on his wife visibly plays out on him instead, belly, stages, and all, while the actual conception, health risk, and birth stay 100% vanilla and unmodified.
- **Two men can conceive, get pregnant, and give birth to a real child**, using a custom conception system built to mirror vanilla's real fertility math (age, health, and traits all matter, not a flat coin flip). The child is correctly recorded as heir to whichever of the two fathers holds the higher-tier title, regardless of who physically carried.
- **AI uses all of this too.** AI characters can become carriers on their own (more likely when younger), and the whole system runs in the background without any player input required.
- **Force or Forbid another character from being a carrier**, a first, simple version of an interaction for a liege/vassal or family authority over this. A fuller version, using hooks, gold, or influence to persuade rather than just order, is planned.

## Installation

1. Download or clone this repo into your CK3 mod folder: `Documents/Paradox Interactive/Crusader Kings III/mod/` (or the equivalent Steam Deck/Flatpak path, `~/.var/app/com.valvesoftware.Steam/.local/share/Paradox Interactive/Crusader Kings III/mod/`).
2. Make sure a `.mod` descriptor file in the parent `mod/` folder points at this directory's `descriptor.mod` (see `descriptor.mod` in this repo for the `path=` and `version=` to match).
3. Enable it in the CK3 launcher's playset like any other mod.
4. Enabling this mod turns on the vanilla `accepted_same_sex_marriage` game rule by default, see Compatibility notes below.

## Known issues, please read before installing

This is a first release, not a finished, feature-complete mod. A few things to know going in:

- **Clothing doesn't fully adjust.** A pregnant male character is forced into a state resembling vanilla's own "no clothes" look, but legwear currently stays on. There's no dedicated maternity clothing yet. That's a known, deliberate trade-off for now, not an oversight.
- **This is actively being developed and will have bugs.** Several real ones have already been found and fixed by testing in a live game, and more are likely, especially around edge cases (a character in two overlapping relationships, a parent dying mid-pregnancy, and similar). If something looks broken, please open an issue.
- **When two fathers are involved, only the higher-titled one is recorded in genealogy.** This is a deliberate trade-off to avoid a real data-corruption risk in vanilla's own "disputed heritage" secret. The weaker parent keeps his gameplay role but isn't a recorded parent of the child. One known consequence: if the carrier ends up being the weaker-titled parent, he currently doesn't get the birth-naming prompt.
- **Save compatibility across updates isn't guaranteed** during the beta. There's a conversion mechanism for known cases, but you should expect some rough edges if you update a mod version mid-save.

## Compatibility notes

- Enabling this mod turns on the vanilla `accepted_same_sex_marriage` game rule by default, which **disables achievements** for that game rule setting, whether you use the pregnancy features or not.
- This mod fully overrides vanilla's male body asset file to add the pregnant belly. **It will conflict with any other mod that also modifies the male body model.**
- Contains a nudity-adjacent outfit for pregnant characters (vanilla's own existing "fig leaf" content, not new art), just be aware if that matters to you or the people you play with.

## Repo structure

```
common/                  Traits, decisions, interactions, game rules,
                          scripted triggers, script values, on_actions,
                          genes. The non-visual game logic.
events/                  mpreg_events.txt, the conception/gestation/
                          birth event chain.
gfx/                     Portrait modifiers, genes-driven visuals, and
                          the custom pregnant male body model/blendshape.
localization/            English text.
descriptor.mod           Mod metadata (version, supported game version).
README.md                This file, player-facing overview.
```

Every script file has a header comment explaining what it does and why, including the reasoning behind less obvious choices, so the code should be reasonably readable on its own.

## Contributing / reporting bugs

Bug reports and feedback are genuinely welcome. Please open an issue with what you saw, your character's situation (same-sex marriage vs. mixed-couple redirect vs. male-male), and your game version. If you're looking to contribute code, read through the header comments in the relevant script files first. Several things that look like bugs are actually confirmed, hard CK3 engine limits (a male character cannot be recognized as "mother," for example), and those are called out where they matter.

## Credits / disclaimer

The mod's script/code was written with AI assistance (Claude). **All custom art assets, the belly sculpt/mesh and any other custom models, are handmade**, not AI-generated.

This is a passion project made because, as far as we're aware, no other CK3 mod has gotten male pregnancy actually working end to end: real conception, real stages, a real birth with a real child. Feedback and bug reports are genuinely welcome. This is early, and there's a lot more planned (flavor events, decisions, and more).
