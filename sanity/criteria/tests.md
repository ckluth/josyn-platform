# Sanity Criteria — tests

> Verify unit test existence, quality, and passing state.
> Framework: NUnit 4.x, .NET 10.

---

## Coverage

- Every code path where undetected misbehaviour is realistically possible has a test.
- Required: positive cases, negative cases, edge cases, and boundary values.
- Tests verify meaningful behaviour — not line coverage for its own sake.

---

## Naming

**Test methods:** `SubjectUnderTest_Scenario_ExpectedResult`

- Operators or constructors: use a descriptive subject — e.g. `ImplicitConversion_FromException_SetsCallerInfo`
- Properties: e.g. `CallStackAsString_OnSuccess_ReturnsNoCallstackMessage`

**Test classes and files:** `<SubjectUnderTest>Tests.cs`
Split into multiple files when a subject has clearly distinct logical concerns.

---

## Structure

- **Arrange-Act-Assert** — three distinct phases, separated by a blank line unless trivially a single statement.
- **Fluent assertions:** `Assert.That(x, Is.EqualTo(y))` — not classic `Assert.AreEqual`.
- **One observable outcome per test.** Multiple `Assert.That` calls are allowed when they jointly verify a single outcome. If a failure on one makes the others redundant to report, they belong together. If not, split the test.
- **No flow control** — no loops, conditionals, or branching inside a test method. Local helper functions and static factory methods are fine; they must not alter which assertions execute.
- **Tests are independent** — no shared mutable state, no reliance on execution order. `[SetUp]` is acceptable for shared, truly read-only construction. Never use it to share state that individual tests mutate.
- **`[TestCase]`** — use when the same behaviour must hold across multiple input/output pairs and the test name + parameter value together are self-explanatory. Prefer it over copy-pasted test methods.

---

## Exception Testing

- Use `Assert.Throws<TException>(() => ...)` or the `Does.Throw` constraint.
- Do not use `[ExpectedException]`.

---

## Determinism

- Tests produce the same result on every run regardless of: execution order, time, machine, randomness.
- Tests that touch the file system, clock, or environment are integration tests — they must be in a separate assembly or marked explicitly. They do not count as unit tests.

---

## Skipped Tests

- No committed `[Ignore]`-decorated tests without a reason string.
- A permanently ignored test is a violation — delete it or fix it.

---

## Test Doubles

| Double | When to use |
|--------|-------------|
| Stub | Control a dependency's output (canned return value) |
| Mock | Only when the *call itself* — not its side effect — is the behaviour under test |
| Fake | Stateful dependencies (repositories, in-memory queues) |

- Avoid over-mocking: if every collaborator is replaced, you are testing wiring, not behaviour.
- Reach for stubs before mocks.

---

## Quality Checklist

| Signal | Verdict |
|--------|---------|
| Public method with no corresponding test | ❌ violation |
| Test with flow control (if/loop) | ❌ violation |
| Test that touches clock, filesystem, or network without isolation | ❌ violation |
| `[Ignore]` without a reason string | ❌ violation |
| Test name does not follow `Subject_Scenario_Expected` pattern | ❌ violation |
| Committed test that fails | ❌ violation |
| Over-mocked test (every collaborator replaced) | ❌ violation |
| `[ExpectedException]` attribute used | ❌ violation |
| All tests pass, deterministic, one outcome each | ✅ pass |
