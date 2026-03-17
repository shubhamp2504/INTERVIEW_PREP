# 🅰️ Angular — Real-time Scenario-Based Questions (Q1–Q10)

> **Audience**: 5–9 years experience | Senior frontend / full-stack roles  
> **Focus**: Designing scalable solutions for real-world Angular scenarios  
> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Interview Script → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q1"></a>
## Q1. Role-Based Dynamic Menu — One route accessible by multiple roles, UI renders based on logged-in role

### 📝 One-Liner
Define a menu config with `title`, `route`, and `roles[]` — filter at runtime based on the logged-in user's role to render only the routes they're allowed to access.

### 🔑 Quick Answer
**Approach**: Maintain a centralized menu config (array of `{ title, route, roles, icon? }`) — at render time, filter this array using the current user's role from `AuthService`. Use `*ngFor` + a pipe or method to dynamically show only permitted menu items. Pair this with **route guards** (`CanActivate`) so even if someone manually types a URL, the guard blocks unauthorized access. *(Menu config mein roles array rakho — login ke baad filter karo, guard se URL bhi protect karo)*

### 📖 How It Works
```
Menu Config (centralized):
  [
    { title: "Dashboard",  route: "/dashboard",  roles: ["ADMIN", "USER", "MANAGER"] },
    { title: "Users",      route: "/users",       roles: ["ADMIN"] },
    { title: "Reports",    route: "/reports",     roles: ["ADMIN", "MANAGER"] },
    { title: "Profile",    route: "/profile",     roles: ["ADMIN", "USER", "MANAGER"] },
    { title: "Settings",   route: "/settings",    roles: ["ADMIN"] }
  ]

Login → AuthService stores user role → Menu component filters config:
  User role = "MANAGER"
  Visible: Dashboard, Reports, Profile  (Settings & Users hidden)

Even if MANAGER types /users manually → RoleGuard blocks → redirect to /unauthorized
```

### 🗣️ Interview Script
"I define a centralized menu configuration — each item has a title, route path, and a roles array listing which roles can access it. A single route like Dashboard can be accessible by ADMIN, USER, and MANAGER — all three roles. On the UI side, I inject AuthService into my sidebar/nav component and filter the menu array at render time — only items whose roles array includes the current user's role are shown. For the actual route protection, I implement a `CanActivate` guard that reads the required roles from the route's `data` property and checks against the logged-in user's role. This gives me two layers of security — UI-level filtering for clean UX and guard-level enforcement to prevent URL tampering. I keep the menu config in a separate constant file or even fetch it from the backend for more dynamic setups."

### 💻 Code Example

```typescript
// ── menu.config.ts ──
export interface MenuItem {
  title: string;
  route: string;
  roles: string[];
  icon?: string;
}

export const MENU_CONFIG: MenuItem[] = [
  { title: 'Dashboard',  route: '/dashboard',  roles: ['ADMIN', 'USER', 'MANAGER'], icon: 'dashboard' },
  { title: 'Users',      route: '/users',       roles: ['ADMIN'],                    icon: 'people' },
  { title: 'Reports',    route: '/reports',     roles: ['ADMIN', 'MANAGER'],          icon: 'assessment' },
  { title: 'Profile',    route: '/profile',     roles: ['ADMIN', 'USER', 'MANAGER'], icon: 'person' },
  { title: 'Settings',   route: '/settings',    roles: ['ADMIN'],                    icon: 'settings' },
];

// ── auth.service.ts ──
@Injectable({ providedIn: 'root' })
export class AuthService {
  private currentRole$ = new BehaviorSubject<string>('');

  getRole(): string {
    return this.currentRole$.getValue();
  }

  getRoleObservable(): Observable<string> {
    return this.currentRole$.asObservable();
  }

  hasRole(allowedRoles: string[]): boolean {
    return allowedRoles.includes(this.getRole());
  }
}

// ── sidebar.component.ts ──
@Component({
  selector: 'app-sidebar',
  template: `
    <nav>
      <a *ngFor="let item of visibleMenu$ | async"
         [routerLink]="item.route"
         routerLinkActive="active">
        <mat-icon>{{ item.icon }}</mat-icon>
        {{ item.title }}
      </a>
    </nav>
  `
})
export class SidebarComponent {
  visibleMenu$: Observable<MenuItem[]>;

  constructor(private auth: AuthService) {
    this.visibleMenu$ = this.auth.getRoleObservable().pipe(
      map(role => MENU_CONFIG.filter(item => item.roles.includes(role)))
    );
  }
}

// ── role.guard.ts ──
@Injectable({ providedIn: 'root' })
export class RoleGuard implements CanActivate {
  constructor(private auth: AuthService, private router: Router) {}

  canActivate(route: ActivatedRouteSnapshot): boolean {
    const requiredRoles = route.data['roles'] as string[];
    if (this.auth.hasRole(requiredRoles)) {
      return true;
    }
    this.router.navigate(['/unauthorized']);
    return false;
  }
}

// ── app-routing.module.ts ──
const routes: Routes = [
  { path: 'dashboard', component: DashboardComponent,
    canActivate: [RoleGuard], data: { roles: ['ADMIN', 'USER', 'MANAGER'] } },
  { path: 'users', component: UsersComponent,
    canActivate: [RoleGuard], data: { roles: ['ADMIN'] } },
  { path: 'reports', component: ReportsComponent,
    canActivate: [RoleGuard], data: { roles: ['ADMIN', 'MANAGER'] } },
];
```

### ⚠️ Pitfalls
- **UI-only filtering is NOT security** — always pair with route guards + backend API authorization
- **Hardcoding roles in template** (`*ngIf="role === 'ADMIN'"`) — doesn't scale; centralize in config
- **Not handling role changes** — if user switches roles (see Q4), menu must reactively update

### 🆚 Comparison

| Approach | Pros | Cons |
|----------|------|------|
| **Static config array** | Simple, fast, no API call | Must redeploy to change menu |
| **Backend-driven menu** | Fully dynamic, admin can change | Extra API call, more complex |
| **Config + feature flags** | Middle ground, toggleable | Still needs deployment for new items |

### 🎯 Tricky Follow-ups
- *"What if the menu also needs to be ordered differently per role?"* → Add `order` field per role or use separate config per role merged at runtime
- *"What if a route is accessible but a specific button inside should be hidden for some roles?"* → Use a structural directive like `*appHasRole="['ADMIN']"` (see Q3)
- *"Can you lazy-load modules and still guard them?"* → Yes — use `canLoad` guard (blocks even the module download)

### ⚡ Remember
- **Two-layer security**: UI filter (UX) + Route guard (enforcement) + Backend check (real security)
- `canActivate` = blocks navigation | `canLoad` = blocks lazy-load download
- Keep menu config **DRY** — single source of truth for both menu rendering and routing

---

<a id="q2"></a>
## Q2. Stepper Form Across Multiple Components — Maintaining a single Reactive Form instance

### 📝 One-Liner
Use a **shared service** to hold one `FormGroup` with nested `FormGroup` per step — each step component receives its sub-group and the service orchestrates validation and submission.

### 🔑 Quick Answer
**Two approaches**: (1) **Parent component owns the form** — passes sub-groups to child step components via `@Input`. (2) **Shared FormService** — creates the entire form, each step component injects the service and accesses its slice. The service approach is better for complex wizards where steps may be dynamic or reusable. Each step component only touches its own `FormGroup` slice. Validation runs per-step (mark step invalid if its sub-group is invalid) and the final submit collects `form.value` from the service. *(Ek hi FormGroup banao service mein — har step component apna slice access kare)*

### 📖 How It Works
```
FormService (single source of truth):
  FormGroup
    ├── personal: FormGroup { firstName, lastName, email }     ← Step 1
    ├── address:  FormGroup { street, city, state, zip }       ← Step 2
    └── payment:  FormGroup { cardNumber, expiry, cvv }        ← Step 3

Step1Component → injects FormService → gets formService.form.get('personal')
Step2Component → injects FormService → gets formService.form.get('address')
Step3Component → injects FormService → gets formService.form.get('payment')

Stepper (parent) controls navigation:
  canGoNext() → current step's sub-group must be valid
  submit()    → formService.form.value has all data
```

### 🗣️ Interview Script
"For a multi-step form, I create a shared FormService that builds one top-level `FormGroup` containing nested `FormGroup`s — one per step. Each step component injects this service and binds only to its own sub-group using `formService.form.get('personal')`. This way, when Step 1 fills in the name and email, navigating to Step 2 doesn't lose the data because it all lives in the same form instance inside the service. The stepper parent component controls navigation — before allowing 'Next', it checks if the current step's sub-group is valid. On final submit, I read `formService.form.value` which gives me the complete object across all steps. I prefer the service approach over passing forms via `@Input` because it scales better — if tomorrow we add a step or rearrange them, we just update the service. For very complex wizards, I might even lazy-load step components, and they all still point to the same service."

### 💻 Code Example

```typescript
// ── stepper-form.service.ts ──
@Injectable()  // provided in stepper module or parent component
export class StepperFormService {
  form: FormGroup;

  constructor(private fb: FormBuilder) {
    this.form = this.fb.group({
      personal: this.fb.group({
        firstName: ['', [Validators.required, Validators.minLength(2)]],
        lastName:  ['', Validators.required],
        email:     ['', [Validators.required, Validators.email]],
      }),
      address: this.fb.group({
        street: ['', Validators.required],
        city:   ['', Validators.required],
        state:  ['', Validators.required],
        zip:    ['', [Validators.required, Validators.pattern(/^\d{5,6}$/)]],
      }),
      payment: this.fb.group({
        cardNumber: ['', [Validators.required, Validators.pattern(/^\d{16}$/)]],
        expiry:     ['', Validators.required],
        cvv:        ['', [Validators.required, Validators.pattern(/^\d{3}$/)]],
      }),
    });
  }

  getStepGroup(step: string): FormGroup {
    return this.form.get(step) as FormGroup;
  }

  isStepValid(step: string): boolean {
    return this.getStepGroup(step).valid;
  }

  getFormValue(): any {
    return this.form.value;
  }

  reset(): void {
    this.form.reset();
  }
}

// ── step1-personal.component.ts ──
@Component({
  selector: 'app-step1',
  template: `
    <form [formGroup]="personalForm">
      <input formControlName="firstName" placeholder="First Name">
      <input formControlName="lastName"  placeholder="Last Name">
      <input formControlName="email"     placeholder="Email">
    </form>
  `
})
export class Step1PersonalComponent implements OnInit {
  personalForm!: FormGroup;

  constructor(private stepperService: StepperFormService) {}

  ngOnInit(): void {
    this.personalForm = this.stepperService.getStepGroup('personal');
  }
}

// ── stepper.component.ts (parent orchestrator) ──
@Component({
  selector: 'app-stepper',
  template: `
    <mat-stepper [linear]="true" #stepper>
      <mat-step [stepControl]="formService.getStepGroup('personal')">
        <ng-template matStepLabel>Personal</ng-template>
        <app-step1></app-step1>
        <button mat-button matStepperNext
                [disabled]="!formService.isStepValid('personal')">Next</button>
      </mat-step>

      <mat-step [stepControl]="formService.getStepGroup('address')">
        <ng-template matStepLabel>Address</ng-template>
        <app-step2></app-step2>
        <button mat-button matStepperPrevious>Back</button>
        <button mat-button matStepperNext
                [disabled]="!formService.isStepValid('address')">Next</button>
      </mat-step>

      <mat-step [stepControl]="formService.getStepGroup('payment')">
        <ng-template matStepLabel>Payment</ng-template>
        <app-step3></app-step3>
        <button mat-button matStepperPrevious>Back</button>
        <button mat-button (click)="submit()">Submit</button>
      </mat-step>
    </mat-stepper>
  `,
  providers: [StepperFormService]  // scoped to this component tree
})
export class StepperComponent {
  constructor(public formService: StepperFormService) {}

  submit(): void {
    if (this.formService.form.valid) {
      console.log(this.formService.getFormValue());
    }
  }
}
```

### ⚠️ Pitfalls
- **Providing service at root level** — form state leaks across navigations; provide at component level or reset on `ngOnDestroy`
- **Not marking fields touched** — user clicks "Next" without filling anything, no errors show; call `markAllAsTouched()` on the step group
- **Memory leak** — if the stepper is in a route, destroy/reset the form when navigating away

### 🆚 Parent @Input vs Shared Service

| Aspect | Parent @Input | Shared Service |
|--------|--------------|----------------|
| **Complexity** | Simple for 2-3 steps | Scales to any number |
| **Reusability** | Steps tightly coupled to parent | Steps are standalone |
| **Testing** | Must test with parent | Mock service easily |
| **Dynamic steps** | Hard (add/remove @Input bindings) | Easy (service always available) |

### 🎯 Tricky Follow-ups
- *"What if steps are loaded lazily?"* → Service must be provided at a common ancestor or module level
- *"How do you save draft state?"* → Subscribe to `form.valueChanges` with `debounceTime` and save to localStorage/API
- *"What if Step 3 depends on Step 1's value?"* → Use cross-field validators or reactive `valueChanges` listeners in Step 3

### ⚡ Remember
- **One FormGroup, multiple slices** — service owns the whole form, components access their piece
- `providers: [StepperFormService]` in stepper component = scoped lifecycle
- `markAllAsTouched()` before validation check to trigger error display

---

<a id="q3"></a>
## Q3. Authentication & Authorization — Role-based guards and UI-level access with custom directives

### 📝 One-Liner
Combine **route guards** (`CanActivate`, `CanLoad`) for page-level access with a **structural directive** (`*appHasRole`) for element-level visibility — both read roles from a centralized `AuthService` backed by JWT claims.

### 🔑 Quick Answer
**Auth flow**: Login → backend returns JWT → decode token to get roles → store in `AuthService`. **Route-level**: `RoleGuard` reads `route.data['roles']` and checks against `AuthService`. **UI-level**: `*appHasRole="['ADMIN']"` directive shows/hides elements by adding/removing them from DOM (like `*ngIf`). **HTTP**: `AuthInterceptor` attaches `Bearer` token to every request. **Backend must always re-validate** — frontend guards are UX only. *(Frontend guard = UX layer; asli security backend pe honi chahiye)*

### 📖 How It Works
```
Login Flow:
  User → POST /auth/login { email, password }
       ← { accessToken: "eyJhb...", refreshToken: "..." }
       → decode JWT payload: { sub: "user123", roles: ["ADMIN", "MANAGER"], exp: ... }
       → AuthService.setUser({ id, roles, token })

Route Guard (CanActivate):
  route.data.roles = ["ADMIN"]
  AuthService.hasAnyRole(["ADMIN"]) → true → allow
  AuthService.hasAnyRole(["ADMIN"]) → false → redirect /unauthorized

Structural Directive (*appHasRole):
  <button *appHasRole="['ADMIN']">Delete User</button>
  → if user has ADMIN role → render button
  → if not → remove from DOM entirely (not just hidden)

HTTP Interceptor:
  Every outgoing request → add header: Authorization: Bearer <token>
  401 response → try refresh token → if fails → logout
```

### 🗣️ Interview Script
"My approach is three layers. First, I implement an `AuthInterceptor` that attaches the JWT token as a Bearer header on every HTTP request — and handles 401 responses by attempting a token refresh. Second, for route-level protection, I use a `RoleGuard` that implements `CanActivate`. It reads the required roles from `route.data['roles']` and checks against the current user's roles from `AuthService`. If the user doesn't have the required role, the guard redirects to an unauthorized page. Third, for element-level access — hiding buttons, sections, or links based on roles — I create a structural directive `*appHasRole`. It works like `*ngIf` but checks roles. The directive injects `AuthService`, takes an array of allowed roles as input, and uses `ViewContainerRef` to create or clear the template. This is important because just using `[hidden]` or CSS would leave the element in the DOM — a developer could inspect and interact with it. The structural directive removes it entirely. But I always emphasize — all this is UX polish. Real authorization must happen on the backend."

### 💻 Code Example

```typescript
// ── has-role.directive.ts ── (structural directive)
@Directive({ selector: '[appHasRole]' })
export class HasRoleDirective implements OnInit, OnDestroy {
  @Input('appHasRole') roles: string[] = [];

  private sub!: Subscription;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private auth: AuthService
  ) {}

  ngOnInit(): void {
    this.sub = this.auth.getRoleObservable().subscribe(role => {
      if (this.roles.includes(role)) {
        this.viewContainer.createEmbeddedView(this.templateRef);
      } else {
        this.viewContainer.clear();
      }
    });
  }

  ngOnDestroy(): void {
    this.sub.unsubscribe();
  }
}

// Usage in template:
// <button *appHasRole="['ADMIN']" (click)="deleteUser()">Delete</button>
// <div *appHasRole="['ADMIN', 'MANAGER']">Admin Panel Section</div>

// ── auth.interceptor.ts ──
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private auth: AuthService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.auth.getToken();
    const cloned = token
      ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
      : req;

    return next.handle(cloned).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          this.auth.logout();
        }
        return throwError(() => error);
      })
    );
  }
}

// ── role.guard.ts (functional guard — Angular 15+) ──
export const roleGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);
  const requiredRoles = route.data['roles'] as string[];

  if (auth.hasAnyRole(requiredRoles)) {
    return true;
  }
  return router.createUrlTree(['/unauthorized']);
};

// Route config:
{ path: 'admin', component: AdminComponent,
  canActivate: [roleGuard], data: { roles: ['ADMIN'] } }
```

### ⚠️ Pitfalls
- **Storing JWT in localStorage** — vulnerable to XSS; consider `httpOnly` cookies for sensitive apps
- **Not refreshing tokens** — user gets silently logged out; implement refresh token flow
- **Using `[hidden]` instead of structural directive** — element stays in DOM, screen readers see it, and it can be re-shown via DevTools
- **Not unsubscribing** in the directive — role changes cause duplicate view creation

### 🆚 Guard Types

| Guard | When | Use For |
|-------|------|---------|
| `CanActivate` | Before route activates | Role checks, auth checks |
| `CanLoad` | Before lazy module loads | Prevent downloading admin module for non-admins |
| `CanDeactivate` | Before leaving route | Unsaved form warning |
| `CanMatch` | Route matching (Angular 15+) | Load different modules for same URL by role |

### 🎯 Tricky Follow-ups
- *"How do you handle permission-based access, not just roles?"* → Store permissions array in JWT, directive checks `*appHasPermission="['user:delete']"`
- *"What's the difference between `CanActivate` and `CanLoad`?"* → `CanLoad` blocks even the lazy-loaded JS bundle download; `CanActivate` only blocks navigation
- *"How do you handle token expiry mid-session?"* → Interceptor catches 401, calls refresh endpoint, replays failed request

### ⚡ Remember
- **3 layers**: Interceptor (token) + Guard (route) + Directive (element) + **Backend (real security)**
- Structural directive > `[hidden]` > `*ngIf` with manual role check
- Angular 15+ prefers **functional guards** (`CanActivateFn`) over class-based

---

<a id="q4"></a>
## Q4. Role Switching Inside App — User logs in once but can switch roles or switch user

### 📝 One-Liner
Use a `BehaviorSubject<string>` for the active role in `AuthService` — switching emits a new value, and all subscribers (menu, guards, directives) reactively update without a full page reload.

### 🔑 Quick Answer
**Role switch**: User has multiple roles (e.g., Admin + Manager) from JWT claims. UI provides a role switcher dropdown. On switch, `AuthService.switchRole(newRole)` updates the `BehaviorSubject`. All components using `async` pipe or subscriptions automatically re-render. Route guard re-evaluates on next navigation. **User switch**: Requires calling backend for a new token (impersonation token) — cannot just change roles client-side because the backend must authorize the switch. *(Role switch = client-side BehaviorSubject update; User switch = backend se naya token mangna padega)*

### 📖 How It Works
```
JWT claims: { sub: "user123", roles: ["ADMIN", "MANAGER", "USER"] }

AuthService:
  availableRoles: ["ADMIN", "MANAGER", "USER"]
  activeRole$ = BehaviorSubject<string>("ADMIN")  // default to first

Role Switcher UI:
  [ADMIN ▼]  →  clicks "MANAGER"
    → authService.switchRole("MANAGER")
    → activeRole$.next("MANAGER")
    → Menu re-filters (Observable pipe)
    → *appHasRole directive re-evaluates
    → Guards use new role for next navigation

User Switch (impersonation):
  Admin clicks "Switch to User X"
    → POST /auth/impersonate { targetUserId: "user456" }
    ← { impersonationToken: "eyJ...", originalToken: "eyJ..." }
    → AuthService stores both tokens
    → "Exit impersonation" button visible to return to original
```

### 🗣️ Interview Script
"For role switching, I design the AuthService with a `BehaviorSubject` for the active role. When the user logs in, I decode the JWT to get their available roles array and set the first one as active. The UI shows a role-switcher dropdown — when they select a different role, I call `switchRole()` which pushes the new value into the BehaviorSubject. Because the menu component, the `*appHasRole` directive, and all other consumers subscribe to this observable, they reactively update without any page reload. For user switching — like an admin impersonating another user — that's a backend operation. I send a request to an impersonation endpoint, which returns a new scoped token for that user. I store both the original admin token and the impersonation token so the admin can 'exit impersonation' and return to their session. This is important for audit trails — the backend should log that admin X is acting as user Y."

### 💻 Code Example

```typescript
// ── auth.service.ts (role switching) ──
@Injectable({ providedIn: 'root' })
export class AuthService {
  private activeRole$ = new BehaviorSubject<string>('');
  private availableRoles: string[] = [];
  private originalToken: string | null = null;  // for impersonation

  setUser(decodedToken: { roles: string[] }): void {
    this.availableRoles = decodedToken.roles;
    this.activeRole$.next(this.availableRoles[0]); // default first role
  }

  getActiveRole(): Observable<string> {
    return this.activeRole$.asObservable();
  }

  getAvailableRoles(): string[] {
    return this.availableRoles;
  }

  switchRole(role: string): void {
    if (this.availableRoles.includes(role)) {
      this.activeRole$.next(role);
    }
  }

  hasRole(allowedRoles: string[]): boolean {
    return allowedRoles.includes(this.activeRole$.getValue());
  }
}

// ── role-switcher.component.ts ──
@Component({
  selector: 'app-role-switcher',
  template: `
    <mat-select [value]="activeRole$ | async" (selectionChange)="onSwitch($event.value)">
      <mat-option *ngFor="let role of roles" [value]="role">{{ role }}</mat-option>
    </mat-select>
  `
})
export class RoleSwitcherComponent {
  roles: string[];
  activeRole$: Observable<string>;

  constructor(private auth: AuthService) {
    this.roles = this.auth.getAvailableRoles();
    this.activeRole$ = this.auth.getActiveRole();
  }

  onSwitch(role: string): void {
    this.auth.switchRole(role);
  }
}

// Everything downstream reacts automatically:
// - Sidebar menu filters via auth.getActiveRole().pipe(map(...))
// - *appHasRole directive re-checks on role emission
// - Guards re-evaluate on next navigation
```

### ⚠️ Pitfalls
- **Role switch without re-checking current route** — user is on /admin, switches to USER role, should be redirected away. Subscribe to role changes and re-navigate or check.
- **Client-only role switch for API calls** — backend doesn't know which role is active; pass active role in API header or query param if backend is role-aware
- **User impersonation without audit logging** — security risk; always log impersonation on backend

### 🎯 Tricky Follow-ups
- *"What if the user is on a page they no longer have access to after switching?"* → Subscribe to role changes in a global service that re-evaluates the current route and redirects if no longer authorized
- *"Should the backend know about the role switch?"* → Yes — either pass the active role header or issue a new scoped token per role
- *"How do you handle impersonation safely?"* → Backend issues a short-lived impersonation token with the impersonator's ID in the claims for audit

### ⚡ Remember
- **BehaviorSubject** = perfect for "current value + future changes" pattern
- Role switch = client-side reactive | User switch = backend impersonation token
- Always re-check current route authorization after role switch

---

<a id="q5"></a>
## Q5. Custom Directive (Tooltip Use Case) — Text or Template tooltip using HostListener, ElementRef, Renderer2

### 📝 One-Liner
Build an attribute directive that accepts either a `string` or a `TemplateRef` — on `mouseenter`, dynamically create a positioned tooltip element (for string) or embed the template (for `TemplateRef`), and clean up on `mouseleave`.

### 🔑 Quick Answer
**Approach**: Directive input type is `string | TemplateRef<any>`. Use `HostListener('mouseenter')` to trigger tooltip show and `HostListener('mouseleave')` to hide. For string: use `Renderer2.createElement('div')`, set text content, position absolutely near the host element using `ElementRef.nativeElement.getBoundingClientRect()`. For template: use `ViewContainerRef.createEmbeddedView(templateRef)` and append it. Always use `Renderer2` instead of direct DOM manipulation for SSR compatibility and security. *(String ho to createElement se div banao; TemplateRef ho to createEmbeddedView use karo)*

### 📖 How It Works
```
Case 1 — String tooltip:
  <button appTooltip="Click to save">Save</button>
  → mouseenter → Renderer2.createElement('div') → setText("Click to save")
                → position next to button → appendChild to body
  → mouseleave → removeChild

Case 2 — Template tooltip:
  <ng-template #tipContent>
    <div class="rich-tooltip">
      <img src="info.png"> <strong>Detailed info</strong>
    </div>
  </ng-template>
  <button [appTooltip]="tipContent">Info</button>
  → mouseenter → ViewContainerRef.createEmbeddedView(tipContent)
                → position the embedded view's root element
  → mouseleave → viewContainer.clear()

Why Renderer2?
  - Direct DOM access (element.innerHTML) → XSS risk, breaks SSR
  - Renderer2 → framework-safe, works with Angular Universal, sanitized
```

### 🗣️ Interview Script
"I create an attribute directive called `appTooltip` that handles both simple text and complex template-based tooltips. The input type is `string | TemplateRef<any>` — at runtime I check `instanceof TemplateRef` to decide the rendering strategy. On `mouseenter`, if it's a string, I use `Renderer2.createElement('div')` to create a tooltip div, set its text, calculate position using `getBoundingClientRect()` of the host element from `ElementRef`, and append it to the document body using `Renderer2.appendChild`. If it's a `TemplateRef`, I use `ViewContainerRef.createEmbeddedView()` to instantiate the template and position its root element similarly. On `mouseleave`, I clean up — either `removeChild` for string or `viewContainer.clear()` for template. I use `Renderer2` throughout because directly manipulating the DOM with `innerHTML` or `document.createElement` is not safe — it's an XSS vector and breaks server-side rendering with Angular Universal."

### 💻 Code Example

```typescript
@Directive({ selector: '[appTooltip]' })
export class TooltipDirective implements OnDestroy {
  @Input('appTooltip') content: string | TemplateRef<any> = '';
  @Input() tooltipPosition: 'top' | 'bottom' | 'left' | 'right' = 'top';

  private tooltipElement: HTMLElement | null = null;
  private embeddedView: EmbeddedViewRef<any> | null = null;

  constructor(
    private el: ElementRef,
    private renderer: Renderer2,
    private viewContainer: ViewContainerRef
  ) {}

  @HostListener('mouseenter')
  onMouseEnter(): void {
    if (this.content instanceof TemplateRef) {
      this.showTemplateTooltip();
    } else if (this.content) {
      this.showTextTooltip(this.content);
    }
  }

  @HostListener('mouseleave')
  onMouseLeave(): void {
    this.hide();
  }

  private showTextTooltip(text: string): void {
    this.tooltipElement = this.renderer.createElement('div');
    this.renderer.addClass(this.tooltipElement, 'app-tooltip');

    // Use createText — NOT innerHTML (XSS-safe)
    const textNode = this.renderer.createText(text);
    this.renderer.appendChild(this.tooltipElement, textNode);

    this.setPosition(this.tooltipElement);
    this.renderer.appendChild(document.body, this.tooltipElement);
  }

  private showTemplateTooltip(): void {
    this.embeddedView = this.viewContainer.createEmbeddedView(
      this.content as TemplateRef<any>
    );
    const rootElement = this.embeddedView.rootNodes[0] as HTMLElement;
    this.renderer.addClass(rootElement, 'app-tooltip');
    this.setPosition(rootElement);
    this.renderer.appendChild(document.body, rootElement);
  }

  private setPosition(tooltip: HTMLElement): void {
    const hostRect = this.el.nativeElement.getBoundingClientRect();
    this.renderer.setStyle(tooltip, 'position', 'absolute');

    switch (this.tooltipPosition) {
      case 'top':
        this.renderer.setStyle(tooltip, 'top', `${hostRect.top - 8 + window.scrollY}px`);
        this.renderer.setStyle(tooltip, 'left', `${hostRect.left + hostRect.width / 2}px`);
        this.renderer.setStyle(tooltip, 'transform', 'translate(-50%, -100%)');
        break;
      case 'bottom':
        this.renderer.setStyle(tooltip, 'top', `${hostRect.bottom + 8 + window.scrollY}px`);
        this.renderer.setStyle(tooltip, 'left', `${hostRect.left + hostRect.width / 2}px`);
        this.renderer.setStyle(tooltip, 'transform', 'translateX(-50%)');
        break;
    }
  }

  private hide(): void {
    if (this.tooltipElement) {
      this.renderer.removeChild(document.body, this.tooltipElement);
      this.tooltipElement = null;
    }
    if (this.embeddedView) {
      const rootElement = this.embeddedView.rootNodes[0];
      this.renderer.removeChild(document.body, rootElement);
      this.viewContainer.clear();
      this.embeddedView = null;
    }
  }

  ngOnDestroy(): void {
    this.hide();
  }
}

// ── Usage ──
// Simple text:
// <button appTooltip="Save changes" tooltipPosition="top">Save</button>

// Rich template:
// <ng-template #richTip>
//   <div class="tooltip-content">
//     <strong>Pro tip:</strong> Click here to export
//   </div>
// </ng-template>
// <button [appTooltip]="richTip" tooltipPosition="bottom">Export</button>
```

### ⚠️ Pitfalls
- **Using `innerHTML`** — XSS vulnerability; always use `Renderer2.createText()` for user-provided strings
- **Not cleaning up on destroy** — tooltip stays orphaned in DOM if component is destroyed while hovering
- **Not handling scroll** — `getBoundingClientRect()` is viewport-relative; add `window.scrollY` for absolute positioning
- **Creating multiple tooltips** — if `mouseenter` fires twice (fast movements), check if tooltip already exists before creating

### 🆚 Direct DOM vs Renderer2

| Aspect | Direct DOM | Renderer2 |
|--------|-----------|-----------|
| **SSR** | Breaks (no `document`) | Works with Angular Universal |
| **Security** | `innerHTML` = XSS risk | `createText` = safe |
| **Testing** | Hard to mock DOM | Renderer2 is injectable/mockable |
| **Platform** | Browser only | Works on any platform (web worker, native) |

### ⚡ Remember
- `string | TemplateRef<any>` → check with `instanceof TemplateRef`
- **Renderer2** = `createElement`, `createText`, `appendChild`, `setStyle`, `addClass`
- Always clean up in `ngOnDestroy` and `mouseleave`

---

<a id="q6"></a>
## Q6. Multiple Router Outlets — Named outlets for rendering components dynamically

### 📝 One-Liner
Use **named `<router-outlet>`** elements alongside the primary outlet to render different components in different areas of the page — routes target a specific outlet by name.

### 🔑 Quick Answer
Angular supports one primary `<router-outlet>` and multiple **named** outlets. Routes targeting a named outlet use `{ outlet: 'outletName' }` in the route config and `[routerLink]="[{ outlets: { outletName: ['path'] } }]"` in navigation. Common use cases: side panel + main content, modal overlays, multi-pane dashboards. The URL reflects named outlets as `(outletName:path)` in the URL. *(Primary outlet default hai — named outlet se alag alag jagah pe components render kar sakte ho)*

### 📖 How It Works
```
Layout:
  ┌──────────────────────────────────────┐
  │  Header                               │
  ├─────────────────┬────────────────────┤
  │                 │                     │
  │  <router-outlet>│  <router-outlet    │
  │   (primary)     │   name="sidebar">  │
  │                 │                     │
  │  Main Content   │  Sidebar Panel     │
  │                 │                     │
  └─────────────────┴────────────────────┘

URL: /dashboard(sidebar:notifications)
  → Primary outlet renders DashboardComponent
  → "sidebar" outlet renders NotificationsComponent

Navigation:
  router.navigate([{ outlets: { primary: 'dashboard', sidebar: 'notifications' } }])
```

### 🗣️ Interview Script
"Named router outlets let me render components in different sections of the page independently. For example, in a dashboard with a main content area and a side panel, I place `<router-outlet>` for the main content and `<router-outlet name='sidebar'>` for the side panel. I define routes targeting the sidebar outlet using `{ path: 'notifications', component: NotificationsComponent, outlet: 'sidebar' }`. When a user clicks a notification icon, I navigate to `{ outlets: { sidebar: 'notifications' } }` — only the sidebar panel updates while the main content stays. The URL becomes `/dashboard(sidebar:notifications)`, which is bookmarkable. This is great for multi-pane UIs — like Gmail's email list + preview pane, or a dashboard with detail panels. For simpler cases where I just need dynamic rendering in a fixed area, I might use `*ngComponentOutlet` or a `ViewContainerRef` approach instead of named outlets."

### 💻 Code Example

```typescript
// ── app.component.html ──
// <div class="layout">
//   <div class="main-content">
//     <router-outlet></router-outlet>
//   </div>
//   <div class="sidebar">
//     <router-outlet name="sidebar"></router-outlet>
//   </div>
// </div>

// ── app-routing.module.ts ──
const routes: Routes = [
  { path: 'dashboard', component: DashboardComponent },
  { path: 'users', component: UsersComponent },

  // Named outlet routes
  { path: 'notifications', component: NotificationsComponent, outlet: 'sidebar' },
  { path: 'chat',          component: ChatComponent,          outlet: 'sidebar' },
  { path: 'help',          component: HelpComponent,          outlet: 'sidebar' },
];

// ── Navigation in template ──
// Open sidebar notifications while on dashboard:
// <a [routerLink]="[{ outlets: { primary: 'dashboard', sidebar: 'notifications' } }]">
//   Dashboard + Notifications
// </a>

// Close sidebar:
// <a [routerLink]="[{ outlets: { sidebar: null } }]">
//   Close Sidebar
// </a>

// ── Programmatic navigation ──
@Component({ /* ... */ })
export class HeaderComponent {
  constructor(private router: Router) {}

  openChat(): void {
    this.router.navigate([{ outlets: { sidebar: ['chat'] } }]);
  }

  closeSidebar(): void {
    this.router.navigate([{ outlets: { sidebar: null } }]);
  }
}
```

### ⚠️ Pitfalls
- **Ugly URLs** — `(sidebar:notifications)` in the URL can look confusing to users; consider if this is the right approach vs `*ngComponentOutlet`
- **Named outlet routes must be at the same routing level** — nested named outlets add complexity
- **Clearing an outlet** — pass `null` to remove the component; forgetting this causes stale content

### 🆚 Named Outlet vs Other Dynamic Rendering

| Approach | Use When | URL Reflected? |
|----------|----------|----------------|
| **Named `<router-outlet>`** | Multi-pane layout, bookmarkable state | Yes |
| **`*ngComponentOutlet`** | Dynamic component in a fixed slot | No |
| **`ViewContainerRef`** | Fully programmatic component creation | No |
| **`*ngIf` / `[ngSwitch]`** | Simple conditional rendering | No |

### ⚡ Remember
- `outlet: 'name'` in route config + `name="name"` on `<router-outlet>`
- Navigate: `{ outlets: { name: ['path'] } }` — close with `{ outlets: { name: null } }`
- URL format: `/primary-path(outletName:outlet-path)`

---

<a id="q7"></a>
## Q7. Facade Service — Why and how to use a facade layer in Angular

### 📝 One-Liner
A **facade service** sits between components and complex state/services — it provides a simplified, unified API that hides the complexity of multiple underlying services, stores, or API calls.

### 🔑 Quick Answer
**Problem**: A component needs data from UserService, PermissionService, NotificationService, and a Store. Without a facade, the component injects 4+ dependencies and orchestrates them itself. **Solution**: Create a `DashboardFacade` that injects all those services and exposes simple methods like `getData$()`, `refreshAll()`, `performAction()`. Components inject only the facade. Benefits: single responsibility for components (just UI), easier testing (mock one facade), loose coupling (swap backend services without touching components). *(Facade = ek simplified front door — andar 5 services hain but component ko sirf 1 inject karna padega)*

### 📖 How It Works
```
Without Facade:
  DashboardComponent
    ├── inject UserService       → getUser()
    ├── inject OrderService      → getOrders()
    ├── inject NotificationService → getNotifications()
    ├── inject AnalyticsService  → trackView()
    └── orchestrate all in ngOnInit()  😵

With Facade:
  DashboardComponent
    └── inject DashboardFacade   → getDashboardData$()  😊

  DashboardFacade (internal):
    ├── inject UserService
    ├── inject OrderService
    ├── inject NotificationService
    ├── inject AnalyticsService
    └── combineLatest() / forkJoin() → single observable
```

### 🗣️ Interview Script
"A facade service is a design pattern where I create an intermediate service layer between components and the complex backend services or state management. Instead of a component injecting UserService, OrderService, NotificationService, and AnalyticsService — then orchestrating `combineLatest` and error handling — I create a `DashboardFacade` that does all this internally and exposes one clean `getDashboardData$()` observable. The component becomes purely a UI renderer — it subscribes to one observable and binds data. This has three big advantages: first, components stay simple and testable — I mock one facade instead of five services. Second, if I refactor from REST to GraphQL or switch to NgRx, only the facade changes — components don't know. Third, it centralizes business logic orchestration — things like 'load user, then load their orders, then track analytics' don't get scattered across multiple components."

### 💻 Code Example

```typescript
// ── dashboard.facade.ts ──
export interface DashboardData {
  user: User;
  orders: Order[];
  notifications: Notification[];
}

@Injectable({ providedIn: 'root' })
export class DashboardFacade {
  constructor(
    private userService: UserService,
    private orderService: OrderService,
    private notificationService: NotificationService,
    private analyticsService: AnalyticsService
  ) {}

  // Single observable combining multiple sources
  getDashboardData$(): Observable<DashboardData> {
    return this.userService.getCurrentUser().pipe(
      switchMap(user => forkJoin({
        user: of(user),
        orders: this.orderService.getOrders(user.id),
        notifications: this.notificationService.getForUser(user.id),
      })),
      tap(() => this.analyticsService.trackPageView('dashboard'))
    );
  }

  // Action methods hide orchestration complexity
  markNotificationRead(id: string): Observable<void> {
    return this.notificationService.markRead(id).pipe(
      tap(() => this.analyticsService.trackAction('notification_read'))
    );
  }

  refreshOrders(): Observable<Order[]> {
    return this.userService.getCurrentUser().pipe(
      switchMap(user => this.orderService.getOrders(user.id))
    );
  }
}

// ── dashboard.component.ts (clean, simple) ──
@Component({
  selector: 'app-dashboard',
  template: `
    <ng-container *ngIf="data$ | async as data; else loading">
      <app-user-card [user]="data.user"></app-user-card>
      <app-order-list [orders]="data.orders"></app-order-list>
      <app-notifications [items]="data.notifications"
                         (read)="onMarkRead($event)">
      </app-notifications>
    </ng-container>
    <ng-template #loading><mat-spinner></mat-spinner></ng-template>
  `
})
export class DashboardComponent implements OnInit {
  data$!: Observable<DashboardData>;

  constructor(private facade: DashboardFacade) {}

  ngOnInit(): void {
    this.data$ = this.facade.getDashboardData$();
  }

  onMarkRead(id: string): void {
    this.facade.markNotificationRead(id).subscribe();
  }
}
```

### ⚠️ Pitfalls
- **God facade** — one facade for the entire app; keep facades scoped to a feature/page
- **Leaking implementation details** — facade should not expose `Store.dispatch()` or raw HTTP responses
- **Over-abstraction for simple pages** — if a component only needs one service, a facade is overkill

### 🆚 Facade vs Direct Service Injection

| Aspect | Direct Injection | Facade |
|--------|-----------------|--------|
| **Dependencies in component** | Many (3-5+) | One |
| **Business logic location** | Scattered in components | Centralized in facade |
| **Testing component** | Mock multiple services | Mock one facade |
| **Refactoring backend** | Touch every component | Touch only facade |
| **When to use** | Simple pages, 1-2 services | Complex pages, multiple data sources |

### ⚡ Remember
- Facade = **simplified API** over complex subsystem (Gang of Four pattern)
- Component → Facade → Services (not Component → Services)
- Keep one facade per feature module, not a global one

---

<a id="q8"></a>
## Q8. Smart and Dumb Components — Separating business logic from UI

### 📝 One-Liner
**Smart (container) components** handle data fetching, state, and logic — **dumb (presentational) components** only receive data via `@Input` and emit events via `@Output`, rendering pure UI.

### 🔑 Quick Answer
**Smart component**: Injects services, subscribes to observables, manages state, makes decisions. Has few/no `@Input`/`@Output`. Examples: `UserListPageComponent`, `DashboardComponent`. **Dumb component**: Zero service injections, all data via `@Input`, all actions via `@Output`. Reusable, testable, pure. Examples: `UserCardComponent`, `DataTableComponent`. **Rule**: Dumb components should be usable in any context — if you need to inject a service, it's no longer dumb. Use `OnPush` change detection on dumb components for performance. *(Smart = data lata hai, dumb = sirf dikhata hai)*

### 📖 How It Works
```
Smart (Container)                    Dumb (Presentational)
┌──────────────────┐                ┌──────────────────┐
│ UserListPage     │                │ UserCard          │
│                  │   @Input()     │                  │
│ - injects Service│ ───────────→  │ - [user]         │
│ - fetches users  │                │ - displays card  │
│ - handles errors │   @Output()   │                  │
│                  │ ←───────────  │ - (edit) event   │
│ - navigates      │                │ - (delete) event │
│ - updates state  │                │                  │
└──────────────────┘                └──────────────────┘
  Knows about:                       Knows about:
  - Services                         - Only its inputs
  - Router                           - Pure UI rendering
  - State management                 - Nothing about "where"
```

### 🗣️ Interview Script
"I follow the smart/dumb component pattern strictly for any non-trivial UI. Smart components — I also call them container components — are responsible for data. They inject services, subscribe to observables, handle errors, and manage routing. They don't have complex templates. Dumb components — presentational components — receive everything through `@Input` and communicate back through `@Output` events. They never inject services. This separation gives me three benefits: first, dumb components are highly reusable — a `UserCardComponent` works in a list page, a dashboard, a search result — anywhere. Second, testing is dramatically easier — dumb components just need mock inputs and verify outputs, no service mocking. Third, I set `changeDetection: OnPush` on all dumb components, which improves performance because Angular only re-renders them when their inputs change by reference. The smart component handles the subscribe/unsubscribe lifecycle, and the dumb components are pure functions of their inputs."

### 💻 Code Example

```typescript
// ── SMART: user-list-page.component.ts ──
@Component({
  selector: 'app-user-list-page',
  template: `
    <app-search-bar (search)="onSearch($event)"></app-search-bar>

    <app-user-card
      *ngFor="let user of filteredUsers$ | async"
      [user]="user"
      (edit)="onEdit($event)"
      (delete)="onDelete($event)">
    </app-user-card>

    <app-loading *ngIf="loading$ | async"></app-loading>
  `
})
export class UserListPageComponent implements OnInit {
  filteredUsers$!: Observable<User[]>;
  loading$ = new BehaviorSubject<boolean>(false);
  private searchTerm$ = new BehaviorSubject<string>('');

  constructor(
    private userService: UserService,
    private router: Router
  ) {}

  ngOnInit(): void {
    this.filteredUsers$ = this.searchTerm$.pipe(
      debounceTime(300),
      switchMap(term => {
        this.loading$.next(true);
        return this.userService.search(term);
      }),
      tap(() => this.loading$.next(false))
    );
  }

  onSearch(term: string): void {
    this.searchTerm$.next(term);
  }

  onEdit(user: User): void {
    this.router.navigate(['/users', user.id, 'edit']);
  }

  onDelete(user: User): void {
    this.userService.delete(user.id).subscribe();
  }
}

// ── DUMB: user-card.component.ts ──
@Component({
  selector: 'app-user-card',
  changeDetection: ChangeDetectionStrategy.OnPush,  // ✅ safe because pure inputs
  template: `
    <mat-card>
      <mat-card-header>
        <img mat-card-avatar [src]="user.avatar">
        <mat-card-title>{{ user.name }}</mat-card-title>
        <mat-card-subtitle>{{ user.email }}</mat-card-subtitle>
      </mat-card-header>
      <mat-card-actions>
        <button mat-button (click)="edit.emit(user)">Edit</button>
        <button mat-button color="warn" (click)="delete.emit(user)">Delete</button>
      </mat-card-actions>
    </mat-card>
  `
})
export class UserCardComponent {
  @Input() user!: User;
  @Output() edit = new EventEmitter<User>();
  @Output() delete = new EventEmitter<User>();
  // No constructor injections — ZERO services
}
```

### ⚠️ Pitfalls
- **Dumb component injecting `Router`** — that makes it aware of navigation; emit events and let smart component route
- **Passing observables as inputs** — pass the unwrapped value; the smart component should subscribe and pass data
- **Too many levels of dumb components** — prop drilling becomes painful; consider a facade or state management for deeply nested UIs

### 🆚 Quick Reference

| Property | Smart (Container) | Dumb (Presentational) |
|----------|-------------------|----------------------|
| **Injects services** | Yes | No |
| **@Input/@Output** | Few/none | Primary API |
| **Template** | Minimal (delegates to dumb) | Rich UI |
| **Change detection** | Default | OnPush |
| **Reusable** | No (page-specific) | Yes (any context) |
| **Testable** | Integration tests | Simple unit tests |

### ⚡ Remember
- Smart = **data + decisions** | Dumb = **display + events**
- `OnPush` on all dumb components → better performance
- Rule of thumb: if a component injects a service, it's smart

---

<a id="q9"></a>
## Q9. Content Projection — How `ng-content` helps build reusable components

### 📝 One-Liner
`<ng-content>` lets a parent **project** (insert) its own HTML/components into a child component's template — enabling flexible, reusable wrappers without the child knowing what content it'll receive.

### 🔑 Quick Answer
**Single-slot projection**: One `<ng-content>` in child — parent's content fills it. **Multi-slot projection**: Multiple `<ng-content select="...">` — content is distributed to specific slots based on CSS selectors. **Conditional projection**: Use `<ng-template>` + `ngTemplateOutlet` when you need to conditionally render projected content. Common use cases: cards, modals, tabs, layouts, data tables with custom cell templates. *(ng-content = parent ka HTML child ke andar daal sakte ho — jaise slot machine)*

### 📖 How It Works
```
Single Slot:
  <app-card>
    <p>This paragraph is projected into ng-content</p>
  </app-card>

  card.component.html:
    <div class="card">
      <ng-content></ng-content>     ← parent's <p> appears here
    </div>

Multi-Slot:
  <app-card>
    <h2 header>Card Title</h2>        ← select="[header]"
    <p>Card body content</p>          ← default slot
    <button footer>OK</button>        ← select="[footer]"
  </app-card>

  card.component.html:
    <div class="card">
      <div class="header">  <ng-content select="[header]"></ng-content>  </div>
      <div class="body">    <ng-content></ng-content>                     </div>
      <div class="footer">  <ng-content select="[footer]"></ng-content>  </div>
    </div>
```

### 🗣️ Interview Script
"Content projection with `ng-content` is how I build truly reusable wrapper components. Take a card component for example — I don't want to hardcode its content. Instead, I define slots using `ng-content`. For a simple card, one slot is enough — whatever the parent puts between `<app-card>` tags appears inside. For a more flexible card with header, body, and footer sections, I use multi-slot projection — `<ng-content select='[header]'>`, a default `<ng-content>`, and `<ng-content select='[footer]'>`. The parent marks its content with attributes like `header` or `footer`, and Angular projects them into the right slot. The `select` attribute supports CSS selectors — element names, classes, attributes, even complex selectors like `select='app-card-header'`. One key thing — `ng-content` doesn't create a new scope or component instance; it's literally moving the parent's DOM into the child's template at compile time. For cases where I need to conditionally render projected content — like only showing the footer if content was actually provided — I use `@ContentChild` to check if content exists."

### 💻 Code Example

```typescript
// ── card.component.ts (multi-slot) ──
@Component({
  selector: 'app-card',
  template: `
    <div class="card" [class.elevated]="elevated">
      <div class="card-header" *ngIf="hasHeader">
        <ng-content select="[card-header]"></ng-content>
      </div>

      <div class="card-body">
        <ng-content></ng-content>  <!-- default slot -->
      </div>

      <div class="card-footer" *ngIf="hasFooter">
        <ng-content select="[card-footer]"></ng-content>
      </div>
    </div>
  `
})
export class CardComponent {
  @Input() elevated = false;
  @ContentChild('cardHeader') headerContent: any;
  @ContentChild('cardFooter') footerContent: any;

  get hasHeader(): boolean { return !!this.headerContent; }
  get hasFooter(): boolean { return !!this.footerContent; }
}

// ── Usage in parent ──
// Full card with all slots:
// <app-card [elevated]="true">
//   <div card-header #cardHeader>
//     <h2>User Profile</h2>
//   </div>
//
//   <p>Main content goes in the default slot</p>
//   <app-user-details [user]="user"></app-user-details>
//
//   <div card-footer #cardFooter>
//     <button (click)="save()">Save</button>
//     <button (click)="cancel()">Cancel</button>
//   </div>
// </app-card>
//
// Minimal card (body only):
// <app-card>
//   <p>Just body content — no header or footer rendered</p>
// </app-card>

// ── Tab component (multi-slot by element selector) ──
@Component({
  selector: 'app-tabs',
  template: `
    <div class="tab-headers">
      <button *ngFor="let tab of tabs; let i = index"
              [class.active]="i === activeIndex"
              (click)="activeIndex = i">
        {{ tab.label }}
      </button>
    </div>
    <div class="tab-content">
      <ng-container *ngFor="let tab of tabs; let i = index">
        <div [hidden]="i !== activeIndex">
          <ng-container [ngTemplateOutlet]="tab.content"></ng-container>
        </div>
      </ng-container>
    </div>
  `
})
export class TabsComponent {
  @ContentChildren(TabComponent) tabs!: QueryList<TabComponent>;
  activeIndex = 0;
}

@Component({
  selector: 'app-tab',
  template: `<ng-template #content><ng-content></ng-content></ng-template>`
})
export class TabComponent {
  @Input() label = '';
  @ViewChild('content') content!: TemplateRef<any>;
}

// Usage:
// <app-tabs>
//   <app-tab label="Profile">Profile content here</app-tab>
//   <app-tab label="Settings">Settings content here</app-tab>
// </app-tabs>
```

### ⚠️ Pitfalls
- **`ng-content` is not lazy** — projected content is instantiated even if hidden with `*ngIf` on the host; use `ng-template` + `ngTemplateOutlet` for lazy projection
- **Styling projected content** — use `::ng-deep` (deprecated) or `encapsulation: ViewEncapsulation.None` carefully
- **Select conflicts** — if no `select` matches, content goes to the default (un-selected) `<ng-content>`

### 🆚 ng-content vs ngTemplateOutlet

| Feature | `ng-content` | `ngTemplateOutlet` |
|---------|-------------|-------------------|
| **Projection** | At compile time | At runtime |
| **Conditional** | Always rendered | Can conditionally render |
| **Context** | Parent's context | Can pass custom context |
| **Multiple renders** | One slot = one projection | Can render same template multiple times |
| **Use for** | Simple wrappers, cards, layouts | Data tables, dynamic lists, modals |

### ⚡ Remember
- `select` supports: `[attribute]`, `.class`, `element-name`, `:not(...)`, compound selectors
- Default `<ng-content>` (no select) = catch-all for unmatched content
- `@ContentChild` / `@ContentChildren` to query projected content in the component

---

<a id="q10"></a>
## Q10. Reusable Form Components using ControlValueAccessor

### 📝 One-Liner
`ControlValueAccessor` is the interface that bridges **custom form components** with Angular's `FormControl` — making your component work with both Reactive Forms (`formControlName`) and Template-driven Forms (`ngModel`).

### 🔑 Quick Answer
Implement the `ControlValueAccessor` interface (4 methods: `writeValue`, `registerOnChange`, `registerOnTouched`, `setDisabledState`) and register the component as `NG_VALUE_ACCESSOR` provider. After this, your custom input component can be used exactly like a native `<input>` — with `formControlName`, validators, `ngModel`, etc. Common use cases: styled input fields, date pickers, rating components, tag/chip inputs, phone number inputs. *(ControlValueAccessor = apna custom component ko Angular form ke saath kaam karne ke liye bridge hai)*

### 📖 How It Works
```
Normal <input>:
  <input formControlName="email">
  Angular knows how to read/write value via DefaultValueAccessor

Custom component:
  <app-phone-input formControlName="phone"></app-phone-input>
  Angular doesn't know how to read/write value from YOUR component
  → Implement ControlValueAccessor to teach Angular:
    writeValue(val)     → form → component (set value when form patches)
    registerOnChange(fn)  → component → form (notify form when value changes)
    registerOnTouched(fn) → component → form (notify when field is touched)
    setDisabledState(disabled) → form → component (toggle disabled)

Flow:
  FormControl.setValue("1234567890")
    → writeValue("1234567890") called on your component
    → you update your internal model

  User types in your component
    → you call onChange("9876543210")
    → FormControl value updates
    → Validators run
    → form.valid updates
```

### 🗣️ Interview Script
"`ControlValueAccessor` is the interface that makes a custom component compatible with Angular forms. It's the bridge between the form API and the component's internal state. When I build a reusable phone input or a rating stars component, I implement four methods: `writeValue` which Angular calls when the form sets a value programmatically — here I update my internal model. `registerOnChange` gives me a callback function that I call whenever the user changes the value in my component — this propagates the new value back to the `FormControl`. `registerOnTouched` is similar but for the blur event — I call it when the user interacts and leaves the component. And `setDisabledState` lets me toggle disabled styling. I register the component as a `NG_VALUE_ACCESSOR` multi-provider using `forwardRef` because the class isn't defined yet at provider registration time. After this, `<app-phone-input formControlName='phone'>` works exactly like a native input — validators, value access, touched state — everything just works."

### 💻 Code Example

```typescript
// ── phone-input.component.ts ──
@Component({
  selector: 'app-phone-input',
  template: `
    <div class="phone-input" [class.disabled]="disabled" [class.error]="showError">
      <select [(ngModel)]="countryCode" (ngModelChange)="emitValue()" [disabled]="disabled">
        <option value="+1">🇺🇸 +1</option>
        <option value="+91">🇮🇳 +91</option>
        <option value="+44">🇬🇧 +44</option>
      </select>
      <input type="tel"
             [(ngModel)]="phoneNumber"
             (ngModelChange)="emitValue()"
             (blur)="onTouched()"
             [disabled]="disabled"
             placeholder="Phone number"
             maxlength="10">
    </div>
  `,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => PhoneInputComponent),
      multi: true,
    }
  ]
})
export class PhoneInputComponent implements ControlValueAccessor {
  countryCode = '+91';
  phoneNumber = '';
  disabled = false;
  showError = false;

  // Callbacks registered by Angular
  private onChange: (value: string) => void = () => {};
  onTouched: () => void = () => {};

  // Form → Component: Angular calls this when form value changes
  writeValue(value: string): void {
    if (value) {
      // Parse "+91-9876543210" → countryCode="+91", phoneNumber="9876543210"
      const parts = value.split('-');
      this.countryCode = parts[0] || '+91';
      this.phoneNumber = parts[1] || '';
    }
  }

  // Store the callback Angular gives us
  registerOnChange(fn: (value: string) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }

  // Component → Form: call onChange to propagate value
  emitValue(): void {
    const fullNumber = `${this.countryCode}-${this.phoneNumber}`;
    this.onChange(fullNumber);
  }
}

// ── Usage in parent (Reactive Form) ──
// form = this.fb.group({
//   name:  ['', Validators.required],
//   phone: ['+91-', [Validators.required, phoneValidator]],
// });
//
// <form [formGroup]="form">
//   <input formControlName="name" placeholder="Name">
//   <app-phone-input formControlName="phone"></app-phone-input>
//   <div *ngIf="form.get('phone')?.errors?.['invalidPhone']">
//     Invalid phone number
//   </div>
// </form>

// ── Custom validator for phone ──
export function phoneValidator(control: AbstractControl): ValidationErrors | null {
  const value = control.value as string;
  if (!value) return null;
  const phone = value.split('-')[1] || '';
  return /^\d{10}$/.test(phone) ? null : { invalidPhone: true };
}

// ── Another example: Star Rating ──
@Component({
  selector: 'app-star-rating',
  template: `
    <span *ngFor="let star of stars; let i = index"
          (click)="!disabled && rate(i + 1)"
          (mouseenter)="hover = i + 1"
          (mouseleave)="hover = 0"
          [class.filled]="(hover || value) >= i + 1"
          class="star">★</span>
  `,
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => StarRatingComponent),
    multi: true,
  }]
})
export class StarRatingComponent implements ControlValueAccessor {
  stars = [1, 2, 3, 4, 5];
  value = 0;
  hover = 0;
  disabled = false;

  private onChange: (val: number) => void = () => {};
  private onTouched: () => void = () => {};

  writeValue(val: number): void { this.value = val || 0; }
  registerOnChange(fn: any): void { this.onChange = fn; }
  registerOnTouched(fn: any): void { this.onTouched = fn; }
  setDisabledState(d: boolean): void { this.disabled = d; }

  rate(val: number): void {
    this.value = val;
    this.onChange(val);
    this.onTouched();
  }
}
// Usage: <app-star-rating formControlName="rating"></app-star-rating>
```

### ⚠️ Pitfalls
- **Forgetting `multi: true`** — this is a multi-provider token; without it, you override the default accessor
- **Forgetting `forwardRef`** — the class isn't yet defined when the decorator runs; `forwardRef` defers the reference
- **Not calling `onTouched()`** — form won't track touched/untouched state; validators like "required on touch" won't trigger
- **Not implementing `setDisabledState`** — `formControl.disable()` won't visually disable your component

### 🎯 Tricky Follow-ups
- *"Can a CVA component have its own validators?"* → Yes — also provide `NG_VALIDATORS` and implement `Validator` interface
- *"How is this different from using @Input/@Output?"* → CVA integrates with the form API (validators, dirty/touched, disable) — @Input/@Output doesn't
- *"What about `Standalone` components with CVA?"* → Same pattern, just use `standalone: true` and import `FormsModule`/`ReactiveFormsModule` in `imports`

### 🆚 @Input/@Output vs ControlValueAccessor

| Aspect | @Input/@Output | ControlValueAccessor |
|--------|---------------|---------------------|
| **Form integration** | Manual wiring | Automatic (formControlName works) |
| **Validation** | Manual | Angular validators just work |
| **Touched/Dirty** | Must track manually | Auto-tracked by form API |
| **Disable state** | Must pass @Input | `formControl.disable()` works |
| **Use when** | Display data, trigger actions | Custom form inputs |

### ⚡ Remember
- **4 methods**: `writeValue` (in), `registerOnChange` (out), `registerOnTouched` (blur), `setDisabledState` (toggle)
- Provider: `{ provide: NG_VALUE_ACCESSOR, useExisting: forwardRef(() => MyComponent), multi: true }`
- After implementing CVA → `formControlName`, `ngModel`, validators all work automatically
