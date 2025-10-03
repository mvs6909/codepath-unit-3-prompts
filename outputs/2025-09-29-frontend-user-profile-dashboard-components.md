---
date: 2025-09-29T18:45:00.000Z
researcher: Claude
topic: "Frontend components for user profile and dashboard - custom avatar and optional custom fieldname features"
tags: [research, codebase, frontend, user-profile, dashboard, avatar, custom-fields]
status: complete
last_updated: 2025-09-29
last_updated_by: Claude
---

# Research: Frontend components for user profile and dashboard - custom avatar and optional custom fieldname features

**Date**: 2025-09-29T18:45:00.000Z  
**Researcher**: Claude  
**Git Commit**: 11acfb51878eba2d9b875387aae728e353c067b8  
**Branch**: manavshah/prompts  
**Repository**: dokploy

## Research Question
Can you please help me research the frontend components for user profile and dashboard. I would be working on a few features like adding a custom avatar for a user profile, and adding a custom fieldname that is optional

## Summary
The Dokploy application has a well-structured frontend architecture for user profile and dashboard management. The current system supports avatar selection from predefined options and Gravatar integration, with a robust form-based profile management system. The database schema and API infrastructure are already set up to handle custom user fields, making it feasible to add custom avatar uploads and optional custom fieldnames.

## Detailed Findings

### User Profile Components

#### Profile Form Component (`apps/dokploy/components/dashboard/settings/profile/profile-form.tsx`)
- **Primary Component**: `ProfileForm` - Main user profile editing interface
- **Schema Definition**: Uses Zod schema for validation with fields: `email`, `password`, `currentPassword`, `image`, `allowImpersonation`
- **Current Avatar System**: 
  - 12 predefined avatar images (`/avatars/avatar-1.png` through `/avatars/avatar-12.png`)
  - Gravatar integration using SHA256 hash of user email
  - Radio button selection interface for avatar choosing
- **Form Framework**: React Hook Form with Zod resolver for validation
- **API Integration**: Uses tRPC `api.user.update.useMutation()` for profile updates

#### Avatar Display Components
- **User Navigation**: `apps/dokploy/components/layouts/user-nav.tsx` - Displays avatar in sidebar using `Avatar`, `AvatarImage`, `AvatarFallback` components
- **Impersonation Bar**: `apps/dokploy/components/dashboard/impersonation/impersonation-bar.tsx` - Shows user avatar during impersonation
- **UI Components**: `apps/dokploy/components/ui/avatar.tsx` - Reusable avatar component built on Radix UI

### Database Schema and Data Layer

#### User Schema (`packages/server/src/db/schema/user.ts`)
- **Table**: `users_temp` (PostgreSQL table using Drizzle ORM)
- **Avatar Field**: `image: text("image")` - Currently stores avatar URL/path
- **Extensible Design**: Schema allows easy addition of new fields
- **Key Fields**: `id`, `name`, `email`, `image`, `role`, `allowImpersonation`, and many other optional fields

#### API Layer (`apps/dokploy/server/api/routers/user.ts`)
- **Update Endpoint**: `user.update` mutation with validation using `apiUpdateUser` schema
- **Validation**: Password validation, current password verification
- **Authentication**: Protected procedure requiring valid session

### Dashboard Layout Architecture

#### Layout Components
- **Main Layout**: `apps/dokploy/components/layouts/dashboard-layout.tsx` - Wrapper for dashboard pages
- **Sidebar**: `apps/dokploy/components/layouts/side.tsx` - Complex navigation with role-based menu filtering
- **Navigation Structure**: Hierarchical menu system with projects, monitoring, settings sections

#### UI Component Patterns
- **Form System**: Comprehensive form components in `apps/dokploy/components/ui/form.tsx`
- **Input Components**: Reusable input, textarea, select components with error handling
- **Card Layout**: Settings pages use Card components for consistent layout
- **Responsive Design**: Mobile-friendly layouts with proper breakpoints

### Authentication and User Management

#### Better Auth Integration (`packages/server/src/lib/auth.ts`)
- **Framework**: Uses Better Auth library with Drizzle adapter
- **User Model**: `users_temp` table with additional fields for role, avatar, permissions
- **Session Management**: Comprehensive session handling with organization context
- **Social Auth**: GitHub and Google OAuth integration

## Code References

### Core Profile Components
- `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:34-40` - Profile schema definition
- `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:44-57` - Avatar images array
- `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:73-78` - Avatar selection logic
- `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx:225-266` - Avatar selection UI

### Database and API
- `packages/server/src/db/schema/user.ts:46` - User image field definition
- `packages/server/src/db/schema/user.ts:298-326` - User update validation schema
- `apps/dokploy/server/api/routers/user.ts:145-178` - User update mutation
- `packages/server/src/services/user.ts:240-251` - User update service function

### UI Components
- `apps/dokploy/components/ui/avatar.tsx:1-48` - Avatar component implementation
- `apps/dokploy/components/ui/form.tsx:1-176` - Form component system
- `apps/dokploy/components/layouts/user-nav.tsx:44-50` - User navigation avatar display

## Architecture Insights

### Current Patterns
1. **Component Structure**: Well-organized component hierarchy with clear separation of concerns
2. **Type Safety**: Comprehensive TypeScript usage with Zod validation throughout
3. **Form Handling**: Consistent React Hook Form + Zod pattern for all forms
4. **State Management**: tRPC for server state, React Hook Form for form state
5. **Styling**: Tailwind CSS with shadcn/ui component library

### Design Decisions
1. **Avatar Storage**: Currently stores URLs/paths as strings rather than handling file uploads
2. **Validation**: Client and server-side validation using shared Zod schemas
3. **Permissions**: Role-based access control integrated throughout the UI
4. **Responsive Design**: Mobile-first approach with proper breakpoints

## Implementation Cases for Future Development

### Case 1: Add Custom Avatar with Initials
**Goal**: Add a dynamically generated avatar option using user's initials from email or name in the current avatar selection list.

**Implementation Strategy**:
1. **Avatar Generation Logic**: 
   - Create utility function to generate initials from user's name or email
   - Generate colored background based on user data (consistent color per user)
   - Use Canvas API or SVG to create avatar image dynamically
2. **Integration with Current System**:
   - Add generated avatar to `availableAvatars` array in `ProfileForm` component
   - Modify avatar selection UI to include the custom initials option
   - Update `AvatarImage` and `AvatarFallback` components to handle generated avatars
3. **Data Layer**:
   - No database changes needed - use special identifier like "initials" as image value
   - Update validation to allow "initials" as valid image option
4. **Display Components**:
   - Modify `user-nav.tsx` and `impersonation-bar.tsx` to render initials avatar
   - Ensure consistent rendering across all avatar display locations

**Key Files to Modify**:
- `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx` - Add initials option
- `apps/dokploy/components/ui/avatar.tsx` - Handle initials rendering
- `apps/dokploy/components/layouts/user-nav.tsx` - Display initials avatar
- Add new utility: `apps/dokploy/lib/avatar-utils.ts` - Generate initials and colors

### Case 2: Add Optional Name Field to Profile
**Goal**: Add an optional custom display name field that users can set in their profile.

**Implementation Strategy**:
1. **Database Schema Extension**:
   - Add `displayName: text("displayName")` field to `users_temp` table
   - Create database migration for the new field
2. **API Layer Updates**:
   - Extend `profileSchema` and `apiUpdateUser` Zod schemas to include `displayName`
   - Update user update mutation to handle new field
   - Modify user query responses to include display name
3. **Frontend Form Integration**:
   - Add new `FormField` for display name in `ProfileForm` component
   - Position after email field, before password fields
   - Add appropriate validation (optional, max length, character restrictions)
4. **Display Logic Updates**:
   - Update user navigation to show display name when available
   - Modify breadcrumbs and user mentions to use display name
   - Fallback to email or existing name when display name is not set
5. **User Experience**:
   - Add help text explaining the purpose of display name
   - Show preview of how name will appear in the interface

**Key Files to Modify**:
- `packages/server/src/db/schema/user.ts` - Add displayName field
- Database migration file - Add column to users_temp table
- `apps/dokploy/components/dashboard/settings/profile/profile-form.tsx` - Add form field
- `apps/dokploy/components/layouts/user-nav.tsx` - Display custom name
- `apps/dokploy/server/api/routers/user.ts` - Handle field in updates

## Open Questions

### For Case 1 (Initials Avatar):
1. **Color Generation**: What algorithm should be used to generate consistent colors for initials avatars?
2. **Initials Logic**: Should initials be derived from name field, email, or both with fallback?
3. **Rendering Method**: Canvas API vs SVG vs CSS-based solution for generating initials avatars?

### For Case 2 (Optional Name Field):
1. **Field Naming**: Should it be `displayName`, `customName`, or another field name?
2. **Character Limits**: What length restrictions should apply to the custom name field?
3. **Display Priority**: When should custom display name take precedence over existing name/email?
4. **Migration Strategy**: How should existing users be handled when adding the new optional field?

## Related Research
- Profile management follows similar patterns to other settings components in `apps/dokploy/components/dashboard/settings/`
- Form validation patterns are consistent with other forms throughout the application
- The codebase shows good separation of concerns between UI, API, and database layers
