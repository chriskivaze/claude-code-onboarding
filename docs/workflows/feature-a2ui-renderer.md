# A2UI Angular Renderer

> **When to use**: Building an agent-to-user interface renderer — Angular components that display AI-generated UI payloads from backend agents
> **Time estimate**: 4–8 hours for initial renderer; 1–2 hours per additional component type
> **Prerequisites**: Angular SPA scaffolded; backend agent generating A2UI payloads; `a2ui-angular` skill loaded

## Overview

A2UI (Agent-to-User Interface) renderer development for Angular 21.x using the `a2ui-angular` agent. Builds a recursive component catalog, agent communication service, streaming payload handler, and action dispatcher — all from agent-generated JSON payloads.

---

## Iron Law (from `skills/a2ui-angular/SKILL.md`)

> **VALIDATE ALL A2UI PAYLOADS BEFORE RENDERING — NEVER TRUST AGENT OUTPUT DIRECTLY**
> Agent-generated payloads can contain malicious component types or action payloads; validate at the renderer boundary.

---

## A2UI Payload Structure

An A2UI payload is a JSON tree that describes what the agent wants to render:

```json
{
  "type": "card",
  "props": {
    "title": "Order #12345",
    "status": "PENDING"
  },
  "children": [
    {
      "type": "button",
      "props": {
        "label": "Confirm Order",
        "action": {
          "type": "CONFIRM_ORDER",
          "payload": { "orderId": "12345" }
        }
      }
    }
  ]
}
```

---

## Phases

### Phase 1 — Load Skill and Plan Component Catalog

**Skill**: Load `a2ui-angular`
**Agent**: `a2ui-angular`

**Define component catalog** (what types your renderer supports):

| Type | Component | Description |
|------|-----------|-------------|
| `card` | `A2uiCardComponent` | Container with title and body |
| `button` | `A2uiButtonComponent` | Actionable button with label |
| `text` | `A2uiTextComponent` | Paragraph/heading text |
| `list` | `A2uiListComponent` | Ordered/unordered list |
| `form` | `A2uiFormComponent` | Dynamic form with fields |
| `image` | `A2uiImageComponent` | Image with alt text |
| `unknown` | `A2uiUnknownComponent` | Graceful fallback |

---

### Phase 2 — Scaffold with `a2ui-angular` Agent

**Dispatch `a2ui-angular` agent**:
```
Build an A2UI renderer for Angular 21.x.
Component catalog: card, button, text, list, form, image.
Include: recursive renderer, agent service, action handler, payload validation.
Security: validate component type against allowlist before rendering.
```

**Generated structure**:
```
src/app/a2ui/
├── renderer/
│   ├── a2ui-renderer.component.ts      # Recursive entry point
│   └── a2ui-renderer.component.html
├── components/
│   ├── a2ui-card/
│   ├── a2ui-button/
│   ├── a2ui-text/
│   ├── a2ui-list/
│   ├── a2ui-form/
│   └── a2ui-unknown/                   # Fallback for unknown types
├── services/
│   ├── agent.service.ts                # HTTP + SSE streaming to agent
│   └── action-handler.service.ts       # Dispatch user actions to backend
├── models/
│   ├── a2ui-node.ts                    # Payload TypeScript types
│   └── a2ui-action.ts
└── validators/
    └── payload.validator.ts            # Security validation
```

---

### Phase 3 — Recursive Renderer

```typescript
// a2ui-renderer.component.ts
import { Component, Input, inject } from '@angular/core';
import { NgComponentOutlet, NgSwitch, NgSwitchCase } from '@angular/common';
import { A2uiNode } from '../models/a2ui-node';
import { PayloadValidator } from '../validators/payload.validator';
import { A2uiCardComponent } from '../components/a2ui-card/a2ui-card.component';
import { A2uiButtonComponent } from '../components/a2ui-button/a2ui-button.component';
import { A2uiUnknownComponent } from '../components/a2ui-unknown/a2ui-unknown.component';

const COMPONENT_MAP: Record<string, Type<unknown>> = {
  card: A2uiCardComponent,
  button: A2uiButtonComponent,
  text: A2uiTextComponent,
  // ... other registered types
};

@Component({
  selector: 'app-a2ui-renderer',
  standalone: true,
  template: `
    @if (validatedNode) {
      <ng-container [ngComponentOutlet]="resolveComponent(validatedNode.type)"
                    [ngComponentOutletInputs]="{ node: validatedNode }">
      </ng-container>
    }
  `,
})
export class A2uiRendererComponent {
  private validator = inject(PayloadValidator);

  @Input() set node(value: A2uiNode) {
    // SECURITY: validate before rendering
    this.validatedNode = this.validator.validate(value) ? value : null;
  }

  validatedNode: A2uiNode | null = null;

  resolveComponent(type: string) {
    return COMPONENT_MAP[type] ?? A2uiUnknownComponent;
  }
}
```

---

### Phase 4 — Security Validation

```typescript
// payload.validator.ts
const ALLOWED_TYPES = new Set(['card', 'button', 'text', 'list', 'form', 'image']);
const ALLOWED_ACTIONS = new Set(['CONFIRM_ORDER', 'CANCEL_ORDER', 'NAVIGATE']);
const MAX_DEPTH = 10;

@Injectable({ providedIn: 'root' })
export class PayloadValidator {
  validate(node: unknown, depth = 0): boolean {
    if (depth > MAX_DEPTH) return false;
    if (!node || typeof node !== 'object') return false;

    const n = node as A2uiNode;

    // Type must be in allowlist
    if (!ALLOWED_TYPES.has(n.type)) return false;

    // Action type must be in allowlist
    if (n.props?.action && !ALLOWED_ACTIONS.has(n.props.action.type)) return false;

    // Recursively validate children
    if (n.children) {
      return n.children.every(child => this.validate(child, depth + 1));
    }

    return true;
  }
}
```

---

### Phase 5 — Streaming Agent Service

```typescript
// agent.service.ts — SSE streaming for real-time A2UI updates
@Injectable({ providedIn: 'root' })
export class AgentService {
  private http = inject(HttpClient);

  streamPayload(prompt: string): Observable<A2uiNode> {
    return new Observable(observer => {
      const eventSource = new EventSource(`/api/agent/stream?prompt=${encodeURIComponent(prompt)}`);

      eventSource.onmessage = (event) => {
        try {
          const node = JSON.parse(event.data) as A2uiNode;
          observer.next(node);
        } catch (e) {
          this.logger.error('Failed to parse A2UI payload', e);
          // Not calling observer.error() — partial failure should not kill the stream
        }
      };

      eventSource.onerror = () => {
        observer.error(new Error('Agent stream error'));
        eventSource.close();
      };

      return () => eventSource.close();
    });
  }
}
```

---

### Phase 6 — Action Handler

```typescript
// action-handler.service.ts
@Injectable({ providedIn: 'root' })
export class ActionHandlerService {
  private http = inject(HttpClient);
  private router = inject(Router);

  async dispatch(action: A2uiAction): Promise<void> {
    switch (action.type) {
      case 'CONFIRM_ORDER':
        await firstValueFrom(this.http.post(`/api/orders/${action.payload.orderId}/confirm`, {}));
        break;
      case 'NAVIGATE':
        await this.router.navigate([action.payload.path]);
        break;
      default:
        this.logger.warn('Unknown A2UI action type', action.type);
        throw new Error(`Unhandled action type: ${action.type}`);
    }
  }
}
```

---

## Quick Reference

| Phase | Action | Agent | Gate |
|-------|--------|-------|------|
| 1 — Catalog | Define supported component types | Manual | Type list defined |
| 2 — Scaffold | `a2ui-angular` agent | `a2ui-angular` | File structure created |
| 3 — Renderer | Recursive component outlet | Manual | All types render |
| 4 — Validation | Allowlist check at boundary | Manual | Unknown types → fallback |
| 5 — Streaming | SSE service | Manual | Payloads stream to renderer |
| 6 — Actions | Action dispatcher | Manual | Actions call correct endpoints |

---

## Common Pitfalls

- **No type validation** — rendering arbitrary component types from agent output is an XSS vector
- **No action allowlist** — agent could generate actions that trigger unintended operations
- **No unknown fallback** — renderer crashes on unrecognized types instead of degrading gracefully
- **No max depth guard** — deeply nested payloads can cause stack overflow in recursive renderer
- **Trusting agent output as HTML** — never use `innerHTML` with agent-generated content

## Related Workflows

- [`feature-angular-spa.md`](feature-angular-spa.md) — base Angular patterns
- [`feature-agentic-ai.md`](feature-agentic-ai.md) — the backend agent generating A2UI payloads
- [`security-audit.md`](security-audit.md) — security review of the renderer boundary
