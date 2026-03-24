# Security Hardening

> **When to use**: After a security audit surfaces findings, before first production release, or when implementing auth/crypto/input handling from scratch
> **Time estimate**: 2–8 hours depending on finding severity and count
> **Prerequisites**: Security audit completed (`/audit-security`); findings prioritized by severity

## Overview

4-phase hardening pipeline: assess findings → remediate vulnerabilities → implement controls → validate remediation. Dispatches 4 agents sequentially — `security-reviewer` (assessment), `silent-failure-hunter` (error path hardening), `threat-modeling-expert` (control design), then `security-reviewer` again (validation).

---

## Phase 1 — Assessment

**Command**: Review output from `/audit-security`, `/security-sast`, `/security-dependencies`
**Agent**: `security-reviewer` (assessment mode)

**Triage matrix** (process findings in this order):

| Priority | Criteria | Action |
|----------|---------|--------|
| P0 | CRITICAL severity OR exploitable without auth | Fix before any deploy. Stop other work. |
| P1 | HIGH severity OR exposed to internet | Fix before production deploy |
| P2 | MEDIUM severity | Fix within current sprint |
| P3 | LOW severity | Track in backlog |

**Assessment output format**:
```
## Security Assessment — [date]
P0 Findings: [N]
P1 Findings: [N]
P2 Findings: [N]
P3 Findings: [N]

### P0: [Finding title] — [file:line]
Type: [Injection / Auth / Crypto / XSS / etc.]
Exploitable via: [exact attack vector]
Fix: [specific remediation]
```

---

## Phase 2 — Remediation (by Category)

### Injection (SQL, LDAP, OS Command)

```java
// ❌ VULNERABLE — Java Spring
String query = "SELECT * FROM users WHERE email = '" + email + "'";
jdbcTemplate.query(query, ...);

// ✅ FIXED — parameterized
String query = "SELECT * FROM users WHERE email = ?";
jdbcTemplate.query(query, new Object[]{email}, ...);
```

```typescript
// ❌ VULNERABLE — NestJS / Prisma
prisma.$queryRaw(`SELECT * FROM users WHERE email = '${email}'`);

// ✅ FIXED — Prisma parameterized
prisma.$queryRaw`SELECT * FROM users WHERE email = ${email}`;
// Or use Prisma model API (always parameterized)
prisma.user.findFirst({ where: { email } });
```

```python
# ❌ VULNERABLE — Python SQLAlchemy
session.execute(f"SELECT * FROM users WHERE email = '{email}'")

# ✅ FIXED — parameterized
session.execute(text("SELECT * FROM users WHERE email = :email"), {"email": email})
```

### Authentication and Authorization

```typescript
// ❌ MISSING — no auth guard on protected route
@Get(':id')
async getOrder(@Param('id') id: string) { ... }

// ✅ FIXED — JwtAuthGuard + ownership check
@UseGuards(JwtAuthGuard)
@Get(':id')
async getOrder(@Param('id') id: string, @CurrentUser() user: User) {
  const order = await this.ordersService.findById(id);
  if (order.userId !== user.id) throw new ForbiddenException();
  return order;
}
```

**Authorization checklist**:
- [ ] Every protected route has an auth guard
- [ ] Resource ownership verified (not just authentication)
- [ ] Role-based checks explicit (not just "logged in")
- [ ] Admin endpoints explicitly restricted to admin role

### Cryptographic Failures

```python
# ❌ VULNERABLE — MD5, weak hashing
import hashlib
hashed = hashlib.md5(password.encode()).hexdigest()

# ✅ FIXED — bcrypt (cost >= 12)
import bcrypt
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
```

```typescript
// ❌ DANGEROUS — secrets in code
const JWT_SECRET = "my-secret-key";

// ✅ FIXED — environment variable
const JWT_SECRET = process.env.JWT_SECRET;
if (!JWT_SECRET) throw new Error('JWT_SECRET env var required');
```

### Secrets in Code

```bash
# Find secrets using semgrep
semgrep scan --config p/secrets .

# Common patterns to fix:
# API keys → process.env.API_KEY
# DB passwords → DATABASE_URL env var
# JWT secrets → JWT_SECRET env var
# Private keys → loaded from Secret Manager at startup
```

### Input Validation

```typescript
// NestJS — DTO validation with class-validator
export class CreateOrderDto {
  @IsString()
  @Length(1, 100)
  @Matches(/^[a-zA-Z0-9-_]+$/)
  itemId: string;

  @IsInt()
  @Min(1)
  @Max(1000)
  quantity: number;
}

// Controller — ValidationPipe enforces it
@Post()
async create(@Body() dto: CreateOrderDto) { ... }
```

```python
# FastAPI — Pydantic validation
class CreateOrderRequest(BaseModel):
    item_id: str = Field(..., min_length=1, max_length=100, pattern=r'^[a-zA-Z0-9-_]+$')
    quantity: int = Field(..., ge=1, le=1000)
```

---

## Phase 3 — Agent: `silent-failure-hunter`

After fixing vulnerabilities, dispatch `silent-failure-hunter` to verify error paths don't hide security failures:

**Dispatch**:
```
Agent: silent-failure-hunter
Focus: Error handling in auth flows, exception handling around security-critical operations
```

**What it finds**:
- `catch (e) { return []; }` — silently returning empty on auth failure
- `catch (e) { return mockUser; }` — fake data on auth error
- Missing logging on security events (failed login attempts, auth failures)
- Swallowed exceptions that prevent audit trail

**Security-specific error handling pattern**:
```typescript
// ❌ SILENT FAILURE — hides auth error
async validateToken(token: string) {
  try {
    return jwt.verify(token, secret);
  } catch (e) {
    return null;  // Caller may not check for null
  }
}

// ✅ EXPLICIT — callers know about failure
async validateToken(token: string): Promise<JwtPayload> {
  try {
    return jwt.verify(token, secret) as JwtPayload;
  } catch (e) {
    this.logger.warn('Token validation failed', { error: e.message });
    throw new UnauthorizedException('Invalid token');
  }
}
```

---

## Phase 4 — Controls Implementation

**Agent**: `threat-modeling-expert` (control design)

Controls to implement after vulnerability remediation:

**Rate limiting** (API endpoints):
```typescript
// NestJS — Throttler guard
@UseGuards(ThrottlerGuard)
@Throttle({ default: { limit: 10, ttl: 60000 } })
@Post('auth/login')
async login(@Body() dto: LoginDto) { ... }
```

**Security headers** (HTTP responses):
```typescript
// NestJS — Helmet middleware
app.use(helmet({
  contentSecurityPolicy: { directives: { defaultSrc: ["'self'"] } },
  hsts: { maxAge: 31536000, includeSubDomains: true },
}));
```

**Audit logging** (security events):
```typescript
// Log all auth events with context
this.logger.log('User authenticated', {
  userId: user.id,
  ip: req.ip,
  userAgent: req.get('user-agent'),
  timestamp: new Date().toISOString(),
});
```

**Session security**:
- JWT expiry: access token 15 min, refresh token 7 days
- Refresh token rotation on use
- Token blacklist on logout

---

## Phase 5 — Validation

Re-run the full audit pipeline after remediation:

```bash
# 1. Re-run SAST
semgrep scan --config p/owasp-top-ten --config p/secrets .

# 2. Re-run dependency scan
npm audit --audit-level=high  # or pip-audit, mvn dependency-check

# 3. Manual verification of P0/P1 findings
# For each finding that was marked "fixed":
# - Read the fixed code
# - Trace the attack vector — is it still exploitable?
# - Confirm the fix is in the right layer
```

**Dispatch `security-reviewer` agent** (second pass):
- Focus on the specific files modified during remediation
- Verify no new vulnerabilities introduced during fixing
- Confirm all P0/P1 findings have `FIXED` status

**Gate**: Zero CRITICAL and zero HIGH findings in re-scan. Every P0/P1 finding from Phase 1 has:
- [ ] Root cause documented
- [ ] Fix applied (file:line)
- [ ] Re-scan confirms resolved
- [ ] Test added to prevent regression

---

## Quick Reference

| Phase | Action | Agent | Gate |
|-------|--------|-------|------|
| 1 — Assess | Triage findings by priority | `security-reviewer` | P0/P1 list finalized |
| 2 — Remediate | Fix by category (injection, auth, crypto, secrets) | Manual | Code changed |
| 3 — Error paths | Silent failure audit | `silent-failure-hunter` | No swallowed auth exceptions |
| 4 — Controls | Rate limiting, headers, audit logging | `threat-modeling-expert` | Controls implemented |
| 5 — Validate | Re-run all scanners | `security-reviewer` | Zero CRITICAL/HIGH in re-scan |

---

## Common Pitfalls

- **Fixing symptoms, not root cause** — parameterizing one query but leaving 10 others raw; fix all instances
- **Auth fix without test** — after fixing an auth bypass, write a test that would have caught it
- **Ignoring MEDIUM** — MEDIUM findings in auth code are effectively HIGH; context matters
- **No audit trail** — security remediations must be committed with clear commit messages for compliance
- **Re-introducing during fix** — when fixing injection, don't create XSS; run SAST after every remediation batch

## Related Workflows

- [`security-audit.md`](security-audit.md) — produces the findings this workflow remediates
- [`threat-modeling.md`](threat-modeling.md) — design-time security that prevents findings from appearing
- [`test-driven-development.md`](test-driven-development.md) — write regression tests after each security fix
