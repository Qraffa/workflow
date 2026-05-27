# Test Rules — The single source of truth

**Status: MANDATORY READ before writing or modifying any test in a
SliceSpec slice.** Both the implementer (`/slicespec-implement`,
step 2 of the TDD cycle) and the code-quality reviewer
(`/slicespec-review`) operate from this file. It is the only place
test / mock / anti-pattern rules are defined. Anywhere else that
mentions them must point here, not restate them.

Length budget: one screen per section. If you find yourself
expanding a section to a full page, the rule is probably wrong.

---

## §1 The single standard

A good test verifies **behaviour through a public interface**. It
describes *what* the system does, not *how* it does it. It would
still pass if the internals were rewritten in a different style
and still fail if a user-visible behaviour broke.

If renaming a private helper or restructuring an internal module
breaks a test without changing observable behaviour, that test is
wrong.

---

## §2 Good test — the shape

```ts
// GOOD: observable behaviour through the public API
test("user can checkout with valid cart", async () => {
  const cart = createCart();
  cart.add(product);
  const result = await checkout(cart, paymentMethod);
  expect(result.status).toBe("confirmed");
});
```

Properties to copy:

- Calls only the public surface (`createCart`, `checkout`).
- Asserts on the observable outcome (`result.status`).
- One logical assertion.
- Survives any pure refactor of `checkout`'s internals.
- Test name describes the capability ("user can checkout with
  valid cart"), not the mechanism.

---

## §3 Bad test taxonomy — four shapes to refuse

Each bad shape below has a typical severity if it slips into the
codebase (used by the quality review). The implementer's job is
**not to produce them at all**.

### §3.a Asserting on internal calls — Warning (Critical if sole assertion)

```ts
// BAD
expect(mockPayment.process).toHaveBeenCalledWith(cart.total);
```

Renaming `process` to `charge` breaks the test for no behavioural
reason. The test is coupled to the wiring.

Correct form: assert on what the user (or the caller) observes —
the order state, the receipt, the returned object.

### §3.b Bypassing the interface to verify — Warning

```ts
// BAD: bypasses the public reader
await createUser({ name: "Alice" });
const row = await db.query("SELECT * FROM users WHERE name = ?", ["Alice"]);
expect(row).toBeDefined();
```

This test will break when the schema changes even if `createUser`
still works. Worse, it locks in today's persistence model.

Correct form:

```ts
const u = await createUser({ name: "Alice" });
const retrieved = await getUser(u.id);
expect(retrieved.name).toBe("Alice");
```

### §3.c Mocking the system under test — **Critical**

If the test mocks the very function or class it claims to verify,
it proves nothing. Common smells:

```ts
const sut = jest.mock("./checkout");      // wrong
sut.checkout.mockReturnValue({status: "confirmed"});
expect(checkout(...)).toEqual({status: "confirmed"});  // tautology
```

Delete the test and rewrite against the real implementation.

### §3.d Mocking code you own — Warning

Mocking internal collaborators (a service the team owns, a
database wrapper the team wrote, a helper from the same package)
to make the test "simpler" creates a test that verifies *today's
wiring* instead of behaviour. Any refactor that moves the
collaboration around breaks the test even though behaviour is
fine.

Correct form: use real collaborators inside the test boundary.
Mock only at the boundaries listed in §4.

---

## §4 Mock boundary — the only places mocks are licensed

You may mock at these boundaries and **only** these:

| Boundary | Why mock | Notes |
|---|---|---|
| External APIs (third-party services) | network, billing, flakiness | prefer a fake SDK over raw HTTP stubs |
| Databases you don't own | slow, shared state | use a real test DB if at all feasible — that's still preferred |
| Time / `Date.now` / timers | non-determinism | use a clock interface or library helper |
| Randomness | non-determinism | inject the RNG |
| Filesystem (sometimes) | slow, side-effecty | use a tmp dir before reaching for a mock |

You may **not** mock:

- Code your team owns.
- Internal collaborators (helpers, internal services, other
  modules in the same package).
- The system under test itself (§3.c).

### Design for mockability at the boundary

When you must mock, design the boundary to be mockable:

```ts
// EASY to mock: dependency is injected
function processPayment(order, paymentClient) {
  return paymentClient.charge(order.total);
}

// HARD to mock: dependency is constructed inside
function processPayment(order) {
  const client = new StripeClient(process.env.STRIPE_KEY);
  return client.charge(order.total);
}
```

Prefer SDK-style interfaces (one function per external operation)
over a generic `fetch(endpoint, options)`. SDK-style mocks return
one specific shape per call and don't need conditional logic
inside the mock.

---

## §5 Anti-pattern: horizontal slicing — Critical when found

**Do NOT** write all the Scenario tests first ("RED = all tests")
and then implement all of them ("GREEN = all the code").

That shape produces tests of *imagined* behaviour and the *shape*
of data, not what callers actually need. Such tests pass when
behaviour breaks and break when behaviour is fine.

The correct shape is **vertical** — one test, one impl, one
commit, then the next:

```
WRONG (horizontal):           RIGHT (vertical):
  RED:   t1, t2, t3, t4, t5     RED→GREEN: t1 → impl1
  GREEN: i1, i2, i3, i4, i5     RED→GREEN: t2 → impl2
                                RED→GREEN: t3 → impl3
                                ...
```

Each cycle is informed by what the previous cycle taught you about
the design. A slice that `covers` five Scenarios runs as five
small cycles, not one big batch.

The quality reviewer detects horizontal slicing from the commit
log (tests-only commits followed by impl-only commits with no
interleaving). The implementer's job is to not produce that
pattern in the first place.

---

## §6 Refactor candidates (after GREEN)

After each cycle reaches green, scan for these and address them in
the same commit (or a follow-up refactor commit, still green):

- **Duplication** → extract function / class.
- **Long methods** → break into private helpers; keep tests on
  the public interface only.
- **Shallow modules** → combine or deepen. Aim for *small
  interface + thick implementation*. A shallow module is a thin
  wrapper that just passes arguments through.
- **Feature envy** → move logic to where the data lives.
- **Primitive obsession** → introduce a value object.
- **Existing code the new code revealed as problematic** —
  flag in commit message; do not silently fix outside `write_scope`.

Never refactor while RED. The green bar is what tells you the
change was safe.

---

## §7 Naming — WHAT, not HOW

Test and function names should describe the observable behaviour
or capability, not the mechanism.

| Bad (HOW) | Good (WHAT) |
|---|---|
| `test("checkout calls paymentService.process")` | `test("user can checkout with valid cart")` |
| `getUserByDbQuery(...)` | `getUser(...)` |
| `handlerThatLoopsUntilDone(...)` | `processUntilStable(...)` |

If renaming an internal helper would force renaming a test, the
test name is HOW. Rename it.

---

## §8 Pre-flight checklist — run before EACH test you write

Five-question gate. If any answer is "no" or "I don't know",
stop and reshape the test before running it.

1. Does it test behaviour through the public interface? (§1, §2)
2. Is it free of internal mocks / private inspection / raw DB
   verification? (§3.a, §3.b)
3. Are mocks (if any) only at the §4 boundaries?
4. Is the test name about WHAT, not HOW? (§7)
5. Would it still pass if the internals were rewritten and still
   fail if a user-visible behaviour broke? (§1)

If all five pass, run the test (it must fail for the right
reason), then write the minimum code to make it pass, then
refactor per §6, then commit.

---

## §9 What the reviewer adds on top

The code-quality review (`/slicespec-review`, whose audit detail
lives in `../slicespec-review/reviewer-prompt.md`) layers on checks
that need post-hoc evidence the implementer cannot fully
self-judge:

- Commit-history detection of horizontal slicing.
- Commit-content detection of refactor-while-RED.
- Dead test detection (test passes without exercising new code).
- "Rationalisation" detection (tests-after, manual-only, etc.).
- Severity assignment and accepted-with-risk record-keeping.

Those checks exist as a safety net, not as the first place these
rules are learned. **The implementer is expected to ship code
that does not trigger them.** Reviewer findings beyond Info are a
signal the pre-flight checklist (§8) was skipped.
