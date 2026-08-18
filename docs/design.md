# Design

## Where the palette comes from

The gym's own colours. Black and red, taken off its signage and kit, on the real brand
value **`#E83643`**.

This is not decoration. The trainer opens the app in his own gym and it looks like it
belongs there rather than like a generic tool he happens to run. A client glancing at the
phone recognises it. No invented palette gets that.

Three other directions were designed and rejected — the record is in
[ADR 0006](decisions/0006-house-colours-as-the-palette.md).

## The red is three tokens

`#E83643` measures **4.17:1 against white**. WCAG AA requires 4.5:1 for normal text, so the
brand value cannot carry a small white button label. It is 4.29:1 on the dark surface,
also short.

Rather than change the gym's colour, the brand value stays the brand value and two derived
tokens do the jobs it cannot:

| Token | Value | Contrast | Job |
|---|---|---|---|
| `--red-brand` | `#E83643` | 4.17:1 on white | Identity. Large fills, chips with dark text, the app icon, focus rings. **Never small text.** |
| `--red-action` | `#D8232F` | 5.00:1 on white | Filled buttons carrying white labels, links, red text on light grounds |
| `--red-on-dark` | `#F0525C` | 5.17:1 on `#17171A` | Everything red in dark mode |

All three read as the same red at a glance.

`--red-on-dark` also solves a second problem: saturated red on near-black shimmers at the
edges — badly for anyone with astigmatism. The lifted value and a `#111114` ground rather
than `#000000` both reduce it. Pure black maximises the halation and reads cheaper anyway.

**The alternative was one red everywhere**, with every button label set at 18.66 px bold so
it counts as large text and only needs 3:1. Rejected — it constrains every button in the
app forever to protect one hex value.

## Tokens

Light is the base. Dark redefines only the values.

```css
:root {
  /* ground and surface */
  --ground:        #FAFAFA;
  --surface:       #FFFFFF;
  --surface-sunk:  #F1F1F2;

  /* ink */
  --ink:           #111114;   /* 18.85:1 on surface */
  --ink-soft:      #3F434A;
  --ink-muted:     #6B7280;   /*  4.83:1 on surface */

  /* rules */
  --rule:          #E4E4E7;
  --rule-strong:   #C9C9CE;

  /* brand */
  --red-brand:     #E83643;
  --red-action:    #D8232F;
  --red-tint:      #FDE8EA;
  --red-ink:       #8A1018;   /* red text on --red-tint */

  /* semantic — never red, red is spoken for */
  --success:       #2F9E5E;
  --success-tint:  #E6F4EC;
  --warning:       #B26A00;
  --warning-tint:  #FBF0DC;
}

@media (prefers-color-scheme: dark) {
  :root {
    --ground:        #111114;
    --surface:       #17171A;
    --surface-sunk:  #212127;

    --ink:           #EDEDEF;   /* 15.30:1 on surface */
    --ink-soft:      #C2C2C8;
    --ink-muted:     #8A8F98;   /*  5.51:1 on surface */

    --rule:          #292930;
    --rule-strong:   #3A3A43;

    --red-brand:     #F0525C;
    --red-action:    #F0525C;
    --red-tint:      #2B1719;
    --red-ink:       #F0525C;

    --success:       #4FBF80;
    --success-tint:  #14251C;
    --warning:       #D9992E;
    --warning-tint:  #251C0F;
  }
}
```

**Every colour is a token on the bare `:root`.** A value whose only definition lives inside
the media query never applies where the query does not match, which is the classic
unreadable-theme bug.

## Theming

**System-driven. No toggle.**

The app follows `prefers-color-scheme` and there is no theme picker in the UI. Both themes
are designed and both ship; the trainer does not choose between them, the phone does.

A manual override is a real feature — persistence, a settings surface, a flash-of-wrong-theme
problem on load — and it is deferred, not forgotten. Because everything is tokens, adding it
later is a `data-theme` attribute and a service, not a redesign.

## Red is 5% of the screen

The version of black-and-red that looks like an energy drink uses red for large fills,
gradients and glows. This one does not.

The working interface is black, white and grey. Red appears only where an action or a state
lives: the primary button, the active nav item, a positive delta, a focus ring. Restraint is
the entire difference between a gym's brand and a gym's cliché.

If a screen has more than one red element competing for attention, one of them is wrong.

## Errors cannot be red-only

Red is the brand, which takes it away from error states. Hunting for a second red does not
work — under gym lighting nobody distinguishes two reds.

**The distinction is shape, not hue.**

| | Looks like |
|---|---|
| Brand / action | A **filled** shape — solid `--red-action`, white label |
| Error | A **tinted surface** with a left border, an icon, and a sentence of text |

An error is never a filled red button. The two never read as the same thing even though the
hue is related.

This is also required independently of branding: WCAG 1.4.1 says colour alone can never
carry meaning. The icon and the text were needed regardless.

## Type

| Role | Face | Notes |
|---|---|---|
| Headings, exercise names | **Barlow Condensed** 600/700 | Condensed buys horizontal room on a phone, and reads athletic without trying |
| Body, labels, numbers | **Barlow** 400/500/600 | Real tabular figures — essential where digits change in place |

Both from Google Fonts, one family, two widths. Fallback stack:
`"Barlow", system-ui, -apple-system, sans-serif`.

**`font-variant-numeric: tabular-nums` on every number the trainer reads.** Weights, reps,
measurements, dates. Proportional figures shift the layout as a value changes, which is
exactly the moment the trainer is looking at it.

## Hierarchy: the weight is the biggest thing

On the set-logging card the number being read mid-set is the largest element. Athlete name,
date and exercise are all secondary to it.

This holds on every screen: the value the user came for outranks the context around it.

## Density and touch

The trainer is holding a phone one-handed, mid-session, sometimes with a tape measure in the
other hand.

- **48 × 48 px minimum** for anything tappable. Increment and decrement controls on numeric
  fields get more.
- **Angular Material's default density is designed for a mouse.** It is increased across the
  board — do not accept the defaults.
- Primary actions sit within one-handed thumb reach, at the bottom of the screen, not the
  top.
- Nothing important lives behind a hover state. There is no hover on a phone.

## Angular Material

The tokens above drive an M3 theme. Material's `primary` maps to `--red-action` — not
`--red-brand` — because Material puts white text on `primary` by default and the brand value
does not have the contrast for it.

`error` gets its own role and is styled as a tinted surface, not a filled button, per the
rule above.

Do not override Material's focus indicators without replacing them with something at least
as visible. Angular Material's accessibility defaults are good; the failure mode here is
undoing them for looks.
