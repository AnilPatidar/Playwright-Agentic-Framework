# Playwright E2E Architecture

This document describes the design pattern and layering used by the Playwright
E2E suite in `src/e2e/`. It's the developer-facing companion to
`.claude/CLAUDE.qa.md` (agent context) and `.claude/CLAUDE.md`.

## Stack

- **App:** Next.js 14 (App Router) + TypeScript + Mantine v6 + Auth0
- **E2E:** `@playwright/test` 1.60+
- **Target environment:** deployed QA app (`https://qa.turbomechanica.ai` /
  `https://next.turbomechanica.ai`) — there is no local dev server involved;
  Playwright always drives a real deployed environment.
- **Auth:** pre-authenticated via a saved session at `.auth/admin-state.json`,
  role is switched per-test via a `user_role` cookie (not per-role login).

## Pattern in one sentence

**Page Object Model**, composed through a **Facade** (`App`), with locators
built by a **Factory** (`LocatorFactory`), and business logic split into two
single-purpose layers — **Flows** (orchestration, no assertions) and
**Assertions** (pure `expect()` helpers, no DOM interaction) — wired into
specs via a **fixture** that injects role and auth state.

```
spec.ts  ──uses──▶  app(role) fixture  ──creates──▶  App (facade)
                                                        │
                                    ┌───────────────────┼───────────────────┐
                                    ▼                                       ▼
                            <feature>-page.ts                      <feature>-page.ts
                            (extends BasePage)                     (extends BasePage)
                                    │
                                    ▼
                            LocatorFactory (buttons/inputs/tables/dialogs/...)

flows/<feature>/*.ts   — orchestrates page objects, no expect() assertions
assertions/<feature>/*.ts — pure expect() helpers, no DOM interaction

spec.ts composes: app(role) → page object → flow → assertion, inside
test.step() blocks, following Arrange → Act → Assert.
```

## Directory map

```
src/e2e/
├── assertions/<feature>/          Pure expect() helpers — no DOM interaction
├── config/test-config.ts          AUTH0_CONFIG, E2E_CONFIG.timeouts, storage state path
├── enums/timeouts.ts              Timeouts enum
├── enums/user-roles.ts            UserRole enum
├── fixtures/base.ts               app(UserRole) fixture — entry point for tests
├── fixtures/<feature>/            CSV, JSON test data files
├── flows/<feature>/               Async orchestration — no assertions
├── locators/common-locators.ts    LocatorFactory pattern for standard UI elements
├── locators/<feature>-locators.ts Feature-specific locator factories
├── pages/app.ts                   Composite/facade: exposes all page objects
├── pages/base-page.ts             Abstract base (goto retry, waits, reload)
├── pages/<feature>-page.ts        Locator getters (sync) + async wait methods
├── reporters/custom-reporter.ts   Role-based test result reporter
├── setup/                         Auth0 login → .auth/admin-state.json
├── tests/<feature>/               *.spec.ts — Arrange → Act → Assert
├── utils/api-client.ts            Typed API operations (teams, components, sensors)
├── utils/auth0-helpers.ts         Auth flow helpers (handleRoleSelection, etc.)
├── utils/test-helpers.ts          waitForAPIResponse, retryAction, mockAPIResponse
├── agents/plans/                  Agent-generated test plans (committed)
├── agents/lessons.md              Accumulated locator/timing/data lessons (committed)
└── agents/runs/                   Agent run logs — ephemeral, gitignored
```

---

## Layers

### 1. `BasePage` — abstract base (`pages/base-page.ts`)

Every page object extends this. It owns navigation with retry/backoff, load
waiting, and a `LocatorFactory` instance so subclasses get `this.locators.*`
for free.

```typescript
export abstract class BasePage {
  protected locators: LocatorFactory;

  constructor(
    protected page: Page,
    protected path: string = "/",
  ) {
    this.locators = new LocatorFactory(page);
  }

  async goto(options?: { waitUntil?: "load" | "domcontentloaded" | "networkidle"; timeout?: number }): Promise<void> {
    const maxRetries = 3;
    let attempt = 0;
    while (attempt < maxRetries) {
      try {
        if (this.page.url().includes(this.path.split("?")[0])) return;
        await this.page.goto(this.path, { waitUntil: options?.waitUntil || "domcontentloaded" });
        await this.page.waitForLoadState("domcontentloaded", { timeout: 30000 });
        return;
      } catch (error) {
        // exponential-backoff retry on network/navigation errors
      }
    }
  }

  async reload(): Promise<void> {
    await this.withNetworkRetry(async () => {
      await this.page.reload({ waitUntil: "domcontentloaded", timeout: 30000 });
      await this.page.waitForLoadState("networkidle", { timeout: 20000 });
    }, "Page reload");
  }
}
```

Also provided: `waitForLoad()`, `waitForNavigation(callback)`,
`withNetworkRetry<T>()`, `waitForNetworkStability()`, generic `click`/`fill`/
`getText`/`waitFor`/`isVisible` helpers.

### 2. Concrete page objects (`pages/<feature>-page.ts`)

Locator getters (sync) + async wait/navigation helpers. Example —
`SensorsLabelPage`:

```typescript
export class SensorsLabelPage extends BasePage {
  constructor(page: Page) {
    super(page, "/settings/labels");
  }

  getPageHeading(): Locator {
    return this.page.getByRole("heading", { name: "Labels", level: 1 });
  }

  getSearchInput(): Locator {
    return this.page.getByPlaceholder("Search by Name");
  }

  getLabelsTable(): Locator {
    return this.page.getByRole("table");
  }

  async navigateAndWait(): Promise<string> {
    await this.page.goto(this.path);
    await this.page.waitForLoadState("networkidle");
    await Promise.race([
      this.getPageHeading().waitFor({ state: "visible", timeout: Timeouts.EXTRA_LONG_WAIT }),
      this.getLabelsTable().waitFor({ state: "visible", timeout: Timeouts.EXTRA_LONG_WAIT }),
    ]).catch(() => {});
    return await this.getCurrentURL();
  }
}
```

### 3. `App` — composite/facade (`pages/app.ts`)

Instantiates every page object against the same `Page` and exposes them as
one entry point, so tests never construct page objects directly.

```typescript
export class App {
  public readonly dashboardPage: DashboardPage;
  public readonly sensorsLabelPage: SensorsLabelPage;
  public readonly equipmentSensorsPage: EquipmentSensorsPage;
  // ...14 more page objects

  constructor(
    public readonly page: Page,
    facilityId: number = 1,
    equipmentId: number = 1,
  ) {
    this.dashboardPage = new DashboardPage(page);
    this.sensorsLabelPage = new SensorsLabelPage(page);
    this.equipmentSensorsPage = new EquipmentSensorsPage(page, facilityId, equipmentId);
    // ...
  }

  async navigateTo(pageName: "dashboard" | "equipment" | "diagnostics" | "settings"): Promise<void> {
    const pages = { dashboard: this.dashboardPage, equipment: this.equipmentPage, /* ... */ };
    await pages[pageName].goto();
  }
}
```

### 4. `LocatorFactory` — factory pattern (`locators/common-locators.ts`)

Per-domain locator factories (`ButtonLocators`, `InputLocators`,
`NavigationLocators`, `DialogLocators`, `TableLocators`,
`NotificationLocators`) combined into one `LocatorFactory`, so common
selectors aren't hand-duplicated across page objects.

```typescript
export class ButtonLocators {
  constructor(private page: Page) {}

  getByRole(name: string): Locator {
    return this.page.getByRole("button", { name });
  }

  submit(): Locator {
    return this.page.getByRole("button", { name: /submit|save|create/i });
  }
}

export class LocatorFactory {
  public buttons: ButtonLocators;
  public inputs: InputLocators;
  public tables: TableLocators;
  // ...

  constructor(private page: Page) {
    this.buttons = new ButtonLocators(page);
    this.inputs = new InputLocators(page);
    // ...
  }

  getByTestId(testId: string): Locator {
    return this.page.getByTestId(testId);
  }
}
```

### 5. Flows — orchestration, no assertions (`flows/<feature>/*.ts`)

Multi-step actions built out of page objects. Note: flows do use `expect()`
internally as **deterministic wait-gates** (e.g. "wait until input has this
value before continuing"), not as test assertions — the pure assertion layer
is `assertions/`.

```typescript
export async function navigateToLabelsPage(appInstance: App): Promise<SensorsLabelPage> {
  const labelsPage = appInstance.sensorsLabelPage;
  await labelsPage.navigateAndWait();
  return labelsPage;
}

export async function searchLabels(
  appInstance: App,
  labelsPage: SensorsLabelPage,
  query: string,
): Promise<void> {
  const searchInput = labelsPage.getSearchInput();
  await expect(searchInput).toBeVisible({ timeout: Timeouts.LONG_WAIT });
  await searchInput.click();
  await searchInput.fill(query);
  await expect(searchInput).toHaveValue(query); // wait-gate, not a test assertion
  await appInstance.page.waitForLoadState("networkidle");
}
```

### 6. Assertions — pure `expect()` helpers (`assertions/<feature>/*.ts`)

No DOM interaction — just checks.

```typescript
export function assertLabelsPageUrl(url: string): void {
  expect(url).toContain("/settings/labels");
}

export async function assertLabelsTableHeaders(labelsPage: SensorsLabelPage): Promise<void> {
  const table = labelsPage.getLabelsTable();
  await expect(table).toContainText("Id");
  await expect(table).toContainText("Name");
  await expect(table).toContainText("Type");
}

export async function assertLabelsPageLoaded(labelsPage: SensorsLabelPage): Promise<void> {
  await assertLabelsHeadingVisible(labelsPage);
  await assertSearchInputVisible(labelsPage);
  await assertLabelsTableVisible(labelsPage);
  await assertLabelsTableHeaders(labelsPage);
}
```

### 7. Fixture — `app(UserRole)` entry point (`fixtures/base.ts`)

Injects role via a `user_role` cookie and hands back a ready `App` facade.

```typescript
export type TestFixtures = {
  app: (userRole?: number, ids?: AppIds) => Promise<App>;
};

async function launchApp(page: Page, userRole?: number, ids?: AppIds): Promise<App> {
  const app = new App(page, ids?.facilityId, ids?.equipmentId);

  if (userRole !== undefined) {
    const baseUrl = (page.context() as { _options?: { baseURL?: string } })._options?.baseURL;
    await page.context().addCookies([
      { name: "user_role", value: String(userRole), domain: new URL(baseUrl!).hostname, path: "/", secure: true },
    ]);
  }

  await page.goto("/");
  return app;
}

export const test = base.extend<TestFixtures, WorkerFixtures>({
  app: async ({ page }, provide, testInfo) => {
    provide(async (userRole, ids) => {
      await setupNetworkMonitoring(page, testInfo);
      return await launchApp(page, userRole, ids);
    });
  },
});

export { expect } from "@playwright/test";
```

### 8. Config, enums, utils

```typescript
// config/test-config.ts
export const E2E_CONFIG = {
  timeouts: { short: 5000, medium: 15000, long: 30000, navigation: 10000, api: 5000 },
} as const;

export const AUTH0_CONFIG = {
  domain: process.env.AUTH0_ISSUER_BASE_URL,
  username: process.env.AUTH0_USERNAME,
  password: process.env.AUTH0_PASSWORD,
} as const;

export const AUTH_STORAGE_STATE = ".auth/admin-state.json";
```

> ⚠️ There are two parallel timeout systems: `E2E_CONFIG.timeouts` (used by
> `utils/test-helpers.ts`) and the `Timeouts` enum (used pervasively by page
> objects/flows/assertions). Prefer `Timeouts` for new page/flow/assertion
> code to stay consistent with the rest of the suite.

```typescript
// enums/user-roles.ts
export enum UserRole {
  POWER_USER = 1,
  ADMINISTRATOR = 2,
  SENSOR_MANAGER = 6,
  // ...
}

// enums/timeouts.ts
export enum Timeouts {
  SHORT_WAIT = 2000,
  LONG_WAIT = 5000,
  EXTRA_LONG_WAIT = 10000,
  PAGE_LOAD_WAIT = 30000,
  // ...
}
```

```typescript
// utils/test-helpers.ts
export async function waitForAPIResponse(
  page: Page,
  urlPattern: string | RegExp,
  options?: { timeout?: number; status?: number },
): Promise<unknown> { /* page.waitForResponse(...).json() */ }

export async function retryAction<T>(
  action: () => Promise<T>,
  options?: { retries?: number; delay?: number; onRetry?: (attempt: number, error: Error) => void },
): Promise<T> { /* retry with linear backoff */ }

export async function mockAPIResponse(
  page: Page,
  urlPattern: string | RegExp,
  responseData: unknown,
  status = 200,
): Promise<void> { /* page.route(...).fulfill(...) */ }
```

```typescript
// utils/api-client.ts — direct API calls reusing the browser session's cookies,
// used for test-data setup/cleanup, not for UI assertions
export class ApiClient {
  static async fromPage(page: Page): Promise<ApiClient> { /* ... */ }
  async listAllTeams(pageSize = 200): Promise<TeamOutSchema[]> { /* ... */ }
  async deleteTeamsMatching(predicate: (name: string) => boolean): Promise<number> { /* ... */ }
  async getAllComponents(equipmentId: number): Promise<ComponentNode[]> { /* ... */ }
}
```

### 9. Auth setup (`setup/auth.setup.ts`)

Runs once as the `auth-setup` Playwright project and writes the session used
by every other project via `storageState`.

```typescript
setup("authenticate", async ({ page, context }) => {
  await completeAuth0Login(
    page,
    context,
    { username: AUTH0_CONFIG.username, password: AUTH0_CONFIG.password },
    { domain: AUTH0_CONFIG.domain, appUrl: AUTH0_CONFIG.appUrl },
  );
  await page.waitForLoadState("domcontentloaded");
  expect(await validateAuth0Session(page)).toBeTruthy();
  await context.storageState({ path: AUTH_STORAGE_STATE }); // .auth/admin-state.json
});
```

### 10. A full spec — composing everything (`tests/sensors/sensors-label.spec.ts`)

```typescript
import { test } from "@e2e/fixtures/base";
import { UserRole } from "@e2e/enums/user-roles";
import { waitForLabelsHeading } from "@e2e/flows/sensors/sensors-label.flows";
import {
  assertLabelsPageUrl,
  assertLabelsTableHeaders,
  assertLabelsPageLoaded,
} from "@e2e/assertions/sensors/sensors-label.assertions";
import { expectText } from "@e2e/assertions/ui-diff.assertions";

test.describe("Sensors Labels", () => {
  for (const { role, label } of [
    { role: UserRole.ADMINISTRATOR, label: "Administrator" },
    { role: UserRole.SENSOR_MANAGER, label: "Sensor Manager" },
  ]) {
    test(`SL-01 Page loads and renders correctly for ${label} @smoke @regression`, async ({ app }) => {
      const appInstance = await app(role);
      const labelsPage = appInstance.sensorsLabelPage;

      await test.step("Navigate to Sensors Label page", async () => {
        const url = await labelsPage.navigateAndWait();
        assertLabelsPageUrl(url);
      });

      await test.step("Verify page heading", async () => {
        await waitForLabelsHeading(labelsPage);
        await expectText(labelsPage.getPageHeading(), "Labels", { context: "Sensors Labels > page heading" });
      });

      await test.step("Verify table structure and page controls are rendered", async () => {
        await assertLabelsTableHeaders(labelsPage);
        await assertLabelsPageLoaded(labelsPage);
      });
    });
  }
});
```

This demonstrates the whole chain: `app(role)` fixture → `App` facade →
page object → flow helper → assertion helper, wrapped in `test.step()`
blocks that mirror Arrange → Act → Assert, plus tag-driven filtering
(`@smoke`, `@regression`) and data-driven role loops.

---

## Playwright projects (`playwright.config.ts`)

There is **no `webServer` block** — the suite always targets a real deployed
environment (`baseURL` from `.env.local` / defaults to
`https://next.turbomechanica.ai`), never a locally-started dev server.

| Project | Purpose |
|---|---|
| `setup` | Runs `global.setup.ts`; paired with `cleanup` as its teardown |
| `auth-setup` | Runs `auth.setup.ts` → writes `.auth/admin-state.json` |
| `test-data-setup` | Seeds test data; depends on `auth-setup` |
| `cleanup` | Runs `global.cleanup.ts` |
| `smoke` | `grep: /@smoke/` |
| `regression` | `grep: /@regression/`, `grepInvert: /@first-seen/` |
| `wip` | `grep: /@wip/`, records video, 1 retry |
| `first-seen` | `grep: /@first-seen/` — run explicitly, never alongside `chromium` |
| `flaky` | `grep: /@flaky/`, 0 retries |
| `chromium` | `grepInvert: /@wip|@first-seen/` — everything else |

All test-running projects depend on `setup`, `auth-setup`, and
`test-data-setup`, and consume the shared `storageState: AUTH_STORAGE_STATE`.

## npm scripts

```
npm run test:e2e              # --project=regression
npm run test:e2e:smoke        # --project=smoke
npm run test:e2e:regression   # --project=regression
npm run test:e2e:wip          # --project=wip
npm run test:e2e:chromium     # --project=chromium
npm run test:e2e:flaky        # --project=flaky
npm run test:e2e:first-seen   # --project=first-seen
npm run test:e2e:debug        # --debug
npm run test:e2e:report       # playwright show-report
```

> ⚠️ `test:e2e:firefox` and `test:e2e:webkit` exist in `package.json` but
> there are no matching `firefox`/`webkit` projects in `playwright.config.ts`
> — running them currently fails with "Project(s) not found."

---

## Why this shape

- **Facade (`App`)** keeps specs from wiring up 17+ page objects by hand and
  gives every test one shared `page` instance across all page objects.
- **Factory (`LocatorFactory`)** stops the same `getByRole("button", ...)`
  boilerplate from being copy-pasted into every page object.
- **Flows vs. Assertions split** lets the same orchestration (e.g. "search
  for a label") be reused across tests that check different things, and
  keeps assertion logic testable/reviewable in isolation.
- **Fixture-injected `App`** centralizes auth/role setup so specs never touch
  cookies or `page.goto("/")` directly.
- **`test.step()` + AAA** keeps specs readable in the HTML report and makes
  failures point at the exact phase (navigate / act / assert) that broke.

This is also the shape the Playwright agent-authoring pipeline
(`playwright-test-planner` → `-generator` → `-healer` → `-code-quality`,
see `.claude/CLAUDE.md`) is instructed to produce and enforce.
