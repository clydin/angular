---
title: Signal Form with Conditional Validation
summary: Applies a validator to a signal form field only when a specific condition is met by using the `validateWhen` function.
keywords:
  - signal forms
  - form
  - control
  - validation
  - conditional validation
  - validateWhen
  - schema
  - submit
required_packages:
  - '@angular/forms'
related_concepts:
  - 'signals'
---

## Purpose

The purpose of this pattern is to create dynamic forms where the validation rules for one field depend on the state or value of another. It solves the problem of applying validation logic conditionally, which prevents the over-validation of fields that are not relevant in a given context.

## When to Use

Use this pattern when a validation rule for one field depends on the state of another. This is useful for creating dynamic forms that adapt to user input, making the user experience smoother by not enforcing unnecessary rules. This provides a more integrated and type-safe way to handle logic that was previously managed manually in `ReactiveFormsModule` by enabling/disabling controls or validators.

## Key Concepts

- **`validateWhen()`:** A function used within a schema to apply a validator conditionally.
- **`valid` signal:** A signal on the `FieldState` that is `true` only when the field and all its descendants are valid.
- **`(ngSubmit)`:** An event binding on a `<form>` element that triggers a method when the form is submitted.

## Example Files

This example consists of a standalone component that defines and manages a contact form with a conditional field.

### contact-form.component.ts

This file defines the component's logic, including a schema that uses `validateWhen` to apply a validator conditionally.

```typescript
import { Component, signal, Output, EventEmitter, ChangeDetectionStrategy } from '@angular/core';
import { form, schema, validateWhen } from '@angular/forms/signals';
import { JsonPipe } from '@angular/common';

export interface ContactForm {
  subscribe: boolean;
  email: string;
}

const contactSchema = schema<ContactForm>((contactForm) => {
  validateWhen(
    contactForm.email,
    ({valueOf}) => valueOf(contactForm.subscribe),
    ({value}) => (value === '' ? { required: true } : null)
  );
});

@Component({
  selector: 'app-contact-form',
  standalone: true,
  imports: [JsonPipe],
  templateUrl: './contact-form.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ContactFormComponent {
  @Output() submitted = new EventEmitter<ContactForm>();

  contactModel = signal<ContactForm>({
    subscribe: false,
    email: '',
  });

  contactForm = form(this.contactModel, contactSchema);

  handleSubmit() {
    if (this.contactForm().valid()) {
      this.submitted.emit(this.contactForm().value());
    }
  }
}
```

### contact-form.component.html

This file provides the template for the form, which includes a checkbox that controls the validation of another field.

```html
<form (ngSubmit)="handleSubmit()">
  <div>
    <label>
      <input type="checkbox" [control]="contactForm.subscribe" />
      Subscribe to our newsletter
    </label>
  </div>

  <div>
    <label>
      Email:
      <input type="email" [control]="contactForm.email" />
    </label>
    @if (contactForm.email().errors().length > 0) {
      <div class="errors">
        <p>Email is required when you subscribe.</p>
      </div>
    }
  </div>

  <button type="submit" [disabled]="!contactForm().valid()">Submit</button>
</form>
```

## Usage Notes

- The submit button is disabled based on the form's `valid` signal (`[disabled]="!contactForm().valid()"`).
- The `handleSubmit` method checks `this.contactForm().valid()` as a safeguard before emitting the data.

## How to Use This Example

The parent component listens for the `(submitted)` event and receives the strongly-typed form data.

```typescript
// in app.component.ts
import { Component } from '@angular/core';
import { JsonPipe } from '@angular/common';
import { ContactFormComponent, ContactForm } from './contact-form.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ContactFormComponent, JsonPipe],
  template: `
    <h1>Contact Us</h1>
    <app-contact-form (submitted)="onFormSubmit($event)"></app-contact-form>
    
    @if (submittedData) {
      <h2>Submitted Data:</h2>
      <pre>{{ submittedData | json }}</pre>
    }
  `,
})
export class AppComponent {
  submittedData: ContactForm | null = null;

  onFormSubmit(data: ContactForm) {
    this.submittedData = data;
    console.log('Contact data submitted:', data);
  }
}
```