---
title: Signal Form with Asynchronous Validation
summary: Implements an asynchronous validator on a signal form field using `validateAsync` to check for a unique username against a mock backend service.
keywords:
  - signal forms
  - form
  - control
  - validation
  - asynchronous validation
  - async
  - validateAsync
  - schema
required_packages:
  - '@angular/forms'
related_concepts:
  - 'signals'
---

## Purpose

The purpose of this pattern is to perform validation that requires an asynchronous operation, such as an HTTP request. It solves the problem of verifying field values against a backend or other external data source without blocking the user interface.

## When to Use

Use this pattern for validation that requires a network request, a debounce, or any other asynchronous operation. It allows the UI to remain responsive while the validation is in progress and provides feedback to the user once the validation is complete. This is the modern, signal-based equivalent of `AsyncValidator` in `ReactiveFormsModule`.

## Key Concepts

- **`validateAsync()`:** A function used within a schema to add an asynchronous validator to a field.
- **Asynchronous Validator Function:** A function that returns a `Promise` which resolves to an error object or `null`.
- **`pending` signal:** A signal on the `FieldState` that is `true` while an asynchronous validator is running.

## Example Files

This example consists of a standalone component and a service that work together to perform asynchronous validation.

### username.service.ts

This file defines a mock service that simulates an asynchronous check for username uniqueness.

```typescript
import { Injectable } from '@angular/core';
import { of } from 'rxjs';
import { delay } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class UsernameService {
  private existingUsernames = ['admin', 'user', 'test'];

  isUsernameTaken(username: string): Promise<boolean> {
    return of(this.existingUsernames.includes(username.toLowerCase()))
      .pipe(delay(500))
      .toPromise();
  }
}
```

### registration-form.component.ts

This file defines the component's logic, including a schema that uses `validateAsync` to call the username service.

```typescript
import { Component, inject, signal, Output, EventEmitter, ChangeDetectionStrategy } from '@angular/core';
import { form, schema, validate, validateAsync } from '@angular/forms/signals';
import { JsonPipe } from '@angular/common';
import { UsernameService } from './username.service';

export interface RegistrationForm {
  username: string;
}

const createRegistrationSchema = (usernameService: UsernameService) => {
  return schema<RegistrationForm>((registrationForm) => {
    validate(registrationForm.username, ({value}) => {
      if (value === '') {
        return { required: true };
      }
      return null;
    });
    validateAsync(registrationForm.username, async ({value}) => {
      if (value === '') return null;
      const isTaken = await usernameService.isUsernameTaken(value);
      return isTaken ? { unique: true } : null;
    });
  });
};

@Component({
  selector: 'app-registration-form',
  standalone: true,
  imports: [JsonPipe],
  templateUrl: './registration-form.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class RegistrationFormComponent {
  @Output() submitted = new EventEmitter<RegistrationForm>();

  private usernameService = inject(UsernameService);

  registrationModel = signal<RegistrationForm>({
    username: '',
  });

  registrationForm = form(this.registrationModel, createRegistrationSchema(this.usernameService));

  handleSubmit() {
    if (this.registrationForm().valid()) {
      this.submitted.emit(this.registrationForm().value());
    }
  }
}
```

### registration-form.component.html

This file provides the template for the form, showing how to display feedback while the asynchronous validation is pending.

```html
<form (ngSubmit)="handleSubmit()">
  <div>
    <label>
      Username:
      <input type="text" [control]="registrationForm.username" />
    </label>
    @if (registrationForm.username().pending()) {
      <p>Checking availability...</p>
    }
    @if (registrationForm.username().errors().length > 0) {
      <div class="errors">
        @for (error of registrationForm.username().errors()) {
          @if (error.required) {
            <p>Username is required.</p>
          } @else if (error.unique) {
            <p>This username is already taken.</p>
          }
        }
      </div>
    }
  </div>

  <button type="submit" [disabled]="!registrationForm().valid() || registrationForm().pending()">Submit</button>
</form>
```

## Usage Notes

- The **`validateAsync`** function takes a `FieldPath` and an async validator function.
- The async validator function should return a `Promise` that resolves with the validation result.
- The **`pending`** signal on the field state (`registrationForm.username().pending()`) can be used to show a loading indicator to the user.

## How to Use This Example

The parent component listens for the `(submitted)` event to receive the strongly-typed form data.

```typescript
// in app.component.ts
import { Component } from '@angular/core';
import { JsonPipe } from '@angular/common';
import { RegistrationFormComponent, RegistrationForm } from './registration-form.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RegistrationFormComponent, JsonPipe],
  template: `
    <h1>Register</h1>
    <app-registration-form (submitted)="onFormSubmit($event)"></app-registration-form>
    
    @if (submittedData) {
      <h2>Submitted Data:</h2>
      <pre>{{ submittedData | json }}</pre>
    }
  `,
})
export class AppComponent {
  submittedData: RegistrationForm | null = null;

  onFormSubmit(data: RegistrationForm) {
    this.submittedData = data;
    console.log('Registration data submitted:', data);
  }
}
```
