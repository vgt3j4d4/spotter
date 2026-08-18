# AGENTS.md — `web/`

Angular 20, Angular Material, TypeScript. Read the [root AGENTS.md](../AGENTS.md) first.

> The application does not exist yet — it is scaffolded in
> [#2](https://github.com/vgt3j4d4/spotter/issues/2). These are the rules it is built to.

## Commands

```bash
npm start          # dev server on :4200, proxied to the API on :8080
npm test           # unit — Jest + Testing Library
npm run e2e        # Playwright
npm run lint -- --fix
```

## The user is standing in a gym

This is the constraint behind most of the UI rules, and it is not decoration.

The trainer is holding a phone in one hand, mid-session, possibly with a tape measure in
the other. Sometimes the gym has bad signal. Every screen is designed for that, not for the
desktop it is being developed on.

- **Mobile-first.** Write the small-screen layout first and widen it, never the reverse.
- **Large touch targets**, reachable one-handed. Material's default density is designed for
  a mouse; increase it.
- **Increment and decrement controls** on numeric fields. Typing 62.5 on a phone keyboard
  mid-set is not going to happen.
- **Prefill from last time.** Set logging and measurement entry both start from the
  previous values so the trainer edits a delta rather than entering numbers from scratch.
  This is the single most important usability decision in the app.
- **Autosave with a visible saved indicator.** Never a Save button the trainer has to
  remember before locking their phone.

## Structure

Standalone components, no NgModules. Feature-routed with lazy loading, so the athlete's
public view does not download the trainer's admin bundle.

```
src/app/
  core/         interceptors, guards, api client
  shared/       reusable components, pipes
  features/
    exercises/  athletes/  sessions/  measurements/  public/
```

`features/public/` is the athlete's read-only view. It must not import anything from a
trainer feature — that coupling is how a write control ends up on a read-only page.

## State

- Angular **signals** for component state.
- Services with `toSignal` for server data.
- **No NgRx.** This is CRUD; the ceremony would exceed the complexity. Do not introduce a
  state library.

## Forms

Reactive forms, **typed**. Validation mirrors the server's — and the server's is
authoritative. A client-side rule that the server does not also enforce is decoration.

## HTTP

Three interceptors in `core/`: one attaches the JWT, one turns RFC 9457 Problem Details
into user-facing messages, one adds the correlation ID.

Never call `HttpClient` directly from a component. Go through a service.

## Units

The API speaks canonical units — kilograms, centimetres. The trainer's metric/imperial
preference is applied **only** at the display edge, through the shared pipes.

- Convert on read for display, convert on write before it reaches the API.
- No component does unit arithmetic inline.
- **Every displayed value renders its unit label**, always, even where it feels repetitive.
  An unlabelled number is a bug report waiting to be filed.
- Round for display only. Never round before sending.

See [ADR 0004](../docs/decisions/0004-store-measurements-in-canonical-units.md).

## Accessibility

Angular Material gives you a lot of this free — do not undo it.

- Never remove a focus outline without replacing it with something visible.
- Every icon-only button gets an `aria-label`.
- Chart data is also available as a table. A line chart alone is not accessible, and the
  table is genuinely more useful for reading exact numbers anyway.
- Respect `prefers-reduced-motion`.

## Testing

See [`docs/testing.md`](../docs/testing.md).

- **Testing Library queries** — by role and label, never by CSS class. A test that breaks
  when a class is renamed is a test that gets deleted; a test that queries by role fails
  when the accessible name breaks, which is exactly when you want to know.
- `HttpTestingController` for HTTP. No real network in a unit test.
- Unit conversion is property-tested: convert, convert back, land where you started.
- Do not assert on DOM structure. Assert on what the user can see and do.
