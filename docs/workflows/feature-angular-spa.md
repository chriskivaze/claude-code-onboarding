# Angular SPA Feature Development

> **When to use**: Building a new Angular 21.x page, component, service, or full feature end-to-end
> **Time estimate**: 2–4 hours per screen; 1–2 days for a full feature with multiple views
> **Prerequisites**: Angular project scaffolded; daisyUI + TailwindCSS 4.x configured

## Overview

Angular feature development using the `angular-spa` skill, Angular CLI MCP, and `angular-spa` agent. Follows standalone component architecture, signals-based state management, lazy routing, and daisyUI design tokens. Ends with `ui-standards-expert` + `accessibility-auditor` review.

---

## Iron Law (from `skills/angular-spa/SKILL.md`)

> **QUERY ANGULAR CLI MCP FOR CURRENT API BEFORE WRITING COMPONENT CODE**
> Angular 21.x has changed significantly from 15.x — `ngModule`, `@NgModule`, zone.js patterns are deprecated.

---

## Phases

### Phase 1 — Load Skill and Query Docs

**Skills to load**:
- `angular-spa` — workspace conventions, TailwindCSS 4.x, daisyUI 5.5.5 (always load first)
- `angular` — load when working with SSR, hydration, signal testing, or complex DI patterns
- `angular-best-practices` — load when reviewing code or optimizing performance
- `angular-ui-patterns` — load when building components with async data (loading/error/empty states)
**MCP**: Angular CLI MCP → query for current patterns before writing

```
mcp__angular-cli__list_projects         → find workspace path
mcp__angular-cli__get_best_practices    → get Angular 21.x standards
mcp__context7__resolve-library-id       → angular, @angular/core, @angular/router
mcp__context7__query-docs               → query for signals, standalone components
```

---

### Phase 2 — Scaffold Feature

**Command**: `/scaffold-angular-app [feature-name]`

**Generate standalone component**:
```bash
ng generate component features/orders/order-list --standalone --skip-tests=false
ng generate service features/orders/order
ng generate interface features/orders/models/order
```

**Feature directory structure**:
```
src/app/features/orders/
├── order-list/
│   ├── order-list.component.ts     # Standalone component
│   ├── order-list.component.html   # Template with daisyUI
│   ├── order-list.component.scss
│   └── order-list.component.spec.ts
├── order-detail/
│   ├── order-detail.component.ts
│   └── ...
├── models/
│   └── order.ts                    # Interfaces + types
├── services/
│   └── order.service.ts            # HTTP client service
└── orders.routes.ts                # Lazy routes
```

---

### Phase 3 — Implement with Signals

**Standalone component** (Angular 21.x pattern):
```typescript
// order-list.component.ts
import { Component, inject, signal, computed, OnInit } from '@angular/core';
import { AsyncPipe, NgFor, NgIf } from '@angular/common';
import { OrderService } from '../services/order.service';
import { Order } from '../models/order';

@Component({
  selector: 'app-order-list',
  standalone: true,
  imports: [NgFor, NgIf, AsyncPipe],
  templateUrl: './order-list.component.html',
})
export class OrderListComponent implements OnInit {
  private orderService = inject(OrderService);

  orders = signal<Order[]>([]);
  loading = signal(true);
  error = signal<string | null>(null);

  filteredOrders = computed(() =>
    this.orders().filter(o => o.status !== 'CANCELLED')
  );

  async ngOnInit() {
    try {
      const result = await this.orderService.getOrders();
      this.orders.set(result);
    } catch (err) {
      this.error.set('Failed to load orders');
      // No silent failure — error is visible to user
    } finally {
      this.loading.set(false);
    }
  }
}
```

**Template** (daisyUI semantic tokens):
```html
<!-- order-list.component.html -->
@if (loading()) {
  <div class="loading loading-spinner loading-lg"></div>
}

@if (error()) {
  <div class="alert alert-error">
    <span>{{ error() }}</span>
  </div>
}

@for (order of filteredOrders(); track order.id) {
  <div class="card bg-base-100 shadow-md">
    <div class="card-body">
      <h2 class="card-title">Order #{{ order.id }}</h2>
      <div class="badge badge-{{ order.status | lowercase }}">{{ order.status }}</div>
    </div>
  </div>
}

@if (!loading() && filteredOrders().length === 0) {
  <div class="hero min-h-32">
    <p class="text-base-content/60">No active orders</p>
  </div>
}
```

---

### Phase 4 — HTTP Service

```typescript
// order.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';
import { Order, CreateOrderRequest } from '../models/order';

@Injectable({ providedIn: 'root' })
export class OrderService {
  private http = inject(HttpClient);
  private baseUrl = '/api/orders';

  async getOrders(): Promise<Order[]> {
    return firstValueFrom(this.http.get<Order[]>(this.baseUrl));
  }

  async createOrder(request: CreateOrderRequest): Promise<Order> {
    return firstValueFrom(this.http.post<Order>(this.baseUrl, request));
  }
}
```

---

### Phase 5 — Lazy Routing

```typescript
// orders.routes.ts
import { Routes } from '@angular/router';
import { authGuard } from '../../core/guards/auth.guard';

export const ordersRoutes: Routes = [
  {
    path: '',
    canActivate: [authGuard],
    children: [
      {
        path: '',
        loadComponent: () => import('./order-list/order-list.component')
          .then(m => m.OrderListComponent),
      },
      {
        path: ':id',
        loadComponent: () => import('./order-detail/order-detail.component')
          .then(m => m.OrderDetailComponent),
      },
    ],
  },
];

// app.routes.ts — register lazy route
{
  path: 'orders',
  loadChildren: () => import('./features/orders/orders.routes')
    .then(m => m.ordersRoutes),
}
```

---

### Phase 6 — Unit Tests

```typescript
// order-list.component.spec.ts
describe('OrderListComponent', () => {
  let component: OrderListComponent;
  let orderService: jasmine.SpyObj<OrderService>;

  beforeEach(async () => {
    orderService = jasmine.createSpyObj('OrderService', ['getOrders']);
    await TestBed.configureTestingModule({
      imports: [OrderListComponent],
      providers: [{ provide: OrderService, useValue: orderService }],
    }).compileComponents();
    component = TestBed.createComponent(OrderListComponent).componentInstance;
  });

  it('loads orders on init', async () => {
    orderService.getOrders.and.resolveTo([{ id: '1', status: 'PENDING' }]);
    await component.ngOnInit();
    expect(component.orders()).toHaveSize(1);
    expect(component.loading()).toBeFalse();
  });

  it('sets error signal on service failure', async () => {
    orderService.getOrders.and.rejectWith(new Error('Network error'));
    await component.ngOnInit();
    expect(component.error()).toBeTruthy();
    expect(component.loading()).toBeFalse();
  });
});
```

**Run**:
```bash
ng test --watch=false
```

---

### Phase 7 — Design System Review

**Command**: `/lint-design-system`

**Agent 1**: `ui-standards-expert`
- No hardcoded colors (only daisyUI semantic: `bg-primary`, `text-base-content`)
- No raw spacing values (use Tailwind scale: `p-4`, not `p-[16px]`)
- Touch targets ≥ 44px for interactive elements

**Agent 2**: `accessibility-auditor`
- Semantic HTML (`<button>` not `<div>` for interactive elements)
- ARIA labels on icon-only buttons
- Color contrast ≥ 4.5:1 for normal text
- Keyboard navigation for all interactive elements

**Step 3**: `web-design-guidelines` skill
- Load skill and fetch guidelines from source URL
- Run against all changed component files
- Verify: loading states present, error states present, empty states present, focus management correct

**Gate**: `/lint-design-system` returns zero violations. Both agents pass.

---

### Phase 8 — Code Review and PR

```
/review-code        → dispatch angular-spa reviewer (ui-standards-expert + code-reviewer)
/validate-changes   → output-evaluator agent: APPROVE / NEEDS_REVIEW / REJECT
/ship               → comprehensive pre-deploy check
```

---

## Quick Reference

| Phase | Action | Agent / Command | Gate |
|-------|--------|----------------|------|
| 1 — Setup | Load skill + query Angular CLI MCP | MCP query | Current API confirmed |
| 2 — Scaffold | `/scaffold-angular-app` | Angular CLI | Directory structure created |
| 3 — Component | Standalone + signals | Manual | Component renders |
| 4 — Service | HTTP service | Manual | API calls work |
| 5 — Routing | Lazy routes | Manual | Navigation works |
| 6 — Tests | Jasmine unit tests | `ng test` | All tests pass |
| 7 — Design | `/lint-design-system` | `ui-standards-expert` + `accessibility-auditor` | Zero violations |
| 8 — Review | `/review-code` + `/ship` | Multiple agents | Zero CRITICAL findings |

---

## Common Pitfalls

- **Using `NgModule`** — deprecated in Angular 21.x; use standalone components only
- **Zone.js-based change detection** — prefer `OnPush` or signals; zone.js is being phased out
- **Hardcoded colors** — always daisyUI semantic tokens; never `#FF5733` or `text-blue-500`
- **`async pipe` without loading state** — users see blank screen during load; always show loading indicator
- **No error state** — errors that produce an empty screen look like empty data; show `alert-error`

## Related Workflows

- [`feature-a2ui-renderer.md`](feature-a2ui-renderer.md) — agent-driven UI in Angular
- [`design-system-compliance.md`](design-system-compliance.md) — enforcing design tokens
- [`browser-e2e-testing.md`](browser-e2e-testing.md) — E2E testing the Angular app
- [`angular-best-practices`](../../.claude/skills/angular-best-practices/SKILL.md) — performance-ranked rules for PR review
- [`web-design-guidelines`](./../.claude/skills/web-design-guidelines/SKILL.md) — run after Phase 3 to audit component against Vercel Web Interface Guidelines (third review layer after `/lint-design-system` and `accessibility-auditor`)
- [`angular-ui-patterns`](../../.claude/skills/angular-ui-patterns/SKILL.md) — loading/error/empty state doctrine
