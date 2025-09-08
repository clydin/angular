---
title: Signal Form with Cross-Field Validation
summary: Implements a cross-field validator by applying a validator to one field that reactively depends on the value of another field.
keywords:
  - signal forms
  - form
  - control
  - validation
  - cross-field validation
  - valueOf
  - schema
  - minLength
  - submit
required_packages:
  - '@angular/forms'
related_concepts:
  - 'signals'
---

## Purpose

The purpose of this pattern is to implement validation logic that depends on the values of multiple fields simultaneously. It solves the problem of ensuring consistency between related fields, such as verifying that a "confirm password" field matches the "password" field.

## When to Use

Use this pattern when the validity of one form field depends on the value of another. This is common in registration forms, password change forms, and any form where two fields must be consistent with each other. The recommended approach is to apply a validator to the field where the error should be reported (e.g., `confirmPassword`) and use the `valueOf` function in the validation context to reactively get the value of the other field it depends on (e.g., `password`).

## Key Concepts

- **`validate()`:** A function used within a schema to add a validator to a specific field.
- **`valueOf()`:** A function available in the validation context that allows you to reactively access the value of any other field in the form.
- **`valid` signal:** A signal on the `FieldState` that is `true` only when the field and all its descendants are valid.

## Example Files

This example consists of a standalone component that defines and manages a password change form.

### password-form.component.ts

This file defines the component's logic. The schema applies a validator to `confirmPassword` that compares its value to the value of the `password` field.

```typescript
import { Component, signal, Output, EventEmitter, ChangeDetectionStrategy } from '@angular/core';
import { form, schema, validate } from '@angular/forms/signals';
import { JsonPipe } from '@angular/common';

export interface PasswordForm {
  password: string;
  confirmPassword: string;
}

const passwordSchema = schema<PasswordForm>((passwordForm) => {
  validate(passwordForm.password, ({value}) => {
    if (value === '') return { required: true };
    if (value.length < 8) return { minLength: { requiredLength: 8, actualLength: value.length } };
    return null;
  });

  validate(passwordForm.confirmPassword, ({value, valueOf}) => {
    if (value !== valueOf(passwordForm.password)) {
      return { mismatch: true };
    }
    return null;
  });
});

@Component({
  selector: 'app-password-form',
  standalone: true,
  imports: [JsonPipe],
  templateUrl: './password-form.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class PasswordFormComponent {
  @Output() submitted = new EventEmitter<PasswordForm>();

  passwordModel = signal<PasswordForm>({
    password: '',
    confirmPassword: '',
  });

  passwordForm = form(this.passwordModel, passwordSchema);

  handleSubmit() {
    if (this.passwordForm().valid()) {
      this.submitted.emit(this.passwordForm().value());
    }
  }
}
```

### password-form.component.html

This file provides the template for the form, showing how to display errors for both field-level and cross-field validation rules.

```html
<form (ngSubmit)="handleSubmit()">
  <div>
    <label>
      Password:
      <input type="password" [control]="passwordForm.password" />
    </label>
    @if (passwordForm.password().errors().length > 0) {
      <div class="errors">
        @for (error of passwordForm.password().errors()) {
          @if (error.required) { <p>Password is required.</p> }
          @if (error.minLength) { <p>Password must be at least 8 characters long.</p> }
        }
      </div>
    }
  </div>
  <div>
    <label>
      Confirm Password:
      <input type="password" [control]="passwordForm.confirmPassword" />
    </label>
    @if (passwordForm.confirmPassword().errors().length > 0) {
      <div class="errors">
        @for (error of passwordForm.confirmPassword().errors()) {
          @if (error.mismatch) { <p>Passwords do not match.</p> }
        }
      </div>
    }
  </div>

  <button type="submit" [disabled]="!passwordForm().valid()">Submit</button>
</form>
```

## Usage Notes

- The submit button is disabled based on the form's `valid` signal (`[disabled]="!passwordForm().valid()"`).
- The `handleSubmit` method checks `this.passwordForm().valid()` as a safeguard before emitting the data.

## How to Use This Example

The parent component listens for the `(submitted)` event and receives the strongly-typed form data.

```typescript
// in app.component.ts
import { Component } from '@angular/core';
import { JsonPipe } from '@angular/common';
import { PasswordFormComponent, PasswordForm } from './password-form.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [PasswordFormComponent, JsonPipe],
  template: `
    <h1>Change Password</h1>
    <app-password-form (submitted)="onPasswordSubmit($event)"></app-password-form>
    
    @if (submittedData) {
      <h2>Submitted Data:</h2>
      <pre>{{ submittedData | json }}</pre>
    }
  `,
})
export class AppComponent {
  submittedData: PasswordForm | null = null;

  onPasswordSubmit(data: PasswordForm) {
    this.submittedData = data;
    console.log('Password data submitted:', data);
  }
}
```