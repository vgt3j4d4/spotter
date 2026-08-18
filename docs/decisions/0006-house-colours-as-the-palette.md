# 0006 — House colours as the palette

**Status:** Accepted · 2026-08-18

## Context

The application needed a visual identity. Four directions were designed and rendered on the
set-logging screen, in light and dark:

1. **Calibrated** — IWF plate colours as the semantic system, cast-iron neutrals, blue primary
2. **Sodium** — dark-first, sodium-amber accent, light as the variant
3. **Clinic** — light, deep teal, data-forward, Roboto Flex only
4. **House Colours** — the gym's own black and red

The gym this is built for already has an identity: black and red, on its signage and kit,
brand red `#E83643`.

## Decision

Use the gym's colours.

The trainer opens the app in his own gym and it looks like it belongs there rather than like
a generic tool he happens to run there. A client glancing at the phone recognises it. That is
a stronger reason than any palette invented from a general design vocabulary — including
Calibrated, which was the previous recommendation and was invented from the domain rather
than taken from the client.

Typography: Barlow Condensed for headings and exercise names, Barlow for body and numbers.

Theming: both themes ship, driven by `prefers-color-scheme`. **No manual toggle.**

Full token system in [`../design.md`](../design.md).

## The problem the decision created, and how it is resolved

`#E83643` measures **4.17:1 against white** — under the 4.5:1 WCAG AA floor for normal text.
It cannot carry a small white button label. On the dark surface it is 4.29:1, also short.

Two resolutions were considered.

**Rejected: one red everywhere, with every button label at 18.66 px bold** so it qualifies as
large text and only needs 3:1. It works, and it constrains every button in the application
forever in order to protect one hex value.

**Accepted: the brand value stays the brand value, and two derived tokens do the jobs it
cannot** — `#D8232F` (5.00:1) for filled buttons and red text on light grounds, `#F0525C`
(5.17:1) for everything red in dark mode. All three read as the same red at a glance.

`#F0525C` also addresses halation: saturated red on near-black shimmers at the edges, badly
for anyone with astigmatism. The ground is `#111114` rather than `#000000` for the same
reason.

**Red also collides with error states.** Since red is the brand, it cannot also be the sole
signal for failure. The distinction is carried by **shape, not hue**: brand red is always a
filled shape, an error is always a tinted surface with a left border, an icon and text. This
is independently required by WCAG 1.4.1, which says colour alone can never carry meaning.

## Alternatives rejected

**Calibrated (plate colours).** Genuinely good and grounded in the domain — a 20 kg chip in
plate blue is the same signal the bar gives. Rejected because it was invented rather than
taken from the actual client, and House Colours does the same job with a real provenance.

**Sodium (dark-first amber).** The best pure-usability argument: sessions happen early and
late, in dim rooms, and a white screen at 6 a.m. is hostile. Rejected because the gym's
identity outweighs it, and because system-driven dark mode captures most of the benefit.

**Clinic (light teal).** The safest, and safe reads as generic. It would have flattered the
measurement charts most.

**A manual theme toggle.** Deferred, not rejected on principle. It is a real feature —
persistence, a settings surface, and a flash-of-wrong-theme problem on load — and it can be
added later as a `data-theme` attribute over the same tokens.

## Consequences

**Good.** The app looks like it belongs to the gym that uses it. The contrast constraint was
measured rather than assumed, and the resolution is documented, which makes both the palette
and the accessibility work defensible rather than accidental.

**Bad.** The identity is borrowed. If the gym rebrands, or this is ever shown to a second
gym, the colours belong to someone else. Everything is tokens, so that is an afternoon — but
it is a real cost and it should be known now rather than discovered later.

**Also bad.** Three red tokens is more to remember than one, and the wrong one will get used.
The mitigation is that `--red-brand` is documented as never carrying small text, and the
accessibility audit ([#37](https://github.com/vgt3j4d4/spotter/issues/37)) checks it.
