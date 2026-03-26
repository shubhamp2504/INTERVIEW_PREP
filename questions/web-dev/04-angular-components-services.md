# 🅰️ Angular — Components & Services Fundamentals (Q11–Q20)

> **Audience**: All levels | Interview essentials for Angular roles
> **Focus**: Component structure, services, DI, lifecycle, communication, testing
> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Interview Script → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q11"></a>
## Q11. What is an Angular Component?

### 📝 One-Liner
A component is the **fundamental building block** of an Angular app — it controls a portion of the UI by combining an HTML template (view), a TypeScript class (logic), and CSS (styles).

### 🔑 Quick Answer
Every Angular app is a tree of components. A component is a class decorated with `@Component()` that has: (1) `selector` — custom HTML tag name, (2) `templateUrl` / `template` — the HTML view, (3) `styleUrls` / `styles` — scoped CSS. The class contains properties and methods that drive the template via data binding. Angular apps are built by composing small, reusable components. *(Angular mein har UI piece ek component hai — HTML + TypeScript + CSS ka combo — app components ka tree hai)*

### 📖 How It Works
```
@Component({
  selector: 'app-user-card',       ← Used as <app-user-card> in HTML
  templateUrl: './user-card.html', ← View (HTML)
  styleUrls: ['./user-card.css']   ← Scoped styles
})
export class UserCardComponent {   ← Logic (TS class)
  @Input() user: User;             ← Data from parent
  showDetails = false;             ← Local state

  toggleDetails() {                ← Behavior
    this.showDetails = !this.showDetails;
  }
}

Component Tree:
  AppComponent
  ├── HeaderComponent
  ├── SidebarComponent
  └── MainComponent
      ├── UserListComponent
      │   └── UserCardComponent (reused per user)
      └── FooterComponent
```

### 🗣️ Interview Script
"An Angular component is the basic building block of the UI. It's a TypeScript class decorated with `@Component` that binds together three things — an HTML template for the view, a class for the logic, and CSS for styling. Each component has a selector that turns it into a custom HTML tag. For example, I create a `UserCardComponent` with selector `app-user-card` — then I can use `<app-user-card>` anywhere in my templates. Angular apps are essentially a tree of components — the root `AppComponent` contains child components like Header, Sidebar, and Main, which themselves contain further children. This composition pattern makes the app modular, testable, and reusable."

### 💻 Code
```typescript
// ── user-card.component.ts ──
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  template: `
    <div class="card">
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
      <button (click)="toggleDetails()">
        {{ showDetails ? 'Hide' : 'Show' }} Details
      </button>
      <div *ngIf="showDetails">
        <p>Role: {{ user.role }}</p>
        <p>Joined: {{ user.joinDate | date:'mediumDate' }}</p>
      </div>
    </div>
  `,
  styles: [`
    .card { border: 1px solid #ddd; padding: 16px; border-radius: 8px; }
    h3 { margin: 0 0 8px; }
  `]
})
export class UserCardComponent {
  @Input() user!: { name: string; email: string; role: string; joinDate: Date };
  showDetails = false;

  toggleDetails(): void {
    this.showDetails = !this.showDetails;
  }
}
```

### ⚡ Remember
> Component = **@Component** decorator + HTML template + TS class + CSS | `selector` = custom tag | App = component tree | Reusable & composable

### 🔗 Follow-ups
- Standalone components (Angular 14+)
- View encapsulation modes (Emulated, None, ShadowDom)
- Dynamic component loading with `ViewContainerRef`

---

<a id="q12"></a>
## Q12. What is the role of a Service in Angular?

### 📝 One-Liner
A Service is a **plain TypeScript class** (decorated with `@Injectable`) that encapsulates business logic, API calls, and shared state — keeping components lean and focused on the UI.

### 🔑 Quick Answer
Services handle: (1) **HTTP calls** — `HttpClient` GET/POST/PUT/DELETE to backend APIs. (2) **Business logic** — data transformation, validation, calculations. (3) **Shared state** — data that multiple components need (like logged-in user). (4) **Cross-cutting concerns** — logging, error handling, caching. Angular's DI system injects services into components via the constructor. *(Service mein saari heavy lifting hoti hai — API calls, logic, shared data — component sirf UI handle karta hai)*

### 📖 How It Works
```
Without Services:                With Services:
┌──────────────┐               ┌──────────────┐
│ Component A  │               │ Component A  │──┐
│ - API call   │               │ - UI only    │  │
│ - Logic      │               └──────────────┘  │
│ - UI         │               ┌──────────────┐  ├── UserService
└──────────────┘               │ Component B  │──┘   (shared)
┌──────────────┐               │ - UI only    │       │
│ Component B  │               └──────────────┘       ↓
│ - SAME API   │ ← Duplicate!                    ┌─────────┐
│ - SAME Logic │                                  │ Backend │
│ - UI         │                                  │  API    │
└──────────────┘                                  └─────────┘
```

### 🗣️ Interview Script
"A service in Angular is a class decorated with `@Injectable` that holds business logic and API communication. The key principle is separation of concerns — components should only handle the view and user interaction, while services handle everything else. For example, a `UserService` would contain methods like `getUsers()`, `createUser()`, `deleteUser()` — all using `HttpClient`. Multiple components can inject the same service and share data. If the service is `providedIn: 'root'`, Angular creates a single instance for the whole app — this singleton pattern is perfect for shared state like authentication or a shopping cart."

### 💻 Code
```typescript
// ── user.service.ts ──
@Injectable({ providedIn: 'root' })  // Singleton for entire app
export class UserService {
  private apiUrl = '/api/users';

  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }

  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`);
  }

  createUser(user: User): Observable<User> {
    return this.http.post<User>(this.apiUrl, user);
  }

  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}

// ── user-list.component.ts (uses service) ──
@Component({ /* ... */ })
export class UserListComponent implements OnInit {
  users: User[] = [];

  constructor(private userService: UserService) {}  // DI injects service

  ngOnInit(): void {
    this.userService.getUsers().subscribe(users => this.users = users);
  }
}
```

### ⚡ Remember
> Service = `@Injectable` class | Holds API calls + logic + shared state | Components stay lean (UI only) | DI injects automatically | `providedIn: 'root'` = singleton

### 🔗 Follow-ups
- RxJS operators in services (switchMap, catchError)
- Service with BehaviorSubject for state management
- Interceptors for cross-cutting HTTP concerns

---

<a id="q13"></a>
## Q13. Why should we not call APIs directly in components?

### 📝 One-Liner
Calling APIs in components creates **tight coupling**, **code duplication**, and makes **testing difficult** — services provide a single, reusable, mockable layer for all data operations.

### 🔑 Quick Answer
**Problems with API calls in components**: (1) **Duplication** — same endpoint called in multiple components = copy-paste code. (2) **Hard to test** — need to mock HttpClient in every component test. (3) **Tight coupling** — component knows about URLs, headers, error handling. (4) **Hard to maintain** — if API URL changes, update every component. **Solution**: Extract all API calls to services — single source of truth, easy to mock, easy to update. *(Component mein seedha API call karne se code repeat hota hai, test karna mushkil hota hai — service mein rakho, ek jagah change karo)*

### 📖 How It Works
```
❌ BAD: API in component
  ComponentA → http.get('/api/users')     ← Duplicated
  ComponentB → http.get('/api/users')     ← Same URL
  ComponentC → http.get('/api/users')     ← Hard to maintain
  If URL changes → update 3 places!

✅ GOOD: API in service
  ComponentA → UserService.getUsers()     ← Single call
  ComponentB → UserService.getUsers()     ← Reuses service
  ComponentC → UserService.getUsers()     ← Same method
  If URL changes → update 1 place (service)!

Testing:
  ❌ BAD:  TestBed needs HttpClientTestingModule in EVERY component test
  ✅ GOOD: Component test just mocks UserService → simple, fast
```

### 🗣️ Interview Script
"Calling APIs directly in components violates the Single Responsibility Principle. The component's job is to manage the UI — displaying data, handling user interactions, template rendering. When I put HTTP calls in a component, I'm mixing concerns. If three components need the same user data, I'd have three copies of the same `http.get()` call with the same URL, headers, and error handling. If the API changes — say the URL moves from v1 to v2 — I have to update every component. With a service, I write the API call once, inject the service everywhere, and all components share the same implementation. Testing also becomes much simpler — in component tests, I just provide a mock service instead of setting up HttpClientTestingModule."

### 🆚 vs.
| Aspect | API in Component | API in Service |
|--------|-----------------|----------------|
| Reusability | ❌ Copy-paste | ✅ Inject & use |
| Testability | Hard (mock HTTP) | Easy (mock service) |
| Maintainability | Update N places | Update 1 place |
| SRP | ❌ Violated | ✅ Respected |
| Caching | Per-component | Centralized |

### ⚡ Remember
> **Never HTTP in component** | Service = single source for API calls | Mock service in tests, not HttpClient | SRP: component = UI, service = data

### 🔗 Follow-ups
- Facade pattern for complex service orchestration
- NgRx for state management beyond services
- HTTP interceptors for auth tokens / error handling

---

<a id="q14"></a>
## Q14. How do components communicate with each other?

### 📝 One-Liner
**Parent → Child** uses `@Input()` to pass data down; **Child → Parent** uses `@Output()` + `EventEmitter` to emit events up; **Unrelated components** communicate via a shared service with `Subject` or `BehaviorSubject`.

### 🔑 Quick Answer
Four patterns: (1) **@Input()** — parent binds data to child's input property. (2) **@Output() + EventEmitter** — child emits events that parent listens to with `(event)` binding. (3) **Shared Service** — inject same service in both components, use RxJS `Subject`/`BehaviorSubject` for pub-sub. (4) **ViewChild** — parent accesses child's public methods/properties directly. *(Parent se child mein data @Input se jaata hai, child se parent mein event @Output se jaata hai, unrelated components ke liye shared service use karo)*

### 📖 How It Works
```
1. Parent → Child (@Input):
   <app-child [user]="selectedUser"></app-child>
   ChildComponent { @Input() user: User; }

2. Child → Parent (@Output + EventEmitter):
   ChildComponent {
     @Output() deleted = new EventEmitter<number>();
     onDelete(id) { this.deleted.emit(id); }
   }
   <app-child (deleted)="handleDelete($event)"></app-child>

3. Shared Service (Unrelated components):
   Service { private msg$ = new BehaviorSubject<string>(''); }
   ComponentA → service.msg$.next('hello')  // publish
   ComponentB → service.msg$.subscribe()     // listen

4. @ViewChild (Parent access child directly):
   @ViewChild(ChildComponent) child!: ChildComponent;
   this.child.refresh();  // call child's method
```

### 🗣️ Interview Script
"There are four main patterns. For parent-to-child, I use `@Input()` — the parent component binds a value in its template, and the child receives it as an input property. For child-to-parent, I use `@Output()` with `EventEmitter` — the child emits an event when something happens, like a button click, and the parent listens with event binding syntax using parentheses. For sibling or completely unrelated components, I use a shared service with a `BehaviorSubject` — one component pushes data into the subject, and any subscribing component receives updates reactively. The fourth option is `@ViewChild` where the parent gets a direct reference to the child component instance — but I prefer the reactive approach with services for loose coupling."

### 💻 Code
```typescript
// ── Pattern 1: @Input (Parent → Child) ──
// parent.component.html
// <app-user-card [user]="selectedUser"></app-user-card>

@Component({ selector: 'app-user-card', template: `<h3>{{ user.name }}</h3>` })
export class UserCardComponent {
  @Input() user!: User;
}

// ── Pattern 2: @Output (Child → Parent) ──
@Component({
  selector: 'app-user-card',
  template: `<button (click)="onDelete()">Delete</button>`
})
export class UserCardComponent {
  @Input() user!: User;
  @Output() deleted = new EventEmitter<number>();

  onDelete(): void {
    this.deleted.emit(this.user.id);
  }
}
// parent.component.html
// <app-user-card [user]="u" (deleted)="removeUser($event)"></app-user-card>

// ── Pattern 3: Shared Service (Unrelated components) ──
@Injectable({ providedIn: 'root' })
export class NotificationService {
  private messageSource = new BehaviorSubject<string>('');
  message$ = this.messageSource.asObservable();

  sendMessage(msg: string): void {
    this.messageSource.next(msg);
  }
}

// Component A (sender):
this.notificationService.sendMessage('User deleted!');

// Component B (receiver):
ngOnInit() {
  this.notificationService.message$.subscribe(msg => this.notification = msg);
}
```

### 🆚 vs.
| Pattern | Direction | Coupling | Use Case |
|---------|-----------|----------|----------|
| @Input | Parent → Child | Tight | Direct data passing |
| @Output | Child → Parent | Moderate | Events / actions |
| Shared Service | Any ↔ Any | Loose | Unrelated components |
| @ViewChild | Parent → Child | Tight | Call child methods |

### ⚡ Remember
> **@Input** = data down | **@Output** = events up | **Shared Service** = any direction | BehaviorSubject for latest value | Unsubscribe to prevent memory leaks

### 🔗 Follow-ups
- Content projection with `<ng-content>` for template composition
- NgRx Store for global state management
- Signals (Angular 16+) as reactive primitives

---

<a id="q15"></a>
## Q15. What is Dependency Injection in Angular?

### 📝 One-Liner
Dependency Injection (DI) is a design pattern where Angular **automatically provides** (injects) class dependencies — typically services — into components or other services via the constructor, instead of manually creating them.

### 🔑 Quick Answer
**How it works**: (1) Mark a class as injectable with `@Injectable()`. (2) Register it with a provider (usually `providedIn: 'root'`). (3) Declare it as a constructor parameter in the consuming class. (4) Angular's **Injector** creates and supplies the instance automatically. Benefits: loose coupling, easy testing (swap real service with mock), single instance management. *(DI mein Angular khud service ka object banata hai aur constructor mein de deta hai — tumhe manually `new` nahi karna padta)*

### 📖 How It Works
```
Without DI (manual instantiation):
  class UserComponent {
    service = new UserService(new HttpClient(...));  ← Tightly coupled!
  }

With DI (Angular injects):
  class UserComponent {
    constructor(private userService: UserService) {} ← Angular provides it!
  }

Angular's DI System:
  1. @Injectable({ providedIn: 'root' })  → registers UserService
  2. Angular creates Injector at bootstrap
  3. Component declares constructor(private userService: UserService)
  4. Injector looks up UserService → creates if needed → injects

Injector Hierarchy:
  Root Injector (app-wide singletons)
  └── Module Injector (lazy-loaded modules)
      └── Element Injector (component-level)
          Each level can provide/override services
```

### 🗣️ Interview Script
"Dependency Injection in Angular is a core pattern where the framework manages object creation and lifecycle. Instead of a component creating its own dependencies with `new UserService()`, it simply declares what it needs in its constructor — `constructor(private userService: UserService)`. Angular's injector resolves this dependency, creates an instance if one doesn't exist, and passes it in. The service is registered with `@Injectable({ providedIn: 'root' })` which tells Angular to create a single app-wide instance. This gives us loose coupling — the component doesn't know how the service is created — and easy testing — I can provide a mock service in tests. Angular has a hierarchical injector system: root injector, module injector, and element injector — each level can provide or override services for different scopes."

### 💻 Code
```typescript
// ── 1. Define injectable service ──
@Injectable({ providedIn: 'root' })  // Registered in root injector
export class AuthService {
  private loggedIn = false;

  login(credentials: { email: string; password: string }): Observable<boolean> {
    return this.http.post<boolean>('/api/login', credentials);
  }

  isLoggedIn(): boolean { return this.loggedIn; }

  constructor(private http: HttpClient) {}  // HttpClient itself is injected!
}

// ── 2. Inject into component ──
@Component({ /* ... */ })
export class LoginComponent {
  constructor(private authService: AuthService) {}  // Angular injects

  onSubmit(form: LoginForm): void {
    this.authService.login(form).subscribe(/* ... */);
  }
}

// ── 3. Test with mock ──
describe('LoginComponent', () => {
  let mockAuthService: jasmine.SpyObj<AuthService>;

  beforeEach(() => {
    mockAuthService = jasmine.createSpyObj('AuthService', ['login']);
    TestBed.configureTestingModule({
      providers: [{ provide: AuthService, useValue: mockAuthService }]  // Mock!
    });
  });
});
```

### 🎯 Tricky Follow-ups
- **Q**: "What if two services depend on each other?" → Circular dependency — Angular throws error. Fix: use `forwardRef()` or refactor into a third mediator service.
- **Q**: "Can you inject a service into another service?" → Yes — as long as both are `@Injectable()`. Services can have constructor dependencies just like components.

### ⚡ Remember
> DI = Angular manages object creation | `@Injectable` + constructor param = auto-injection | Root = singleton | Component-level = new instance per component | Testability via mock providers

### 🔗 Follow-ups
- `@Inject` token for non-class dependencies
- `InjectionToken` for interface-based injection
- Multi-providers with `multi: true`

---

<a id="q16"></a>
## Q16. What is the difference between a Component and a Service?

### 📝 One-Liner
A **Component** controls a piece of the UI (template + logic + styles), while a **Service** holds reusable business logic, data access, and shared state — separation of concerns is the core principle.

### 🔑 Quick Answer
**Component**: decorated with `@Component`, has HTML template + CSS, handles user interaction, displays data, lifecycle hooks tied to the DOM. **Service**: decorated with `@Injectable`, no template, contains API calls, business logic, shared state, utility functions. Components are consumers — services are providers. *(Component UI ka kaam karta hai — dikhana, click handle karna. Service data ka kaam karta hai — API call, logic, shared state)*

### 📖 How It Works
```
Component:                         Service:
┌──────────────────────┐          ┌─────────────────┐
│ @Component           │          │ @Injectable     │
│ ├── template (HTML)  │          │ ├── API calls   │
│ ├── styles (CSS)     │          │ ├── logic       │
│ ├── selector (tag)   │          │ ├── shared data │
│ ├── lifecycle hooks  │          │ └── utilities   │
│ └── user interaction │          └─────────────────┘
└──────────────────────┘
     Uses services ──────────────────→
```

### 🗣️ Interview Script
"The key difference is responsibility. A component is responsible for the UI — it has an HTML template, CSS styles, and a TypeScript class that handles user interactions and data binding. It's decorated with `@Component` and has lifecycle hooks like `ngOnInit` and `ngOnDestroy` that are tied to the DOM. A service is responsible for business logic — it's decorated with `@Injectable`, has no template, and contains API calls, data transformation, and shared state. The separation follows the Single Responsibility Principle: the component asks 'what to display and how to interact', while the service answers 'where to get data and how to process it'. A component consumes services via dependency injection."

### 🆚 vs.
| Aspect | Component | Service |
|--------|-----------|---------|
| Decorator | `@Component` | `@Injectable` |
| Template | ✅ HTML + CSS | ❌ None |
| Selector | ✅ Custom HTML tag | ❌ None |
| Lifecycle | DOM-based (ngOnInit, ngOnDestroy) | No lifecycle hooks |
| Purpose | UI rendering + interaction | Logic + data + state |
| Instantiation | Per DOM element | Per injector scope |
| Testing | ComponentFixture + DOM | Plain class + mock deps |
| Reusability | Via template composition | Via DI in any class |

### ⚡ Remember
> Component = **UI** (@Component, template, styles) | Service = **Logic** (@Injectable, API, state) | SRP: components show, services do | Inject service into component, never the reverse

### 🔗 Follow-ups
- Pipes as another form of reusable logic
- Directives vs Components (structural + attribute)
- Smart vs Dumb component architecture

---

<a id="q17"></a>
## Q17. What happens if a service is provided in root? (Singleton behavior)

### 📝 One-Liner
`providedIn: 'root'` registers the service in Angular's **root injector** — creating a **single shared instance** (singleton) for the entire application, available everywhere without importing a module.

### 🔑 Quick Answer
When `@Injectable({ providedIn: 'root' })` is set: (1) Angular registers the service in the root injector at app startup. (2) Only **one instance** is created for the whole app. (3) Every component/service that injects it gets the **same object**. (4) The service is **tree-shakeable** — if no one injects it, it's removed from the bundle. This is Angular's recommended way to create app-wide singletons. *(Root mein provide karne se poore app mein ek hi instance banta hai — singleton pattern automatic mil jaata hai)*

### 📖 How It Works
```
@Injectable({ providedIn: 'root' })
export class CartService {
  items: Product[] = [];
  addItem(p: Product) { this.items.push(p); }
}

Component A adds item → CartService.items = [iPhone]
Component B reads     → CartService.items = [iPhone]  ← Same instance!
Component C adds item → CartService.items = [iPhone, MacBook]
All components see    → [iPhone, MacBook]  ← Shared state!

Why it works:
  Root Injector (created at bootstrap)
  ├── CartService  (single instance)
  ├── AuthService  (single instance)
  └── ...
  Every component resolves from this injector → same instance
```

### 🗣️ Interview Script
"When I set `providedIn: 'root'`, Angular registers the service in the application's root injector. This means there's exactly one instance created when the service is first injected anywhere in the app. Every component, directive, or other service that requests it via constructor injection gets the same instance — it's a singleton. This is ideal for services like authentication, shopping cart, or application settings where you need shared state across the entire app. An added benefit is tree-shaking — if no component actually injects the service, Angular's build process removes it from the final bundle, reducing app size."

### 💻 Code
```typescript
// ── Singleton Service ──
@Injectable({ providedIn: 'root' })  // One instance for entire app
export class AuthService {
  private currentUser: User | null = null;

  login(user: User): void { this.currentUser = user; }
  getUser(): User | null { return this.currentUser; }
  isLoggedIn(): boolean { return this.currentUser !== null; }
}

// HeaderComponent injects AuthService → gets instance #1
// ProfileComponent injects AuthService → gets SAME instance #1
// GuardService injects AuthService → gets SAME instance #1

// All three see the same currentUser value = Singleton!
```

### 🎯 Tricky Follow-ups
- **Q**: "Is `providedIn: 'root'` the same as adding to `AppModule.providers`?" → Functionally similar, but `providedIn: 'root'` is tree-shakeable and recommended. `providers: []` always includes the service regardless of usage.
- **Q**: "What about lazy-loaded modules?" → `providedIn: 'root'` still creates a single app-wide instance. But if you add the service to a lazy module's `providers`, that module gets its own instance.

### ⚡ Remember
> `providedIn: 'root'` = **singleton** for entire app | Tree-shakeable | Shared state across all components | Recommended approach since Angular 6+

### 🔗 Follow-ups
- `providedIn: 'any'` — instance per lazy module
- `providedIn: 'platform'` — shared across multiple apps
- NgModule providers vs `providedIn` comparison

---

<a id="q18"></a>
## Q18. When will multiple instances of a service be created?

### 📝 One-Liner
Multiple instances are created when a service is provided at the **component level** using the `providers` array in `@Component` — each component instance gets its **own separate service instance**.

### 🔑 Quick Answer
**Component-level provider**: Add `providers: [MyService]` in `@Component` decorator → every time Angular creates that component, it also creates a new `MyService` instance. The instance is scoped to that component and its children. **Use case**: when each component needs its own isolated state — e.g., a form service, a local filter, or a per-tab data context. *(Agar component ke `providers` mein service daal do, toh har component ka apna alag instance banega — shared nahi hoga)*

### 📖 How It Works
```
Root-level (Singleton):
  @Injectable({ providedIn: 'root' })
  → 1 instance for entire app

Component-level (Multiple instances):
  @Component({
    providers: [CounterService]  ← Each component gets its own!
  })

Example:
  <app-counter>  →  CounterService instance #1 (count = 5)
  <app-counter>  →  CounterService instance #2 (count = 3)
  <app-counter>  →  CounterService instance #3 (count = 0)
  Each counter is independent!

Injector hierarchy:
  Root Injector ← providedIn: 'root' (singleton)
  └── AppComponent
      ├── CounterComponent #1 ← providers: [CounterService] → instance #1
      ├── CounterComponent #2 ← providers: [CounterService] → instance #2
      └── CounterComponent #3 ← providers: [CounterService] → instance #3
```

### 🗣️ Interview Script
"If I need each component to have its own service instance, I add the service to the component's `providers` array instead of using `providedIn: 'root'`. For example, if I have a counter component and I want each counter on the page to maintain its own count independently, I provide `CounterService` at the component level. Every time Angular creates a new counter component, it also creates a fresh `CounterService` instance. The lifecycle of that service is tied to the component — when the component is destroyed, the service instance is garbage collected. This is useful for form state, local filters, or any component that needs isolated data context."

### 💻 Code
```typescript
// ── Service ──
@Injectable()  // No providedIn — not a singleton!
export class FormStateService {
  private formData: Record<string, unknown> = {};

  setValue(key: string, value: unknown): void { this.formData[key] = value; }
  getValue(key: string): unknown { return this.formData[key]; }
  reset(): void { this.formData = {}; }
}

// ── Component with its own instance ──
@Component({
  selector: 'app-user-form',
  providers: [FormStateService],  // New instance per component!
  template: `<form>...</form>`
})
export class UserFormComponent {
  constructor(private formState: FormStateService) {}
  // Each <app-user-form> on the page has its OWN formState
}

// Usage in parent:
// <app-user-form></app-user-form>  ← Instance #1
// <app-user-form></app-user-form>  ← Instance #2 (independent)
```

### 🆚 vs.
| Aspect | `providedIn: 'root'` | Component `providers` |
|--------|---------------------|----------------------|
| Instances | 1 (singleton) | 1 per component |
| Scope | Entire app | Component + children |
| Lifecycle | App lifetime | Component lifetime |
| Tree-shakeable | ✅ Yes | ❌ No |
| Use case | Shared state (auth, cart) | Isolated state (forms) |

### ⚡ Remember
> Component-level `providers` = **new instance per component** | Instance lifecycle tied to component | Children share parent's instance | Use for isolated state, not shared state

### 🔗 Follow-ups
- `viewProviders` vs `providers` (content projection scoping)
- Lazy module injector creating separate instances
- `useFactory` for conditional service creation

---

<a id="q19"></a>
## Q19. What is the lifecycle of a component? Key lifecycle hooks

### 📝 One-Liner
Angular components go through a defined lifecycle: **creation → change detection → destruction** — with hooks like `ngOnInit` (initialization), `ngOnChanges` (input changes), `ngAfterViewInit` (view ready), and `ngOnDestroy` (cleanup).

### 🔑 Quick Answer
**Lifecycle order**: (1) `constructor` — DI, no DOM yet. (2) `ngOnChanges` — called when `@Input` values change. (3) `ngOnInit` — one-time initialization (API calls go here). (4) `ngDoCheck` — custom change detection. (5) `ngAfterContentInit` — after `<ng-content>` projected. (6) `ngAfterContentChecked` — after projected content checked. (7) `ngAfterViewInit` — after component's view + children initialized. (8) `ngAfterViewChecked` — after view checked. (9) `ngOnDestroy` — cleanup (unsubscribe, detach listeners). *(ngOnInit sabse zyada use hota hai — API calls, initialization. ngOnDestroy mein cleanup karo — memory leaks se bachne ke liye)*

### 📖 How It Works
```
Component Lifecycle (execution order):

  constructor()          ← DI injection, no DOM, no @Input values
       ↓
  ngOnChanges()          ← Called EVERY time @Input changes (first + subsequent)
       ↓
  ngOnInit()             ← ONE TIME — initialization, API calls, subscriptions
       ↓
  ngDoCheck()            ← Every change detection cycle (use sparingly!)
       ↓
  ngAfterContentInit()   ← After <ng-content> projected (once)
       ↓
  ngAfterContentChecked()← After projected content checked
       ↓
  ngAfterViewInit()      ← After component view + child views init (once)
       ↓
  ngAfterViewChecked()   ← After view checked
       ↓
  [... change detection cycles repeat ngDoCheck → ngAfterViewChecked ...]
       ↓
  ngOnDestroy()          ← Component removed — CLEANUP HERE!
```

### 🗣️ Interview Script
"Angular components have a lifecycle managed by the framework. After the constructor runs for DI, `ngOnChanges` fires first — it runs whenever an `@Input` property changes, including the initial binding. Then `ngOnInit` runs once — this is where I do initialization logic like API calls, because by this point all `@Input` values are available. `ngAfterViewInit` fires after the component's view and child views are fully initialized — I use this for DOM manipulations or accessing `@ViewChild` references. During the component's life, change detection cycles trigger `ngDoCheck`, `ngAfterContentChecked`, and `ngAfterViewChecked` repeatedly. Finally, `ngOnDestroy` fires when the component is about to be removed from the DOM — this is critical for cleanup: unsubscribing from Observables, clearing intervals, detaching event listeners to prevent memory leaks."

### 💻 Code
```typescript
@Component({
  selector: 'app-user-detail',
  template: `<h2>{{ user?.name }}</h2> <div #chart></div>`
})
export class UserDetailComponent implements OnInit, OnChanges, AfterViewInit, OnDestroy {
  @Input() userId!: number;
  @ViewChild('chart') chartRef!: ElementRef;
  user: User | null = null;
  private destroy$ = new Subject<void>();

  constructor(private userService: UserService) {
    // ✅ Only DI — no logic here
  }

  ngOnChanges(changes: SimpleChanges): void {
    // Runs when @Input userId changes
    if (changes['userId'] && !changes['userId'].firstChange) {
      this.loadUser();  // Reload on subsequent changes
    }
  }

  ngOnInit(): void {
    // ✅ One-time init — API calls, subscriptions
    this.loadUser();
  }

  ngAfterViewInit(): void {
    // ✅ DOM is ready — safe to access @ViewChild
    this.initChart(this.chartRef.nativeElement);
  }

  ngOnDestroy(): void {
    // ✅ Cleanup — prevent memory leaks
    this.destroy$.next();
    this.destroy$.complete();
  }

  private loadUser(): void {
    this.userService.getUserById(this.userId).pipe(
      takeUntil(this.destroy$)  // Auto-unsubscribe on destroy
    ).subscribe(user => this.user = user);
  }

  private initChart(el: HTMLElement): void { /* chart library init */ }
}
```

### ⚠️ Pitfalls
| Mistake | Fix |
|---------|-----|
| API calls in constructor | Move to `ngOnInit` — inputs not ready in constructor |
| Accessing `@ViewChild` in `ngOnInit` | Use `ngAfterViewInit` — view isn't rendered yet in ngOnInit |
| Not unsubscribing in `ngOnDestroy` | Use `takeUntil(destroy$)` or `async` pipe |
| Heavy logic in `ngDoCheck` | Runs on EVERY detection cycle — keep minimal |

### ⚡ Remember
> `ngOnInit` = **most used** (API calls, init) | `ngOnDestroy` = **cleanup** (unsubscribe!) | `ngOnChanges` = react to @Input changes | `ngAfterViewInit` = DOM ready | Don't put logic in constructor

### 🔗 Follow-ups
- `async` pipe vs manual subscription
- `ChangeDetectionStrategy.OnPush` for performance
- Angular Signals (future replacement for lifecycle patterns)

---

<a id="q20"></a>
## Q20. How do services help in testing?

### 📝 One-Liner
Services make testing easy because they can be **easily mocked** — in component tests, you replace real services with mock objects, isolating the component from backend dependencies and testing only the UI logic.

### 🔑 Quick Answer
**Without services**: Component directly uses `HttpClient` → test needs `HttpClientTestingModule`, mock HTTP responses, handle async — complex and slow. **With services**: Component depends on `UserService` → test provides a mock `UserService` with fake data → no HTTP, no async, fast and focused. **Service tests**: Test the service itself with `HttpClientTestingModule` in isolation. *(Service mock karna easy hai — test mein fake service de do, component ka test clean aur fast hota hai)*

### 📖 How It Works
```
Testing Pyramid with Services:

Unit Test (Component):
  ┌─────────────────────┐
  │ UserListComponent   │
  │ inject: MockService │ ← Fake data, no HTTP
  │ test: renders list? │
  │ test: click works?  │
  └─────────────────────┘
  ⚡ Fast, isolated, no backend

Unit Test (Service):
  ┌─────────────────────┐
  │ UserService         │
  │ inject: HttpClient  │ ← HttpClientTestingModule
  │ test: correct URL?  │
  │ test: maps data?    │
  └─────────────────────┘
  Isolated from component

Integration Test:
  ┌─────────────────────┐
  │ Component + Service │ ← Real service, mocked HTTP
  └─────────────────────┘
  Tests wiring works
```

### 🗣️ Interview Script
"Services are a testing enabler. When I separate API calls into a service, I can test the component and service independently. For component tests, I create a mock service using `jasmine.createSpyObj` or a simple stub object. I configure TestBed to provide the mock instead of the real service — now the component test has zero dependency on HTTP, databases, or external APIs. The test runs instantly and deterministically. I test things like 'does the component render the user list?', 'does clicking delete emit the right event?' — pure UI logic. Separately, I test the service with `HttpClientTestingModule` where I verify it calls the right URLs, sends correct payloads, and transforms responses properly. This separation follows the testing pyramid — many fast unit tests, fewer integration tests."

### 💻 Code
```typescript
// ── Component Test (mock service) ──
describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  let mockUserService: jasmine.SpyObj<UserService>;

  beforeEach(() => {
    // Create mock with fake return values
    mockUserService = jasmine.createSpyObj('UserService', ['getUsers', 'deleteUser']);
    mockUserService.getUsers.and.returnValue(of([
      { id: 1, name: 'Alice' },
      { id: 2, name: 'Bob' }
    ]));

    TestBed.configureTestingModule({
      declarations: [UserListComponent],
      providers: [
        { provide: UserService, useValue: mockUserService }  // Inject mock!
      ]
    });

    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should render user list', () => {
    const items = fixture.nativeElement.querySelectorAll('.user-item');
    expect(items.length).toBe(2);  // Fast — no HTTP!
  });

  it('should call deleteUser on click', () => {
    mockUserService.deleteUser.and.returnValue(of(void 0));
    component.delete(1);
    expect(mockUserService.deleteUser).toHaveBeenCalledWith(1);
  });
});

// ── Service Test (mock HTTP) ──
describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService]
    });
    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  it('should fetch users from correct URL', () => {
    service.getUsers().subscribe(users => {
      expect(users.length).toBe(2);
    });
    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush([{ id: 1 }, { id: 2 }]);
  });
});
```

### ⚡ Remember
> **Mock service** in component tests = fast, isolated | **Mock HTTP** in service tests = verify URLs + payloads | `jasmine.createSpyObj` for quick mocks | `TestBed.configureTestingModule` for DI override | Separation = testability

### 🔗 Follow-ups
- Spectator library for cleaner Angular tests
- `ng-mocks` for automatic mocking
- E2E testing with Cypress / Playwright
