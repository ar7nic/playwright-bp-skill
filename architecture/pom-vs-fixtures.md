# Organizing Reusable Test Code

## Table of Contents

1. [The Layers](#the-layers)
2. [Layer Selection](#layer-selection)
3. [Page Objects](#page-objects)
4. [Custom Fixtures](#custom-fixtures)
5. [Helper Functions](#helper-functions)
6. [Combined Project Structure](#combined-project-structure)
7. [Anti-Patterns](#anti-patterns)

This framework organises reusable code into **layers**, each with one responsibility, each
calling only the layer below:

- **`pages/` (page objects)** — locators only, declared via the `this.el` factory
- **`components/`** — reusable UI blocks with small self-contained interactions
- **`assistants/`** — multi-step, cross-page workflows (behaviour)
- **`asserts/`** — verification helpers (web-first `expect` wrapped in `test.step`)
- **`fixtures/`** — the auto-registering DI container that provides all of the above
- **`utils/`** — `Element` / `ElementFactory` / `BasePage` infrastructure
- **helpers / `config/`** — stateless utilities and constants (see note in
  [Helper Functions](#helper-functions))

Playwright fixtures still underpin everything — but instead of writing one `test.extend()`
per page, a single container builds the whole object graph per test and exposes each piece
as a fixture (see [Custom Fixtures](#custom-fixtures)).

## The Layers

| Layer | Purpose | Lifecycle | Holds |
|---|---|---|---|
| **Page object** | Encapsulate a page's locators | Built per test by the container | Locators only (`this.el`) |
| **Component** | Encapsulate a reusable UI block | Built per test | Locators + small interactions |
| **Assistant** | Multi-step / cross-page flows | Built per test (reads `AppDeps`) | Behaviour |
| **Assert** | Verification | Built per test | `expect` wrapped in `test.step` |
| **Custom fixture** | Resources with setup/teardown | Built-in (`use()`) | Auth state, DB, API clients |
| **Helper** | Stateless utilities | None | Pure functions / constants |

## Layer Selection

```text
What kind of reusable code?
|
+-- A page's selectors?                  --> PAGE OBJECT (locators via this.el)
+-- A reusable UI block (navbar, modal)? --> COMPONENT
+-- A multi-step / cross-page flow?       --> ASSISTANT
+-- A verification / assertion?           --> ASSERT (CommonAsserts-style)
+-- A resource needing setup AND teardown (auth, DB, API client, test user)?
|                                        --> CUSTOM FIXTURE
+-- A stateless utility (random email, format date, build URL)?
                                         --> HELPER / config
```

## Page Objects

A page object holds **locators only** (declared through the `this.el` factory) — one class
per logical page, extending `BasePage`. Behaviour goes to assistants; assertions go to
asserts.

```typescript
// pages/BookingPage.ts
import { BasePage } from '../utils/BasePage';

export class BookingPage extends BasePage {
  readonly PAGE_NAME = 'BookingPage';

  get dateField()  { return this.el.byLabel('Check-in date', 'Check-in date'); }
  get guestCount() { return this.el.byLabel('Guests', 'Guests'); }
  get roomType()   { return this.el.byLabel('Room type', 'Room type'); }
  get reserveBtn() { return this.el.byRole('Reserve button', 'button', { name: 'Reserve' }); }
  get totalPrice() { return this.el.byTestId('Total price', 'total-price'); }
}
```

```typescript
// assistants/BookingAssistant.ts — the flow that uses the page
async reserveStandardRoom(date: string, guests: number): Promise<void> {
  await test.step('Booking → reserve standard room', async () => {
    await this.deps.page.goto('/booking');
    await this.deps.bookingPage.dateField.fill(date);
    await this.deps.bookingPage.guestCount.fill(String(guests));
    await this.deps.bookingPage.roomType.selectOption('standard');
    await this.deps.bookingPage.reserveBtn.click();
    await this.deps.page.waitForURL('**/confirmation');
  });
}
```

```typescript
// tests/booking.spec.ts
import { test } from '../fixtures';

test('complete reservation with standard room', async ({ bookingAssistant, commonAsserts }) => {
  await bookingAssistant.reserveStandardRoom('2026-03-15', 2);
  await commonAsserts.elementIsVisible(/* bookingPage.confirmation */);
});
```

**Page object principles:**
- One class per logical page/component, not per URL
- Extend `BasePage`; set `PAGE_NAME`; declare locators via `this.el`
- **Locators only** — no flow methods, no `expect`
- Give every locator a human-readable name (the first factory argument)

## Custom Fixtures

Pages, components, assistants, and asserts are **auto-registered** from their barrels by the
DI container — you do **not** hand-write a fixture for each. The container builds them once
per test and exposes each by its camelCase name (see
[fixtures-hooks.md](fixtures-hooks.md) and `fixtures/container.ts`):

```typescript
// tests/booking.spec.ts
import { test } from '../fixtures';

test('member sees dashboard widgets', async ({ dashboardPage, commonAsserts }) => {
  await commonAsserts.elementIsVisible(dashboardPage.statsWidget);
});
```

Reserve **hand-written `test.extend()` fixtures for true resources with a lifecycle** —
auth state, database connections, API clients, test users — i.e. anything that needs setup
before and teardown after the test. Compose these on top of the container's `test`:

```typescript
// fixtures/member.fixture.ts
import { generateMember } from '../helpers/data';
import { test as base } from './index';   // the container's test

type MemberFixture = { member: { email: string; password: string; id: string } };

export const test = base.extend<MemberFixture>({
  member: async ({ request }, use) => {
    const res = await request.post('/api/test/members', { data: generateMember() });
    const member = await res.json();
    await use(member);
    await request.delete(`/api/test/members/${member.id}`);  // guaranteed teardown
  },
});

export { expect } from './index';
```

**Fixture principles:**
- The container handles pages/components/assistants/asserts — add to a barrel, not a fixture
- Use `test.extend()` for **resources with lifecycle** — never module-level variables
- `use()` callback separates setup from teardown; teardown runs even if the test fails
- Fixtures compose: one can depend on another; they are lazy (created only when requested)

## Helper Functions

Best for stateless utilities — generating test data, formatting values, building URLs, parsing responses.

> **Test-data split (recommended).** Use **both**, by responsibility:
> `config/` (`env.ts`, `testData.ts`) holds environment, credentials, storageState paths,
> and fixed negative data; `helpers/` factories generate the entities a test **creates**
> (randomised per test, shown below — avoids collisions under `fullyParallel`). Keep both
> **pure** and free of browser/`expect` logic. For verification, use the `asserts/` layer
> ([assertions-waiting.md](../core/assertions-waiting.md)), not assertion helpers. See
> `decide.md` (item 11).

```typescript
// helpers/data.ts
import { randomUUID } from 'node:crypto';

export function generateEmail(prefix = 'user'): string {
  return `${prefix}-${Date.now()}-${randomUUID().slice(0, 8)}@test.local`;
}

export function generateMember(overrides: Partial<Member> = {}): Member {
  return {
    email: generateEmail(),
    password: 'SecurePass456!',
    name: 'Test Member',
    ...overrides,
  };
}

interface Member {
  email: string;
  password: string;
  name: string;
}

export function formatPrice(cents: number): string {
  return `$${(cents / 100).toFixed(2)}`;
}
```

Generated data flows into an assistant via the test, while verification stays in asserts:

```typescript
// tests/account.spec.ts
import { generateEmail } from '../helpers/data';
import { test } from '../fixtures';

test('update account email', async ({ accountAssistant, commonAsserts, accountPage }) => {
  const newEmail = generateEmail('updated');
  await accountAssistant.changeEmail(newEmail);
  await commonAsserts.inputHasValue(accountPage.emailInput, newEmail);
});
```

**Helper principles:**
- Pure functions with no side effects; no browser state, no `expect`
- Promote to a fixture if setup/teardown is needed
- Keep small and focused

## Combined Project Structure

```text
tests/          # specs only
assistants/     # AuthAssistant.ts, BookingAssistant.ts  + index.ts
asserts/        # CommonAsserts.ts, ...                   + index.ts
pages/          # LoginPage.ts, BookingPage.ts, ...       + index.ts
components/     # NavBar.ts, Modal.ts, ...                + index.ts
fixtures/       # container.ts, index.ts (auto-registering DI)
utils/          # BasePage.ts, Element.ts, ElementFactory.ts
helpers/        # data.ts — factories for entities a test creates (randomised)
config/         # env.ts, testData.ts — environment, creds, fixed negative data
playwright.config.ts
```

**Layer responsibilities:**

| Layer | Pattern | Responsibility |
|---|---|---|
| **Test file** | `test()` from `fixtures/` | Describes behavior, orchestrates layers |
| **Assistants** | Classes (`AppDeps`) | Multi-step / cross-page flows |
| **Asserts** | Classes | Verification (`expect` wrapped in `test.step`) |
| **Page objects** | Classes (`BasePage`) | Locators only (`this.el`) |
| **Components** | Classes | Reusable UI blocks + small interactions |
| **Fixtures** | DI container + `test.extend()` | Wire layers; resource lifecycle |
| **Helpers / config** | Functions / constants | Data, formatting, env (stateless) |

## Anti-Patterns

### Behaviour or assertions in a page object

```typescript
// BAD: page object with flows, API calls, and expect
class LoginPage {
  async createUser() { /* API call */ }
  async signIn(email, password) { /* UI flow */ }
  async expectError(msg) { /* expect */ }
}
```

Pages hold **locators only**. Put flows in `assistants/`, verification in `asserts/`, and
resource lifecycle (API/DB) in a `test.extend()` fixture where teardown is guaranteed.

### Calling Playwright locators directly

```typescript
// BAD: raw locator in a test or page object
await page.getByRole('button', { name: 'Login' }).click();
```

Declare locators via `this.el` and act through the `Element` wrapper so every action
auto-logs as a `test.step`. See [locators.md](../core/locators.md).

### Monolithic fixtures

```typescript
// BAD: one fixture doing everything
test.extend({
  everything: async ({ page, request }, use) => {
    const user = await createUser(request);
    const products = await seedProducts(request, 50);
    await setupPayment(request, user.id);
    await page.goto('/dashboard');
    await use({ user, products, page });
    // massive teardown...
  },
});
```

Break into small, composable fixtures. Each fixture does one thing.

### Helpers with side effects

```typescript
// BAD: module-level state
let createdUserId: string;

export async function createTestUser(request: APIRequestContext) {
  const res = await request.post('/api/users', { data: { email: 'test@example.com' } });
  const user = await res.json();
  createdUserId = user.id; // shared across tests!
  return user;
}
```

Module-level state leaks between parallel tests. If it has side effects and needs cleanup, make it a fixture.

### Over-abstracting simple operations

```typescript
// BAD: helper for one-liner
export async function clickButton(page: Page, name: string) {
  await page.getByRole('button', { name }).click();
}
```

Only abstract when there is real duplication (3+ usages) or complexity (5+ interactions).
