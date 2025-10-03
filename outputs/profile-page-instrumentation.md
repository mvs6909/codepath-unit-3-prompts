## Analysis: Profile Page Update Flow

### Overview
The profile page implements a comprehensive user profile management system using React Hook Form, tRPC, and Drizzle ORM. The flow handles email updates, password changes, avatar selection (predefined + Gravatar), and impersonation permissions with robust client-side validation, server-side authentication, and database transactions.

### Entry Points
- `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:59` - ProfileForm component (main UI)
- `apps/dokploy/server/api/routers/user.ts:80` - user.get tRPC query endpoint
- `apps/dokploy/server/api/routers/user.ts:145` - user.update tRPC mutation endpoint

### Core Implementation

#### 1. Profile Data Loading (`apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:61-113`)
- Fetches user data via `api.user.get.useQuery()` at line 61
- tRPC query routes to `apps/dokploy/server/api/routers/user.ts:80-96`
- Queries `member` table joined with `user` and `apiKeys` relations at lines 81-93
- Filters by current user ID and active organization ID at lines 83-84
- Generates Gravatar hash using `generateSHA256Hash()` at lines 107-111
- Updates available avatars array to include Gravatar option at lines 73-78

#### 2. Form Validation Schema (`apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:34-40`)
- Uses Zod schema `profileSchema` with fields: email, password, currentPassword, image, allowImpersonation
- Client-side validation with `zodResolver` integration at line 88
- Form state managed by React Hook Form with default values from user data at lines 80-89

#### 3. Avatar Selection System (`apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:44-78`)
- Predefined avatars array with 12 static images at lines 44-57
- Dynamic Gravatar integration using SHA256 hash of user email
- `availableAvatars` computed with `useMemo` to include Gravatar when hash available
- RadioGroup UI for avatar selection at lines 225-266

#### 4. Form Submission Handler (`apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:115-136`)
- `onSubmit` function processes form data and calls tRPC mutation
- Email normalized to lowercase at line 117
- Optional password and currentPassword handling at lines 118-121
- Mutation success triggers data refetch and form reset at lines 123-132

#### 5. Server-Side Update Processing (`apps/dokploy/server/api/routers/user.ts:145-178`)
- Protected procedure using `apiUpdateUser` schema validation at line 146
- Password validation logic when password fields provided at lines 148-176
- Fetches current account record for password comparison at lines 149-151
- Uses bcrypt to compare current password at lines 152-155
- Hashes new password with bcrypt and updates account table at lines 170-175
- Calls `updateUser` service function for user data update at line 177

#### 6. Database Update Service (`packages/server/src/services/user.ts:240-251`)
- `updateUser` function performs direct database update on `users_temp` table
- Uses Drizzle ORM update query with returning clause at lines 241-248
- Returns updated user record from database

### Data Flow
1. Component mounts → `api.user.get.useQuery()` → `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:61`
2. tRPC query → `apps/dokploy/server/api/routers/user.ts:80` → Database query via Drizzle ORM
3. User data populates form → React Hook Form `useEffect` → `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:91-113`
4. Gravatar hash generation → `generateSHA256Hash()` → `apps/dokploy/lib/utils.ts:8-14`
5. Form submission → `onSubmit` → `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:115`
6. tRPC mutation → `api.user.update.useMutation()` → `apps/dokploy/server/api/routers/user.ts:145`
7. Password validation (if provided) → bcrypt comparison → `apps/dokploy/server/api/routers/user.ts:148-176`
8. User data update → `updateUser()` → `packages/server/src/services/user.ts:240-251`
9. Database update → Drizzle ORM → `users_temp` table
10. Success response → Form reset and data refetch → `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:123-132`

### Key Patterns
- **Form Management**: React Hook Form with Zod validation resolver pattern
- **Data Fetching**: tRPC useQuery with automatic refetching and caching
- **Authentication**: Protected procedures with session validation via middleware
- **Password Security**: bcrypt hashing and comparison for password updates
- **State Management**: Server state via tRPC, form state via React Hook Form
- **Error Handling**: TRPCError with specific error codes and toast notifications

### Configuration
- Avatar images stored in `/public/avatars/` directory (`apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:44-57`)
- Gravatar integration using SHA256 email hash (`apps/dokploy/lib/utils.ts:8-14`)
- Cloud vs self-hosted feature flags from `api.settings.isCloud.useQuery()` at line 62
- bcrypt salt rounds set to 10 in password hashing (`apps/dokploy/server/api/routers/user.ts:173`)

### Authentication & Authorization
- Session validation in tRPC context (`apps/dokploy/server/api/trpc.ts:152-164`)
- User session from Better Auth via `validateRequest()` (`apps/dokploy/server/api/trpc.ts:71`)
- Organization-based access control via `activeOrganizationId` filtering
- Protected procedures ensure authenticated user context

### Database Schema
- Primary table: `users_temp` (`packages/server/src/db/schema/user.ts:28-122`)
- Image field: `text("image")` at line 46 stores avatar URL/path
- Account table for password storage (`account` schema relation)
- Member table for organization permissions and relationships

### Error Handling
- Client-side validation via Zod schema and React Hook Form
- Server-side validation using `apiUpdateUser` schema (`packages/server/src/db/schema/user.ts:298-326`)
- Password validation errors return 400 with specific message (`apps/dokploy/server/api/routers/user.ts:157-169`)
- Toast notifications for success/error feedback (`apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:125-135`)
- Form error display via FormMessage components throughout form fields

### Instrumentation Implementation

#### Client-Side Logging ✅ IMPLEMENTED
Added comprehensive logging in `ProfileForm` component at key lifecycle points:

**User Data Loading** (`profile-form.tsx:61-69`):
```typescript
const { data, refetch, isLoading } = api.user.get.useQuery(undefined, {
  onSuccess: (data) => {
    console.log('[PROFILE] User data loaded:', { 
      userId: data?.user?.id, 
      email: data?.user?.email,
      hasImage: !!data?.user?.image,
      allowImpersonation: data?.user?.allowImpersonation
    });
  }
});
```

**Avatar Selection Computation** (`profile-form.tsx:82-93`):
```typescript
const availableAvatars = useMemo(() => {
  if (gravatarHash === null) return randomImages;
  const avatarsWithGravatar = randomImages.concat([
    `https://www.gravatar.com/avatar/${gravatarHash}`,
  ]);
  console.log('[PROFILE] Available avatars computed:', { 
    count: avatarsWithGravatar.length, 
    hasGravatar: gravatarHash !== null,
    gravatarHash: gravatarHash
  });
  return avatarsWithGravatar;
}, [gravatarHash]);
```

**Form Submission Flow** (`profile-form.tsx:131-166`):
```typescript
const onSubmit = async (values: Profile) => {
  console.log('[PROFILE] Form submission started:', { 
    email: values.email, 
    hasPasswordChange: !!values.password,
    hasCurrentPassword: !!values.currentPassword,
    selectedImage: values.image,
    allowImpersonation: values.allowImpersonation
  });

  // ... mutation call ...

  .then(async () => {
    console.log('[PROFILE] Profile update successful:', { 
      email: values.email,
      updatedImage: values.image
    });
    // ... success handling ...
  })
  .catch((error) => {
    console.error('[PROFILE] Profile update failed:', {
      error: error.message,
      email: values.email
    });
    // ... error handling ...
  });
};
```

#### Server-Side Logging ✅ IMPLEMENTED
Added comprehensive logging in tRPC user router endpoints:

**Profile Data Fetching** (`user.ts:81-108`):
```typescript
get: protectedProcedure.query(async ({ ctx }) => {
  console.log('[USER_API] Profile data requested:', { 
    userId: ctx.user.id, 
    organizationId: ctx.session?.activeOrganizationId 
  });

  // ... database query ...

  console.log('[USER_API] Profile data fetched:', { 
    userId: ctx.user.id,
    hasUserData: !!memberResult?.user,
    userEmail: memberResult?.user?.email,
    userImage: memberResult?.user?.image,
    apiKeysCount: memberResult?.user?.apiKeys?.length || 0
  });

  return memberResult;
});
```

**Profile Update Flow** (`user.ts:161-239`):
```typescript
update: protectedProcedure
  .input(apiUpdateUser)
  .mutation(async ({ input, ctx }) => {
    console.log('[USER_API] Profile update requested:', { 
      userId: ctx.user.id, 
      hasPassword: !!input.password,
      hasCurrentPassword: !!input.currentPassword,
      fields: Object.keys(input),
      email: input.email,
      image: input.image
    });

    // Password validation logging when applicable
    if (input.password || input.currentPassword) {
      console.log('[USER_API] Password update requested:', { 
        userId: ctx.user.id,
        hasNewPassword: !!input.password,
        hasCurrentPassword: !!input.currentPassword
      });

      // ... validation logic ...

      console.log('[USER_API] Password validation:', { 
        userId: ctx.user.id, 
        passwordValid: correctPassword,
        hasAccountRecord: !!currentAuth
      });
    }

    console.log('[USER_API] Calling updateUser service:', { 
      userId: ctx.user.id, 
      updateFields: Object.keys(input)
    });

    const result = await updateUser(ctx.user.id, input);

    console.log('[USER_API] Profile update completed:', { 
      userId: ctx.user.id,
      updatedEmail: result?.email,
      updatedImage: result?.image,
      success: !!result
    });

    return result;
  });
```

#### Database Service Logging ✅ IMPLEMENTED
Added detailed logging in user service layer:

**User Update Function** (`user.ts:241-267`):
```typescript
export const updateUser = async (userId: string, userData: Partial<User>) => {
  console.log('[USER_SERVICE] Updating user data:', { 
    userId, 
    fields: Object.keys(userData),
    email: userData.email,
    image: userData.image,
    allowImpersonation: userData.allowImpersonation
  });

  const user = await db
    .update(users_temp)
    .set({ ...userData })
    .where(eq(users_temp.id, userId))
    .returning()
    .then((res) => res[0]);

  console.log('[USER_SERVICE] User update completed:', { 
    userId, 
    updatedFields: Object.keys(userData),
    resultEmail: user?.email,
    resultImage: user?.image,
    success: !!user
  });

  return user;
};
```

### Testing the Instrumented Flow Locally

1. **Setup Development Environment**:
   ```bash
   cd apps/dokploy
   npm run dev
   ```

2. **Navigate to Profile Page**: `http://localhost:3000/dashboard/settings/profile`

3. **Expected Log Flow for Profile Page Load**:
   ```
   [USER_API] Profile data requested: { userId: "...", organizationId: "..." }
   [USER_API] Profile data fetched: { userId: "...", hasUserData: true, userEmail: "...", ... }
   [PROFILE] User data loaded: { userId: "...", email: "...", hasImage: true, ... }
   [PROFILE] Available avatars computed: { count: 13, hasGravatar: true, gravatarHash: "..." }
   ```

4. **Expected Log Flow for Profile Update**:
   ```
   [PROFILE] Form submission started: { email: "...", hasPasswordChange: false, selectedImage: "..." }
   [USER_API] Profile update requested: { userId: "...", hasPassword: false, fields: [...], ... }
   [USER_API] Calling updateUser service: { userId: "...", updateFields: [...] }
   [USER_SERVICE] Updating user data: { userId: "...", fields: [...], email: "..." }
   [USER_SERVICE] User update completed: { userId: "...", updatedFields: [...], success: true }
   [USER_API] Profile update completed: { userId: "...", updatedEmail: "...", success: true }
   [PROFILE] Profile update successful: { email: "...", updatedImage: "..." }
   ```

5. **Expected Log Flow for Password Change**:
   ```
   [PROFILE] Form submission started: { email: "...", hasPasswordChange: true, hasCurrentPassword: true, ... }
   [USER_API] Profile update requested: { userId: "...", hasPassword: true, hasCurrentPassword: true, ... }
   [USER_API] Password update requested: { userId: "...", hasNewPassword: true, hasCurrentPassword: true }
   [USER_API] Password validation: { userId: "...", passwordValid: true, hasAccountRecord: true }
   [USER_API] Updating password hash: { userId: "..." }
   [USER_API] Calling updateUser service: { userId: "...", updateFields: [...] }
   [USER_SERVICE] Updating user data: { userId: "...", fields: [...] }
   [USER_SERVICE] User update completed: { userId: "...", success: true }
   [USER_API] Profile update completed: { userId: "...", success: true }
   [PROFILE] Profile update successful: { email: "..." }
   ```

6. **Test Scenarios**:
   - **Avatar Change**: Select different avatar → Monitor client-side avatar computation and server-side image field update
   - **Email Update**: Change email address → Watch Gravatar hash regeneration and email validation
   - **Password Change**: Test password update → Observe password validation flow and bcrypt operations
   - **Gravatar Integration**: Use email with existing Gravatar → Check avatar array expansion with Gravatar URL
   - **Error Cases**: Submit invalid data → Monitor error logging and validation failures

7. **Monitor Console Output**: 
   - **Browser Console**: Client-side `[PROFILE]` logs for form interactions and state changes
   - **Server Terminal**: Server-side `[USER_API]` and `[USER_SERVICE]` logs for API calls and database operations

8. **Database Verification**: Query `users_temp` table to verify data persistence:
   ```sql
   SELECT id, email, image, updated_at FROM users_temp WHERE id = '<user_id>';
   ```

### Key Integration Points
- Better Auth integration for session management (`packages/server/src/lib/auth.ts`)
- Member/Organization relationship handling for multi-tenant access
- tRPC context creation and session validation (`apps/dokploy/server/api/trpc.ts:67-94`)
- Drizzle ORM query patterns with relations and filtering
- React Hook Form integration with TypeScript and Zod validation
