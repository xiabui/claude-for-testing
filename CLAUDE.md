# CLAUDE.md — C# Playwright Test Automation

This file governs how Claude writes, edits, and reviews end-to-end UI tests in this repository. These rules are **mandatory**, not suggestions. When a rule conflicts with a quick solution, the rule wins.

> **GLOBAL RULE — ENGLISH ONLY:** All code, identifiers, test names, file/folder names, comments, commit messages, and documentation MUST be written in **English**. No Vietnamese (or any non-English) in code or comments under any circumstance. This applies even when the conversation with the user is in another language.

---

## 1. Tech Stack (pinned)

| Concern | Choice | Notes |
|---|---|---|
| Language | C# (latest LTS, `net8.0`+) | `<Nullable>enable</Nullable>`, `<LangVersion>latest</LangVersion>` |
| Browser automation | `Microsoft.Playwright` | Pin exact version in `.csproj` |
| Test runner | `Microsoft.Playwright.NUnit` (NUnit) | Use `Microsoft.Playwright.MSTest` **only** if the repo already standardizes on MSTest |
| Assertions | `Microsoft.Playwright.Assertions` (`Expect`) | **Never** mix with `Assert.That` for UI state |
| Config | `appsettings.json` + environment overrides | No hardcoded URLs/credentials |
| CI | Headless, sharded | `--workers` tuned per runner |

Do not introduce Selenium, WebDriver, or any other browser tool. Do not add a second assertion library.

---

## 2. Project Structure (flat-by-feature)

```
tests/
  Pages/                 # Page Objects, one class per page/component
    LoginPage.cs
    DashboardPage.cs
  Components/            # Reusable widget objects (nav, modal, table)
  Fixtures/              # Base classes, custom PageTest, data builders
    BaseTest.cs
  Tests/                 # Test classes grouped by feature
    Auth/
      LoginTests.cs
    Dashboard/
      DashboardTests.cs
  TestData/              # Static data, builders, factories
  Config/                # TestSettings, env loading
appsettings.json
appsettings.ci.json
playwright.config.cs (or runsettings)
```

One feature per folder. One page object per file. Test class name mirrors the feature, not the page.

---

## 3. Locator Rules (STRICT — most violations happen here)

**Priority order — always use the highest available:**

1. `page.GetByRole(...)` — preferred, mirrors accessibility tree
2. `page.GetByLabel(...)` — form fields
3. `page.GetByPlaceholder(...)`
4. `page.GetByText(...)` — for non-interactive content
5. `page.GetByTestId(...)` — when semantic locators are impossible

**Prohibited locators:**

- ❌ XPath (`//div[...]`) — banned outright
- ❌ CSS selectors tied to styling (`.btn-primary`, `.col-md-6`)
- ❌ Auto-generated/hashed classes (`.css-1a2b3c`, `.MuiButton-root`)
- ❌ Index-based chaining as a primary strategy (`.Nth(3)`) unless the list is inherently ordered and asserted
- ❌ Locating by raw `id` when a role/label exists

If a semantic locator is impossible, add `data-testid` to the **application** code and use `GetByTestId`. Configure the test-id attribute explicitly:

```csharp
playwright.Selectors.SetTestIdAttribute("data-testid");
```

Locators are lazy. **Declare them as fields/properties on the Page Object; never call `WaitFor` then act.** Use auto-waiting actions and web-first assertions instead.

---

## 4. Waiting & Synchronization (NO flakiness allowed)

**Absolutely prohibited:**

- ❌ `Thread.Sleep(...)`
- ❌ `Task.Delay(...)` as a synchronization mechanism
- ❌ `page.WaitForTimeout(...)`
- ❌ Polling loops written by hand
- ❌ `WaitForSelector` when a web-first assertion would do

**Required approach — rely on auto-waiting + web-first assertions:**

```csharp
// GOOD — retries until condition met or timeout
await Expect(page.GetByRole(AriaRole.Heading, new() { Name = "Dashboard" }))
    .ToBeVisibleAsync();

await Expect(page.GetByTestId("balance")).ToHaveTextAsync("$1,200.00");
```

For navigation/network, use explicit waits scoped to the action:

```csharp
await page.RunAndWaitForResponseAsync(
    () => page.GetByRole(AriaRole.Button, new() { Name = "Save" }).ClickAsync(),
    resp => resp.Url.Contains("/api/save") && resp.Ok);
```

Every assertion about UI state must be a web-first assertion (`Expect(...)`), which retries automatically. A bare `Assert.IsTrue(await locator.IsVisibleAsync())` is **forbidden** — it does not retry and causes flakiness.

---

## 5. Assertions

- Use `Expect(locator)` / `Expect(page)` web-first assertions for all UI/DOM state.
- Use NUnit `Assert` **only** for pure non-UI logic (computed values, API response bodies parsed to objects).
- One logical assertion focus per test where practical; multiple related `Expect` calls are fine.
- Always assert on **observable user-facing state**, not implementation details.
- Set assertion timeout globally; do not sprinkle per-call timeouts unless justified by a comment.

---

## 6. Page Object Model (enforced)

- Each Page Object receives `IPage` via constructor. No static state.
- Expose **actions** (`LoginAsync(user, pass)`) and **locators** (read-only), never raw selectors to tests.
- Page Objects contain **no assertions** except trivial readiness checks; assertions live in tests.
- Return the next Page Object from navigation actions to enable fluent flows.

```csharp
public sealed class LoginPage(IPage page)
{
    private ILocator Email => page.GetByLabel("Email");
    private ILocator Password => page.GetByLabel("Password");
    private ILocator Submit => page.GetByRole(AriaRole.Button, new() { Name = "Sign in" });

    public async Task<DashboardPage> LoginAsync(string email, string password)
    {
        await Email.FillAsync(email);
        await Password.FillAsync(password);
        await Submit.ClickAsync();
        return new DashboardPage(page);
    }
}
```

---

## 7. Test Authoring Rules

- Inherit from `PageTest` (NUnit) so each test gets an isolated `Page`/`Context`. **Never share browser state across tests.**
- Tests must be **independent and order-agnostic** — no test depends on another running first.
- **No conditionals (`if`) or loops driving test logic.** A test with branching is two tests.
- Use `[SetUp]`/`[TearDown]` or fixtures for arrange; keep the test body Arrange-Act-Assert and readable.
- Name tests as `Should_<expectedBehavior>_When_<condition>`.
- One scenario per test method. Use `[TestCase]` for true data-driven variations only.
- No `try/catch` to swallow failures. Let assertions fail loudly.
- No commented-out tests. Delete or `[Ignore("reason + ticket")]`.

```csharp
[Test]
public async Task Should_ShowDashboard_When_CredentialsValid()
{
    var login = new LoginPage(Page);
    await Page.GotoAsync(Settings.BaseUrl);

    var dashboard = await login.LoginAsync(Settings.User, Settings.Password);

    await Expect(dashboard.Heading).ToBeVisibleAsync();
}
```

---

## 8. Data, Config & Secrets

- **No hardcoded URLs, credentials, tokens, or environment names** in test or page code.
- Read config from `appsettings.json` + environment variables; secrets only from env/CI secret store.
- Test data via builders/factories in `TestData/`; avoid magic strings shared implicitly.
- Each test creates/owns its data where feasible; clean up via API teardown, not UI.
- Prefer API setup (seed via HTTP) over slow UI setup for preconditions.

---

## 9. Isolation & Parallelism

- Tests run in parallel by default. Anything not parallel-safe must be marked and justified.
- No shared mutable static fields. No singletons holding `IPage`.
- Use a fresh `BrowserContext` per test (default with `PageTest`).
- Auth: reuse `storageState` via a global setup project for speed, but storage state is **read-only** per test.

---

## 10. Reliability & Debugging

- Enable trace, screenshot, and video on failure (configure in `.runsettings`):
  - `Trace: on-first-retry`, `Screenshot: only-on-failure`, `Video: retain-on-failure`.
- Retries allowed in CI only (`Retry=2`); a test that needs retries to pass locally is a **bug to fix**, not accept.
- Treat any flaky test as a defect: quarantine with a tracking ticket, do not delete the assertion.

---

## 11. Clean Code (STRICT — enforced on every file)

### 11.1 Functions / Methods
- A method does **one thing**, at **one level of abstraction**. Mixing levels is a code smell — extract.
- Target ≤ **20 lines**, ideally ≤ 10. A longer method must be split into named helpers.
- Max **4 parameters**. Beyond that, introduce a DTO / options object / request model.
- **No flag/boolean arguments** that switch behavior — split into separate methods.
- No "super functions" / god methods that orchestrate, parse, assert, and log all at once.

### 11.2 Classes
- **Single Responsibility:** one reason to change. No "super class" / god object holding unrelated concerns.
- A Page Object models **one page or component** — never a catch-all `Helpers` / `Utils` / `Common` dumping ground.
- Depend on **abstractions** (interfaces), not concretions. Inject dependencies; no `new` of services inside methods.
- Page Objects are `sealed` unless explicitly designed for inheritance.

### 11.3 No dummy / dead / junk code
- ❌ **No dummy/placeholder code** — no `throw new NotImplementedException()` left behind, no empty method bodies, no `// TODO` shipped without a tracking ticket reference.
- ❌ **No dead code** — no unused fields, locals, usings, parameters, private methods, or unreachable branches.
- ❌ **No commented-out code.** Delete it; version control is the history.
- ❌ **No speculative "might-need-later" abstractions** (YAGNI). Build only what the current test needs.
- ❌ **No copy-paste duplication** (DRY). Repeated logic → extract a named helper or builder.
- ❌ **No `Console.WriteLine` / debug scaffolding** left in committed code.

### 11.4 Naming
- Names are self-documenting and use domain terminology. **No abbreviations** (`cust` → `customer`), no meaningless names (`temp`, `data`, `str`, `x`, `obj`).
- Methods are **verbs** (`SubmitOrder`), classes are **nouns** (`CheckoutPage`), booleans read as predicates (`IsVisible`, `HasError`).
- A well-named method removes the need for a comment.

### 11.5 Comments — minimal and intentional
- **Do NOT comment steps** — narrating obvious actions is forbidden. The code already says what it does.

  ```csharp
  // ❌ BAD — step narration
  // Click the login button
  await Submit.ClickAsync();

  // ✅ GOOD — no comment needed; the call is self-explanatory
  await Submit.ClickAsync();
  ```

- **Only comment complex functions** — explain the **why** (non-obvious business rule, tricky timing, workaround for a known bug + ticket), never the **what**.

  ```csharp
  // ✅ GOOD — explains non-obvious reasoning
  // Vendor API returns 202 then settles async; poll the status endpoint
  // instead of asserting on the immediate response. See JIRA-1423.
  ```
- Use XML `/// <summary>` only on shared, reusable public APIs (base fixtures, shared helpers), not on every test or trivial member.
- Never leave misleading or stale comments.

---

## 12. Async / C# Hygiene

- Every Playwright call is `await`-ed. No `.Result`, no `.Wait()`, no `.GetAwaiter().GetResult()`.
- All test methods are `async Task` (never `async void`).
- `Nullable` enabled; no `!` null-forgiving to silence the compiler without cause.
- No `console`/`Console.WriteLine` debug noise left in committed code.
- Page Objects `sealed` unless designed for inheritance.

---

## 13. Forbidden — quick reference

| ❌ Never | ✅ Instead |
|---|---|
| `Thread.Sleep` / `WaitForTimeout` | web-first `Expect` assertions, scoped waits |
| XPath / styling CSS selectors | `GetByRole` / `GetByLabel` / `GetByTestId` |
| `Assert.IsTrue(await x.IsVisibleAsync())` | `await Expect(x).ToBeVisibleAsync()` |
| Shared `IPage` across tests | per-test `Page` from `PageTest` |
| Hardcoded URL / password | config + env / secret store |
| `if`/`for` controlling test flow | separate tests / `[TestCase]` |
| `.Result` / `.Wait()` | `await` |
| UI-driven data setup/teardown | API seeding + API cleanup |
| Catching exceptions in tests | let it fail |
| Step-narrating comments | self-explanatory code; comment only complex *why* |
| Dummy / commented-out / dead code | delete it |
| God class / super function | split by single responsibility |
| `NotImplementedException` placeholder | implement or don't add it |
| Vietnamese / non-English in code/comments | English only |
| Abbreviations / `temp` / `data` / `x` names | full domain names |

---

## 14. Definition of Done (per test)

- [ ] Uses semantic locators (no XPath/styling CSS).
- [ ] Zero `Sleep`/`WaitForTimeout`; all waits are auto-wait or scoped.
- [ ] Web-first `Expect` assertions only for UI state.
- [ ] Independent, parallel-safe, order-agnostic.
- [ ] No hardcoded config or secrets.
- [ ] Arrange-Act-Assert, single scenario, descriptive name.
- [ ] Methods ≤ 20 lines, ≤ 4 params, single responsibility.
- [ ] No dummy/dead/commented-out code; no step-narrating comments.
- [ ] All code, names, and comments in English.
- [ ] Passes 10×/10× locally without retries.
