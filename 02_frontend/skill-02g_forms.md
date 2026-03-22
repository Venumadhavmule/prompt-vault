# SKILL 02-G — How to Handle Forms: Validation, Submission, and Errors
Category: Frontend and UI
Applies to: React, Next.js, Vue; React Hook Form, Zod, Valibot

## What this skill covers
Forms are the primary mechanism through which users create and modify data. Poor form implementations cause data loss, confusing error states, silent failures, double-submissions, and security vulnerabilities. This skill covers the correct end-to-end form pattern: schema-driven validation with Zod, submission handling with React Hook Form, server error propagation, loading states, and accessible error messaging.

## When to activate this skill
- When building any form that creates, updates, or sends data
- When adding validation to an existing form
- When a form has inconsistent error state behavior
- When a form can be double-submitted
- When server validation errors are not displayed to the user

## Core principles
1. **Validate on both client and server.** Client validation for UX, server validation for security. Neither can replace the other.
2. **Schema first.** Define the Zod schema before the form UI. The schema is the single source of truth for field types, constraints, and error messages.
3. **Server errors must surface to the user.** A validation error that stays on the server and leaves the user staring at a stuck form is a broken form.
4. **Disable the submit button during submission.** Never allow double-submission. Show a loading indicator during the pending state.
5. **Error messages must be visible, accessible, and specific.** "Invalid input" is not a valid error message. "Email must be at least 5 characters" is.

## Step-by-step guide
1. Define the Zod schema with all validation rules before creating the form:
   ```ts
   const loginSchema = z.object({
     email: z.string().email("Enter a valid email address"),
     password: z.string().min(8, "Password must be at least 8 characters"),
   });
   type LoginFormData = z.infer<typeof loginSchema>;
   ```
2. Set up React Hook Form with the Zod resolver:
   ```ts
   const { register, handleSubmit, formState: { errors, isSubmitting }, setError } =
     useForm<LoginFormData>({ resolver: zodResolver(loginSchema) });
   ```
3. Implement the submit handler:
   - Set loading state (React Hook Form's `isSubmitting` handles this automatically)
   - Call the API
   - On API error: use `setError('fieldName', { message: serverError })` or `setError('root', { message: genericError })`
   - On success: redirect or reset form
4. Render each field with: label, input with `{...register('fieldName')}`, and error message via `{errors.fieldName?.message}`.
5. Connect error messages to inputs with `aria-describedby` for screen readers.
6. Disable the submit button with `disabled={isSubmitting}` and show a spinner alongside it.
7. For multi-step forms: validate each step with the subset of the schema before advancing.
8. For server-rendered forms (Next.js Server Actions): use Zod's `safeParse` in the action, return validation errors in the response, and use `useActionState` to display them.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Validate only on the server | Validate on client for UX AND on server for security |
| `if (email === '')` — manual field-by-field validation | Define a Zod schema and validate the entire form at once |
| Show "An error occurred" for a validation failure | Show the specific Zod error message for the specific field |
| No loading state — user clicks submit twice | `disabled={isSubmitting}` on submit button |
| Server validation error ignored; form looks successful | Call `setError('fieldName', ...)` to surface server errors |
| Error message in a `<p>` not connected to the input | `<p id="email-error" role="alert">{errors.email?.message}</p>` with `aria-describedby="email-error"` on the input |

## Stack-specific notes
**Next.js App Router with Server Actions:** Use `useActionState` hook + Server Actions for progressive-enhancement forms. Zod validation in the Server Action. Return typed error object, not throws.
**Vue:** Use VeeValidate with Zod for equivalent schema-driven validation. The principles (schema first, server error surfacing, disabled during submit) are identical.
**Go (server-rendered):** Validate with `go-playground/validator` or manual checks. Return validation errors as a struct that maps field names to error messages — match this structure in your HTML template.
**Python (FastAPI):** Pydantic handles server validation. Return HTTP 422 with a field-level error array. On the frontend, parse the FastAPI error response shape and call `setError` for each field.

## Common mistakes
1. **Event handler `preventDefault` without disabling the button.** `e.preventDefault()` stops the page refresh but not a second click before the first request resolves.
2. **Swallowing server validation errors.** The most common bug: the API returns 422 with field errors, the client catch block logs to console, the user sees nothing.
3. **Generic error messages.** "Something went wrong" is not actionable. Every error state must tell the user what to fix.
4. **No field-level error display.** A general error banner at the top of a 20-field form does not tell the user which field is wrong.
5. **Validating only on blur, not on submit.** Users who tab quickly through a form without triggering blur events bypass validation.

## Checklist
- [ ] Zod schema defined before form UI
- [ ] React Hook Form (or equivalent) with zodResolver used
- [ ] Client-side validation matches server-side validation rules
- [ ] Submit button disabled during `isSubmitting` with loading indicator
- [ ] Server validation errors surfaced via `setError` with field-level specificity
- [ ] Each field has a visible label
- [ ] Each field's error message connected via `aria-describedby`
- [ ] Error messages are specific, not generic
- [ ] Form resets or redirects correctly on success
- [ ] No double-submission possible
- [ ] Multi-step form validates each step before advancing
