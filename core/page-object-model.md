# Page Object Model (POM)

## Table of Contents

1. [Overview](#overview)
2. [Page Class (locators only)](#page-class-locators-only)
3. [Assistants (behaviour layer)](#assistants-behaviour-layer)
4. [Usage in Tests](#usage-in-tests)
5. [Component Objects](#component-objects)
6. [Composition Patterns](#composition-patterns)
7. [Factory Functions](#factory-functions)
8. [Best Practices](#best-practices)

## Overview

This framework splits the classic Page Object into focused layers:

- **`pages/`** — page objects that hold **locators only** (declared via the `this.el`
  factory). One class per logical page; no actions, no assertions.
- **`assistants/`** — multi-step, cross-page **workflows** (login, checkout) that drive the
  pages. This is where behaviour lives.
- **`asserts/`** — verification helpers (see [assertions-waiting.md](assertions-waiting.md)).

This keeps each piece single-responsibility: selectors are isolated from flows, flows are
isolated from assertions, and reports stay readable because every layer logs to `test.step`
automatically.

## Page Class (locators only)

A page extends `BasePage`, sets `PAGE_NAME`, and exposes locators through `this.el`
(see [locators.md](locators.md)). It contains **no flow methods and no `expect`** —
those belong in assistants and asserts respectively.

```typescript
// pages/LoginPage.ts
import { BasePage } from '../utils/BasePage';

export class LoginPage extends BasePage {
  readonly PAGE_NAME = 'LoginPage';

  get usernameInput() { return this.el.byLabel('Username input', 'Username'); }
  get passwordInput() { return this.el.byLabel('Password input', 'Password'); }
  get submitBtn()     { return this.el.byRole('Submit button', 'button', { name: 'Login' }); }
  get flashMessage()  { return this.el('Flash message', '.flash'); }
}
```

`BasePage` provides the `el` factory bound to `PAGE_NAME` (reference: `utils/BasePage.ts`):

```typescript
export abstract class BasePage {
  abstract readonly PAGE_NAME: string;
  protected get el(): ElementFactory {
    return createElementFactory(this.page, this.PAGE_NAME);
  }
  constructor(protected readonly page: Page) {}
}
```

> **Keep `BasePage` minimal.** Its job is to provide `el`. Thin, **non-navigation** readers
> (e.g. `getTitle()`) may be added if widely reused, but **navigation and flows stay in
> `assistants/`** — don't add an `abstract goto()` or other behaviour to the base class. See
> `decide.md` (item 13).

## Assistants (behaviour layer)

Assistants encapsulate the multi-step flows that used to live as methods on page objects.
They receive the per-test dependency container (`AppDeps`) and read pages/other assistants
lazily, so cross-layer calls work. Each public method is one logical flow.

```typescript
// assistants/AuthAssistant.ts
import { test } from '@playwright/test';
import type { AppDeps } from '../fixtures/container';

export class AuthAssistant {
  constructor(private readonly deps: AppDeps) {}

  async loginAndWaitForDashboard(username: string, password: string): Promise<void> {
    await test.step(`Auth → login as "${username}"`, async () => {
      await this.deps.page.goto('/login', { waitUntil: 'domcontentloaded' });
      await this.deps.loginPage.usernameInput.fill(username);
      await this.deps.loginPage.passwordInput.fill(password);
      await this.deps.loginPage.submitBtn.click();
      await this.deps.page.waitForLoadState('load');
    });
  }
}
```

> A `test.step` wrapping the **whole flow** is appropriate in an assistant (it groups the
> auto-logged element steps underneath). Do **not** wrap individual actions — the `Element`
> wrapper already does. See [annotations.md](annotations.md).

### Trimming assistant boilerplate

A thin `BaseAssistant` removes repeated `this.deps.page…` and gives one navigation+wait
helper. It lives in the same `assistants/` folder — no structural change:

```typescript
// assistants/BaseAssistant.ts
import type { AppDeps } from '../fixtures/container';

export abstract class BaseAssistant {
  constructor(protected readonly deps: AppDeps) {}
  protected get page() { return this.deps.page; }

  /** navigate + wait for load in one call */
  protected async open(path: string) {
    await this.page.goto(path, { waitUntil: 'domcontentloaded' });
    await this.page.waitForLoadState('load');
  }
}
```

Concrete assistants then read cleaner. Destructure `this.deps` at the top of a method to
cut noise further:

```typescript
export class AuthAssistant extends BaseAssistant {
  async loginAndWaitForDashboard(username: string, password: string) {
    const { loginPage } = this.deps;
    await test.step(`Auth → login as "${username}"`, async () => {
      await this.open('/login');
      await loginPage.usernameInput.fill(username);
      await loginPage.passwordInput.fill(password);
      await loginPage.submitBtn.click();
    });
  }
}
```

## Usage in Tests

Tests import `test`/`expect` from `fixtures/`, request the assistants/pages/asserts they
need as fixtures, and never touch raw locators or `expect`:

```typescript
// tests/login.spec.ts
import { THE_INTERNET_USER } from '../config/env';
import { INVALID_LOGIN } from '../config/testData';
import { test } from '../fixtures';

test.describe('Login page', () => {
  test('successful login redirects to secure area @smoke', async ({
    authAssistant,
    commonAsserts,
  }) => {
    await authAssistant.loginAndWaitForDashboard(
      THE_INTERNET_USER.username,
      THE_INTERNET_USER.password,
    );
    await commonAsserts.urlMatches(/.*\/secure/);
  });

  test('shows error for invalid credentials', async ({ authAssistant, commonAsserts }) => {
    const error = await authAssistant.loginExpectError(
      INVALID_LOGIN.username,
      INVALID_LOGIN.password,
    );
    await commonAsserts.textContains(error, 'Your username is invalid!');
  });
});
```

## Component Objects

Reusable UI blocks (`NavBar`, `Modal`) live in `components/`. They declare locators with
their own `ElementFactory` (named for the component) and — unlike pages — **may carry small
interaction methods** for the block they own (a navbar knows how to log out of itself).

```typescript
// components/NavBar.ts
import { Page } from '@playwright/test';
import { createElementFactory } from '../utils/ElementFactory';

export class NavBar {
  private get el() { return createElementFactory(this.page, 'NavBar'); }

  private get nav()         { return this.el('Nav', 'nav'); }
  private get userMenuBtn()  { return this.nav.getByTestId('User menu button', 'user-menu'); }
  private get logoutItem()   { return this.el.byRole('Logout menu item', 'menuitem', { name: 'Logout' }); }

  constructor(private readonly page: Page) {}

  async logout(): Promise<void> {
    await this.userMenuBtn.click();
    await this.logoutItem.click();
  }
}
```

### Component scoping (recommended)

Use a **hybrid** rule:

- **Global, single-instance blocks** (`NavBar`, `Header`) — build page-level with
  `createElementFactory(page, 'Name')` (as above) and let them be auto-registered as
  fixtures.
- **Repeated / nested blocks** (a table row, a card in a list) — scope to a parent
  container so selectors are constrained to that subtree. Build them from a root
  `Element`/`Locator` (e.g. via `Element.find(...)` / chaining) and create them explicitly
  where needed, rather than as a global fixture.

```typescript
// repeated block scoped to its container row
const row = dashboardPage.userRow('Alice');     // Element for one <tr>
await row.find('button', 'Edit button').click();
```

This keeps the common case compatible with the DI container while giving correct subtree
isolation for repeated components, where a page-level binding would match the wrong element.
See `decide.md` (item 14).

## Composition Patterns

### Page exposing a component

A page can expose a component getter so flows reach it through the page. The page still
declares only locators/components — the interaction logic stays in the component (or an
assistant for cross-page flows).

```typescript
// pages/DashboardPage.ts
import { BasePage } from '../utils/BasePage';
import { NavBar } from '../components/NavBar';

export class DashboardPage extends BasePage {
  readonly PAGE_NAME = 'DashboardPage';

  readonly navBar = new NavBar(this.page);

  get welcomeMsg()  { return this.el('Welcome message', '#flash'); }
  get newProjectBtn() { return this.el.byRole('New project button', 'button', { name: 'New Project' }); }
}
```

### Cross-page navigation

Navigation that spans pages belongs in an **assistant**, not in a page method that returns
another page object. The assistant drives the source page's locators, waits, then drives the
destination page:

```typescript
// assistants/AuthAssistant.ts  (excerpt)
async logout(): Promise<void> {
  await test.step('Auth → logout and return to login page', async () => {
    await this.deps.dashboardPage.navBar.logout();
    await this.deps.page.waitForLoadState('load');
  });
}
```

## Factory Functions

The `ElementFactory` (`createElementFactory(page, name)`) is the factory pattern this
framework uses — for **locators**, not whole pages. Page objects themselves are plain
classes extending `BasePage`; they are instantiated once per test by the DI container
(see [fixtures-hooks.md](fixtures-hooks.md)), so there is no need for hand-written page
factory functions.

## Best Practices

### Do

- **Keep pages locator-only** — declared via `this.el`; selectors are the single source of truth
- **Put multi-step flows in `assistants/`** and verification in `asserts/`
- **Give every locator a human name** (first factory argument) — it drives report steps
- **Let components own small interactions** for the block they encapsulate
- **Register classes via barrels** so the DI container wires fixtures automatically

### Don't

- **Don't add flow methods or `expect` to page objects** — use assistants / asserts
- **Don't call Playwright locators directly** — always go through `this.el` / `Element`
- **Don't add manual `test.step` in pages/components** — the `Element` wrapper logs already
- **Don't share state** between page object instances (the container builds fresh per test)

### Directory Structure

```
tests/          # specs only (import { test } from '../fixtures')
assistants/     # AuthAssistant.ts, ...   + index.ts (barrel)
asserts/        # CommonAsserts.ts, ...    + index.ts (barrel)
pages/          # LoginPage.ts, DashboardPage.ts, ... + index.ts (barrel)
components/     # NavBar.ts, Modal.ts, ... + index.ts (barrel)
fixtures/       # container.ts, index.ts (auto-registering DI)
utils/          # BasePage.ts, Element.ts, ElementFactory.ts
config/         # env.ts, testData.ts
playwright.config.ts
```

See [test-suite-structure.md](test-suite-structure.md) for naming and barrel conventions.

### Using with Fixtures

Pages, components, assistants, and asserts are **auto-registered** as fixtures from their
barrels — you do not write per-page `test.extend()` blocks. Just request them by their
camelCase name:

```typescript
import { test } from '../fixtures';

test('can login', async ({ authAssistant, commonAsserts }) => {
  await authAssistant.loginAndWaitForDashboard('tomsmith', 'SuperSecretPassword!');
  await commonAsserts.urlMatches(/.*\/secure/);
});
```

See [fixtures-hooks.md](fixtures-hooks.md) for how the container builds these.

## Related References

- **Locator strategies**: See [locators.md](locators.md) for selecting elements
- **Fixtures**: See [fixtures-hooks.md](fixtures-hooks.md) for advanced fixture patterns
- **Test organization**: See [test-suite-structure.md](test-suite-structure.md) for structuring test suites
