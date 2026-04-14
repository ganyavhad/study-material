# Angular & Webpack - Study Material

> Based on skills from [ganeshavhad.com](https://ganeshavhad.com) — Software Developer Specialist

---

## 1. Angular Overview

Angular is a **TypeScript-based** front-end framework by Google for building **single-page applications (SPAs)**.

### Key Concepts
| Concept | Description |
|---------|-------------|
| Components | Building blocks of UI — template + logic + styles |
| Modules | Organize related components, services, pipes |
| Services | Reusable business logic (dependency injected) |
| Directives | Extend HTML behavior (`*ngIf`, `*ngFor`, custom) |
| Pipes | Transform data in templates (`date`, `currency`, custom) |
| Routing | Navigation between views |
| RxJS | Reactive programming with Observables |

---

## 2. Component Architecture

```typescript
// user-list.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { UserService } from '../services/user.service';
import { Subscription } from 'rxjs';

@Component({
  selector: 'app-user-list',
  templateUrl: './user-list.component.html',
  styleUrls: ['./user-list.component.scss']
})
export class UserListComponent implements OnInit, OnDestroy {
  users: User[] = [];
  private subscription: Subscription;

  constructor(private userService: UserService) {}

  ngOnInit(): void {
    this.subscription = this.userService.getUsers().subscribe({
      next: (users) => this.users = users,
      error: (err) => console.error('Failed to load users:', err)
    });
  }

  ngOnDestroy(): void {
    this.subscription?.unsubscribe();
  }
}
```

```html
<!-- user-list.component.html -->
<div class="user-list">
  <div *ngFor="let user of users; trackBy: trackByUserId" class="user-card">
    <h3>{{ user.name | titlecase }}</h3>
    <p>{{ user.email }}</p>
    <span [ngClass]="{'active': user.isActive, 'inactive': !user.isActive}">
      {{ user.isActive ? 'Active' : 'Inactive' }}
    </span>
  </div>
  <p *ngIf="users.length === 0">No users found.</p>
</div>
```

---

## 3. Services & Dependency Injection

```typescript
// user.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class UserService {
  private apiUrl = '/api/users';

  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl).pipe(
      retry(2),
      catchError(this.handleError)
    );
  }

  createUser(user: Partial<User>): Observable<User> {
    return this.http.post<User>(this.apiUrl, user).pipe(
      catchError(this.handleError)
    );
  }

  private handleError(error: HttpErrorResponse) {
    let errorMessage = 'An unknown error occurred';
    if (error.error instanceof ErrorEvent) {
      errorMessage = `Client Error: ${error.error.message}`;
    } else {
      errorMessage = `Server Error: ${error.status} - ${error.message}`;
    }
    return throwError(() => new Error(errorMessage));
  }
}
```

---

## 4. RxJS — Reactive Extensions

### Common Operators

```typescript
import { of, from, interval, forkJoin } from 'rxjs';
import {
  map, filter, switchMap, mergeMap, debounceTime, 
  distinctUntilChanged, takeUntil, catchError, tap
} from 'rxjs/operators';

// Search with debounce (typeahead)
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(query => this.searchService.search(query)),
  catchError(err => of([]))
).subscribe(results => this.results = results);

// Parallel API calls
forkJoin({
  users: this.userService.getUsers(),
  roles: this.roleService.getRoles(),
  permissions: this.permService.getPermissions()
}).subscribe(({ users, roles, permissions }) => {
  // All resolved
});
```

### Operator Decision Guide
```
Need to transform data?        → map()
Need to filter?                 → filter()
Cancel previous on new emit?    → switchMap()
Run all in parallel?            → mergeMap()
Run in order?                   → concatMap()
Combine latest from multiple?   → combineLatest()
Wait for all to complete?       → forkJoin()
Side effects (logging)?         → tap()
Rate-limit user input?          → debounceTime()
```

---

## 5. Angular Routing

```typescript
// app-routing.module.ts
const routes: Routes = [
  { path: '', redirectTo: '/dashboard', pathMatch: 'full' },
  { path: 'dashboard', component: DashboardComponent, canActivate: [AuthGuard] },
  { path: 'users', component: UserListComponent },
  { path: 'users/:id', component: UserDetailComponent },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    canLoad: [AdminGuard]
  },
  { path: '**', component: NotFoundComponent }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

### Route Guard
```typescript
@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  constructor(private authService: AuthService, private router: Router) {}

  canActivate(): boolean {
    if (this.authService.isAuthenticated()) {
      return true;
    }
    this.router.navigate(['/login']);
    return false;
  }
}
```

---

## 6. State Management with NgRx

```typescript
// actions
export const loadUsers = createAction('[User] Load Users');
export const loadUsersSuccess = createAction('[User] Load Users Success', props<{ users: User[] }>());
export const loadUsersFailure = createAction('[User] Load Users Failure', props<{ error: string }>());

// reducer
export const userReducer = createReducer(
  initialState,
  on(loadUsers, (state) => ({ ...state, loading: true })),
  on(loadUsersSuccess, (state, { users }) => ({ ...state, users, loading: false })),
  on(loadUsersFailure, (state, { error }) => ({ ...state, error, loading: false }))
);

// effect
@Injectable()
export class UserEffects {
  loadUsers$ = createEffect(() => this.actions$.pipe(
    ofType(loadUsers),
    switchMap(() => this.userService.getUsers().pipe(
      map(users => loadUsersSuccess({ users })),
      catchError(error => of(loadUsersFailure({ error: error.message })))
    ))
  ));

  constructor(private actions$: Actions, private userService: UserService) {}
}
```

---

## 7. Webpack Integration with Angular

> *"Integrated Angular with Webpack, streamlining the inclusion and collaboration of modules developed by cross-functional teams"*

### Custom Webpack Config

```javascript
// webpack.config.js (custom config with @angular-builders/custom-webpack)
const webpack = require('webpack');
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'mainApp',
      remotes: {
        teamAModule: 'teamAModule@http://localhost:4201/remoteEntry.js',
        teamBModule: 'teamBModule@http://localhost:4202/remoteEntry.js',
      },
      shared: {
        '@angular/core': { singleton: true, strictVersion: true },
        '@angular/common': { singleton: true, strictVersion: true },
        '@angular/router': { singleton: true, strictVersion: true },
        rxjs: { singleton: true, strictVersion: true }
      }
    })
  ]
};
```

### Key Webpack Concepts

| Concept | Purpose |
|---------|---------|
| **Entry** | Starting point for dependency graph |
| **Output** | Where bundled files are emitted |
| **Loaders** | Transform non-JS files (CSS, images, TS) |
| **Plugins** | Broader tasks (optimization, env variables) |
| **Module Federation** | Share modules between separate builds |
| **Code Splitting** | Split bundles for lazy loading |
| **Tree Shaking** | Remove unused exports |

### Webpack Optimization

```javascript
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all'
        },
        common: {
          name: 'common',
          minChunks: 2,
          chunks: 'all',
          priority: -20,
          reuseExistingChunk: true
        }
      }
    },
    minimize: true
  }
};
```

---

## 8. Angular Performance Optimization

### Change Detection Strategy
```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  // Only re-renders when @Input references change
})
export class PerformantComponent {
  @Input() data: ReadonlyArray<Item>;
}
```

### Lazy Loading Modules
```typescript
{
  path: 'reports',
  loadChildren: () => import('./reports/reports.module').then(m => m.ReportsModule)
}
```

### TrackBy in ngFor
```html
<div *ngFor="let item of items; trackBy: trackById">{{ item.name }}</div>
```
```typescript
trackById(index: number, item: Item): number {
  return item.id;
}
```

---

## 9. Testing

```typescript
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

  afterEach(() => httpMock.verify());

  it('should fetch users', () => {
    const mockUsers = [{ id: 1, name: 'Ganesh' }];

    service.getUsers().subscribe(users => {
      expect(users.length).toBe(1);
      expect(users[0].name).toBe('Ganesh');
    });

    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers);
  });
});
```

---

## 10. Interview Questions

1. **What is Angular's Change Detection?** — Mechanism to sync component state with DOM. Default checks entire tree; OnPush only checks on input reference change.
2. **Explain ViewEncapsulation** — `Emulated` (default, scoped CSS via attributes), `None` (global), `ShadowDom` (native shadow DOM)
3. **What is Zone.js?** — Patches async APIs to trigger change detection automatically
4. **Observable vs Promise?** — Observable: lazy, cancellable, multi-value, has operators. Promise: eager, not cancellable, single value.
5. **What is AOT compilation?** — Ahead-of-Time: compiles templates at build time for faster rendering and smaller bundles
6. **How does Module Federation work?** — Webpack 5 feature allowing separate builds to share modules at runtime, enabling micro-frontends
7. **What are Angular Signals?** — (v16+) Fine-grained reactivity primitive, alternative to Zone.js-based change detection
8. **Explain content projection** — `<ng-content>` allows parent to inject template into child component's designated slots
