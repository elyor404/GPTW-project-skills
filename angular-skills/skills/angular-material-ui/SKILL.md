---
name: angular-material-ui
description: Angular Material UI component patterns - theming, common components, forms integration, accessibility. Use when building UI with Material Design.
---

# Angular Material UI

## When to Use This Skill

Use this skill when:
- Building UI with Material Design components
- Creating consistent forms and tables
- Implementing dialogs and notifications
- Setting up application theming

## Setup

### Installation

```bash
ng add @angular/material
```

### Theme Configuration

```scss
// src/styles.scss
@use '@angular/material' as mat;

// Define custom theme colours
$primary-palette: (
  50: #e3f2fd,
  100: #bbdefb,
  500: #0066cc,  // GPTW Blue
  700: #004c99,
  contrast: (
    50: rgba(0, 0, 0, 0.87),
    500: white,
    700: white,
  )
);

$gptw-primary: mat.define-palette($primary-palette);
$gptw-accent: mat.define-palette(mat.$amber-palette);
$gptw-warn: mat.define-palette(mat.$red-palette);

$gptw-theme: mat.define-light-theme((
  color: (
    primary: $gptw-primary,
    accent: $gptw-accent,
    warn: $gptw-warn,
  ),
  typography: mat.define-typography-config(),
  density: 0,
));

@include mat.all-component-themes($gptw-theme);
```

## Common Components

### Data Table

```typescript
// src/app/features/admin/company-list/company-list.component.ts
import { Component, ViewChild, inject, signal, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { MatTableModule, MatTableDataSource } from '@angular/material/table';
import { MatPaginatorModule, MatPaginator } from '@angular/material/paginator';
import { MatSortModule, MatSort } from '@angular/material/sort';
import { MatInputModule } from '@angular/material/input';
import { MatFormFieldModule } from '@angular/material/form-field';
import { MatButtonModule } from '@angular/material/button';
import { MatIconModule } from '@angular/material/icon';
import { Company } from '../../../shared/models/company.model';

@Component({
  selector: 'app-company-list',
  standalone: true,
  imports: [
    CommonModule,
    MatTableModule,
    MatPaginatorModule,
    MatSortModule,
    MatInputModule,
    MatFormFieldModule,
    MatButtonModule,
    MatIconModule,
  ],
  template: `
    <div class="table-container">
      <mat-form-field appearance="outline" class="filter-field">
        <mat-label>Search companies</mat-label>
        <input matInput (keyup)="applyFilter($event)" placeholder="Type to filter..." />
        <mat-icon matSuffix>search</mat-icon>
      </mat-form-field>

      <table mat-table [dataSource]="dataSource" matSort>
        <!-- Name Column -->
        <ng-container matColumnDef="name">
          <th mat-header-cell *matHeaderCellDef mat-sort-header>Company Name</th>
          <td mat-cell *matCellDef="let company">{{ company.name }}</td>
        </ng-container>

        <!-- Client ID Column -->
        <ng-container matColumnDef="clientId">
          <th mat-header-cell *matHeaderCellDef mat-sort-header>Client ID</th>
          <td mat-cell *matCellDef="let company">{{ company.clientId }}</td>
        </ng-container>

        <!-- Status Column -->
        <ng-container matColumnDef="status">
          <th mat-header-cell *matHeaderCellDef>Status</th>
          <td mat-cell *matCellDef="let company">
            <span class="status-badge" [class.active]="!company.isDeleted">
              {{ company.isDeleted ? 'Inactive' : 'Active' }}
            </span>
          </td>
        </ng-container>

        <!-- Actions Column -->
        <ng-container matColumnDef="actions">
          <th mat-header-cell *matHeaderCellDef>Actions</th>
          <td mat-cell *matCellDef="let company">
            <button mat-icon-button (click)="editCompany(company)">
              <mat-icon>edit</mat-icon>
            </button>
            <button mat-icon-button color="warn" (click)="deleteCompany(company)">
              <mat-icon>delete</mat-icon>
            </button>
          </td>
        </ng-container>

        <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
        <tr mat-row *matRowDef="let row; columns: displayedColumns;"></tr>

        <!-- Empty State -->
        <tr class="mat-row" *matNoDataRow>
          <td class="mat-cell" colspan="4">No companies found</td>
        </tr>
      </table>

      <mat-paginator
        [pageSizeOptions]="[10, 25, 50]"
        showFirstLastButtons
        aria-label="Select page">
      </mat-paginator>
    </div>
  `,
})
export class CompanyListComponent implements OnInit {
  @ViewChild(MatPaginator) paginator!: MatPaginator;
  @ViewChild(MatSort) sort!: MatSort;

  displayedColumns = ['name', 'clientId', 'status', 'actions'];
  dataSource = new MatTableDataSource<Company>();

  ngOnInit(): void {
    this.loadCompanies();
  }

  ngAfterViewInit(): void {
    this.dataSource.paginator = this.paginator;
    this.dataSource.sort = this.sort;
  }

  applyFilter(event: Event): void {
    const filterValue = (event.target as HTMLInputElement).value;
    this.dataSource.filter = filterValue.trim().toLowerCase();
  }

  editCompany(company: Company): void {
    // Navigate to edit
  }

  deleteCompany(company: Company): void {
    // Show confirmation dialog
  }

  private async loadCompanies(): Promise<void> {
    // Load from API
  }
}
```

### Form with Material Components

```typescript
// src/app/features/admin/company-form/company-form.component.ts
import { Component, inject, input, output } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ReactiveFormsModule, FormBuilder, Validators } from '@angular/forms';
import { MatFormFieldModule } from '@angular/material/form-field';
import { MatInputModule } from '@angular/material/input';
import { MatSelectModule } from '@angular/material/select';
import { MatCheckboxModule } from '@angular/material/checkbox';
import { MatButtonModule } from '@angular/material/button';
import { MatProgressSpinnerModule } from '@angular/material/progress-spinner';

@Component({
  selector: 'app-company-form',
  standalone: true,
  imports: [
    CommonModule,
    ReactiveFormsModule,
    MatFormFieldModule,
    MatInputModule,
    MatSelectModule,
    MatCheckboxModule,
    MatButtonModule,
    MatProgressSpinnerModule,
  ],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <mat-form-field appearance="outline" class="full-width">
        <mat-label>Company Name</mat-label>
        <input matInput formControlName="name" />
        @if (form.get('name')?.hasError('required')) {
          <mat-error>Company name is required</mat-error>
        }
      </mat-form-field>

      <mat-form-field appearance="outline" class="full-width">
        <mat-label>Client ID</mat-label>
        <input matInput formControlName="clientId" />
        @if (form.get('clientId')?.hasError('pattern')) {
          <mat-error>Format: XX-1234</mat-error>
        }
      </mat-form-field>

      <mat-form-field appearance="outline" class="full-width">
        <mat-label>CRM ID (HubSpot)</mat-label>
        <input matInput formControlName="crmId" />
      </mat-form-field>

      <h3>Packages</h3>
      <mat-checkbox formControlName="activateAi">Activate AI</mat-checkbox>
      <mat-checkbox formControlName="elevate">Elevate</mat-checkbox>
      <mat-checkbox formControlName="empower">Empower</mat-checkbox>

      <div class="form-actions">
        <button mat-button type="button" (click)="cancel.emit()">Cancel</button>
        <button
          mat-raised-button
          color="primary"
          type="submit"
          [disabled]="form.invalid || saving()">
          @if (saving()) {
            <mat-spinner diameter="20"></mat-spinner>
          } @else {
            Save
          }
        </button>
      </div>
    </form>
  `,
})
export class CompanyFormComponent {
  private readonly fb = inject(FormBuilder);

  company = input<Company | null>(null);
  saving = input(false);
  saved = output<CompanyFormValue>();
  cancel = output<void>();

  form = this.fb.group({
    name: ['', Validators.required],
    clientId: ['', [Validators.required, Validators.pattern(/^[A-Z]{2,4}-\d{4,8}$/)]],
    crmId: [''],
    activateAi: [false],
    elevate: [false],
    empower: [false],
  });

  onSubmit(): void {
    if (this.form.valid) {
      this.saved.emit(this.form.value as CompanyFormValue);
    }
  }
}
```

### Dialog Component

```typescript
// src/app/shared/components/confirm-dialog/confirm-dialog.component.ts
import { Component, inject } from '@angular/core';
import { MAT_DIALOG_DATA, MatDialogRef, MatDialogModule } from '@angular/material/dialog';
import { MatButtonModule } from '@angular/material/button';

export interface ConfirmDialogData {
  title: string;
  message: string;
  confirmText?: string;
  cancelText?: string;
  isDestructive?: boolean;
}

@Component({
  selector: 'app-confirm-dialog',
  standalone: true,
  imports: [MatDialogModule, MatButtonModule],
  template: `
    <h2 mat-dialog-title>{{ data.title }}</h2>
    <mat-dialog-content>
      <p>{{ data.message }}</p>
    </mat-dialog-content>
    <mat-dialog-actions align="end">
      <button mat-button mat-dialog-close>
        {{ data.cancelText || 'Cancel' }}
      </button>
      <button
        mat-raised-button
        [color]="data.isDestructive ? 'warn' : 'primary'"
        [mat-dialog-close]="true">
        {{ data.confirmText || 'Confirm' }}
      </button>
    </mat-dialog-actions>
  `,
})
export class ConfirmDialogComponent {
  readonly data = inject<ConfirmDialogData>(MAT_DIALOG_DATA);
}

// Usage
@Injectable({ providedIn: 'root' })
export class DialogService {
  private readonly dialog = inject(MatDialog);

  confirm(data: ConfirmDialogData): Promise<boolean> {
    const dialogRef = this.dialog.open(ConfirmDialogComponent, {
      width: '400px',
      data,
    });

    return firstValueFrom(dialogRef.afterClosed());
  }
}
```

### Snackbar Notifications

```typescript
// src/app/core/services/notification.service.ts
import { Injectable, inject } from '@angular/core';
import { MatSnackBar } from '@angular/material/snack-bar';

@Injectable({ providedIn: 'root' })
export class NotificationService {
  private readonly snackBar = inject(MatSnackBar);

  showSuccess(message: string): void {
    this.snackBar.open(message, 'Close', {
      duration: 3000,
      panelClass: ['snackbar-success'],
      horizontalPosition: 'end',
      verticalPosition: 'top',
    });
  }

  showError(message: string): void {
    this.snackBar.open(message, 'Close', {
      duration: 5000,
      panelClass: ['snackbar-error'],
      horizontalPosition: 'end',
      verticalPosition: 'top',
    });
  }

  showInfo(message: string): void {
    this.snackBar.open(message, 'Close', {
      duration: 3000,
      horizontalPosition: 'end',
      verticalPosition: 'top',
    });
  }
}
```

## References

- [Angular Material](https://material.angular.io/)
- [Material Design Guidelines](https://material.io/design)
