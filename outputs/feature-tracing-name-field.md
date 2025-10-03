# Feature Tracing: Adding Optional Name Field to User Profile

## Overview
This document provides a comprehensive feature tracing for adding an optional `name` field to the user profile and displaying it in the UI. The analysis covers database schema, API endpoints, UI components, and implementation requirements.

## Current State Analysis

### Database Schema
The user table already has a `name` field defined in the schema:

**File:** `packages/server/src/db/schema/user.ts`
```typescript
export const users_temp = pgTable("user_temp", {
  id: text("id").notNull().primaryKey().$defaultFn(() => nanoid()),
  name: text("name").notNull().default(""), // ✅ Already exists
  // ... other fields
});
```

**Key Finding:** The `name` field already exists in the database schema with a default empty string, so no database migration is required.

### API Schema
The API schema for user updates is defined in the same file:

**File:** `packages/server/src/db/schema/user.ts`
```typescript
const createSchema = createInsertSchema(users_temp, {
  id: z.string().min(1),
  isRegistered: z.boolean().optional(),
}).omit({
  role: true,
});

export const apiUpdateUser = createSchema.partial().extend({
  password: z.string().optional(),
  currentPassword: z.string().optional(),
  // ... other fields
});
```

**Key Finding:** The `name` field is already included in the `createSchema` and `apiUpdateUser` schema, so the API can already handle name updates.

### API Endpoints
The user update endpoint is implemented in:

**File:** `apps/dokploy/server/api/routers/user.ts`
```typescript
update: protectedProcedure
  .input(apiUpdateUser)
  .mutation(async ({ input, ctx }) => {
    // ... password validation logic
    return await updateUser(ctx.user.id, input);
  }),
```

**Key Finding:** The API endpoint already supports updating the `name` field through the `apiUpdateUser` schema.

### UI Components

#### Profile Form
**File:** `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx`

Current form schema:
```typescript
const profileSchema = z.object({
  email: z.string(),
  password: z.string().nullable(),
  currentPassword: z.string().nullable(),
  image: z.string().optional(),
  allowImpersonation: z.boolean().optional().default(false),
});
```

**Missing:** The `name` field is not included in the form schema or UI.

#### User Display Components
The user's name is already displayed in some components:

1. **Impersonation Bar** (`apps/dokploy/components/dashboard/impersonation/impersonation-bar.tsx`):
   ```typescript
   {data?.user?.name || ""}
   ```

2. **User Navigation** (`apps/dokploy/components/layouts/user-nav.tsx`):
   Currently shows email, but could display name instead.

## Implementation Plan

### 1. Update Profile Form Schema
**File:** `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx`

Add `name` field to the form schema:
```typescript
const profileSchema = z.object({
  name: z.string().optional(), // Add this line
  email: z.string(),
  password: z.string().nullable(),
  currentPassword: z.string().nullable(),
  image: z.string().optional(),
  allowImpersonation: z.boolean().optional().default(false),
});
```

### 2. Update Form Default Values
Update the form's default values to include the name field:
```typescript
const form = useForm<Profile>({
  defaultValues: {
    name: data?.user?.name || "", // Add this line
    email: data?.user?.email || "",
    password: "",
    image: data?.user?.image || "",
    currentPassword: "",
    allowImpersonation: data?.user?.allowImpersonation || false,
  },
  resolver: zodResolver(profileSchema),
});
```

### 3. Update Form Reset Logic
Update the useEffect that resets form values:
```typescript
useEffect(() => {
  if (data) {
    form.reset(
      {
        name: data?.user?.name || "", // Add this line
        email: data?.user?.email || "",
        password: form.getValues("password") || "",
        image: data?.user?.image || "",
        currentPassword: form.getValues("currentPassword") || "",
        allowImpersonation: data?.user?.allowImpersonation,
      },
      {
        keepValues: true,
      },
    );
    // ... rest of the logic
  }
}, [form, data]);
```

### 4. Add Name Input Field to UI
Add the name input field to the form, positioned before the email field:
```typescript
<FormField
  control={form.control}
  name="name"
  render={({ field }) => (
    <FormItem>
      <FormLabel>{t("settings.profile.name")}</FormLabel>
      <FormControl>
        <Input
          placeholder={t("settings.profile.name")}
          {...field}
        />
      </FormControl>
      <FormMessage />
    </FormItem>
  )}
/>
```

### 5. Update Form Submission
Update the onSubmit function to include the name field:
```typescript
const onSubmit = async (values: Profile) => {
  await mutateAsync({
    name: values.name, // Add this line
    email: values.email.toLowerCase(),
    password: values.password || undefined,
    image: values.image,
    currentPassword: values.currentPassword || undefined,
    allowImpersonation: values.allowImpersonation,
  })
    .then(async () => {
      await refetch();
      toast.success("Profile Updated");
      form.reset({
        name: values.name, // Add this line
        email: values.email,
        password: "",
        image: values.image,
        currentPassword: "",
      });
    })
    .catch(() => {
      toast.error("Error updating the profile");
    });
};
```

### 6. Update User Navigation Display
**File:** `apps/dokploy/components/layouts/user-nav.tsx`

Update the user navigation to display the name instead of "Account":
```typescript
<div className="grid flex-1 text-left text-sm leading-tight">
  <span className="truncate font-semibold">
    {data?.user?.name || "Account"}
  </span>
  <span className="truncate text-xs">{data?.user?.email}</span>
</div>
```

And update the dropdown label:
```typescript
<DropdownMenuLabel className="flex flex-col">
  {data?.user?.name || "My Account"}
  <span className="text-xs font-normal text-muted-foreground">
    {data?.user?.email}
  </span>
</DropdownMenuLabel>
```

### 7. Add Translation Keys
**File:** `apps/dokploy/public/locales/en.json` (and other locale files)

Add translation for the name field:
```json
{
  "settings": {
    "profile": {
      "name": "Name",
      "email": "Email",
      "password": "Password",
      "avatar": "Avatar",
      "title": "Profile",
      "description": "Manage your account settings and preferences"
    }
  }
}
```

## Files to Modify

### Primary Changes
1. `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx`
   - Add `name` to form schema
   - Add name input field to UI
   - Update form default values and reset logic
   - Update form submission

2. `apps/dokploy/components/layouts/user-nav.tsx`
   - Display user name in navigation
   - Update dropdown label

3. `apps/dokploy/public/locales/en.json` (and other locale files)
   - Add translation for "name" field

### No Changes Required
- Database schema (name field already exists)
- API endpoints (already support name updates)
- Database migrations (not needed)

## Testing Considerations

### Manual Testing
1. **Profile Form:**
   - Navigate to `/dashboard/settings/profile`
   - Verify name field appears and is editable
   - Test form submission with name field
   - Verify name is saved and persisted

2. **User Navigation:**
   - Verify name displays in user navigation
   - Test with empty name (should fallback to "Account")
   - Test with long names (should truncate properly)

3. **Impersonation:**
   - Verify name displays correctly in impersonation bar
   - Test with users who have and don't have names set

### Edge Cases
- Empty name handling
- Long name truncation
- Special characters in names
- Form validation for name field

## Implementation Priority

### High Priority
1. Update profile form schema and UI
2. Add name input field
3. Update form submission logic

### Medium Priority
1. Update user navigation display
2. Add translation keys

### Low Priority
1. Enhanced validation for name field
2. Name display in other components

## Dependencies

### Required
- No external dependencies
- Uses existing form validation (Zod)
- Uses existing UI components (shadcn/ui)

### Optional Enhancements
- Name validation rules (length, characters)
- Name display in other user-related components
- Search/filter by name functionality

## Conclusion

The implementation is straightforward since the database schema and API endpoints already support the `name` field. The main work involves:

1. Adding the name field to the profile form UI
2. Updating form handling logic
3. Displaying the name in user navigation
4. Adding appropriate translations

The feature can be implemented incrementally, starting with the profile form and then updating other display components as needed.
