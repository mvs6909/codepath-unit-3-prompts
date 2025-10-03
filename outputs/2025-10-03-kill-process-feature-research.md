---
topic: "Adding Kill Process Feature for Running Deployments and Scheduled Jobs"
tags: [research, codebase, deployments, scheduled-jobs, process-management, bullmq]
date: "2025-10-03"
---

# Research: Adding Kill Process Feature for Running Deployments and Scheduled Jobs

**Date**: October 3, 2025 (PST)

## Research Question

How can I add a "Kill process" feature to this application for deployments that are running? For example, I should have a button to cancel a scheduled job if it's taking long to run.

## Summary

The Dokploy application uses **BullMQ** for job queue management and **child processes** (via Node.js spawn) for executing deployments and scheduled tasks. Currently, there's a "Cancel Queues" feature that only removes **waiting or delayed jobs** from the queue, but it **cannot kill actively running processes**. To add a kill process feature, you'll need to:

1. **Track active child process IDs** (PIDs) in the database or in-memory store
2. **Expose BullMQ active job management** to retrieve currently running jobs
3. **Add a new API endpoint** to kill running deployments by sending SIGTERM/SIGKILL signals
4. **Update the UI** to show a "Kill Process" button for running deployments
5. **Handle cleanup** when a process is killed (update deployment status, clean up resources)

## Detailed Findings

### 1. Current Deployment Architecture

#### Queue System (BullMQ)
**Location**: `apps/dokploy/server/queues/`

- **Queue Setup** (`queueSetup.ts:1-44`):
  - Uses BullMQ with Redis connection
  - Queue name: "deployments"
  - `cleanQueuesByApplication()` and `cleanQueuesByCompose()` only handle **waiting/delayed** jobs
  - Currently calls `myQueue.getJobs(["waiting", "delayed"])` - does not include "active" jobs

```typescript
// Current implementation only cancels queued jobs, not active ones
export const cleanQueuesByApplication = async (applicationId: string) => {
  const jobs = await myQueue.getJobs(["waiting", "delayed"]); // Missing "active"
  
  for (const job of jobs) {
    if (job?.data?.applicationId === applicationId) {
      await job.remove();
      console.log(`Removed job ${job.id} for application ${applicationId}`);
    }
  }
};
```

- **Deployment Worker** (`deployments-queue.ts:20-122`):
  - Worker processes jobs from the "deployments" queue
  - Calls deployment functions like `deployApplication`, `deployCompose`, etc.
  - Worker configuration: `{ autorun: false, connection: redisConfig }`
  - Started in `server.ts:64-66` for non-cloud environments

#### Process Spawning
**Location**: `packages/server/src/utils/process/`

- **spawnAsync** (`spawnAsync.ts:8-58`):
  - Wraps Node.js `spawn()` for async child process execution
  - **Key feature**: Returns a promise with `child` property attached
  - The `child` property is the actual `ChildProcess` object that can be killed
  
```typescript
export const spawnAsync = (
  command: string,
  args?: string[] | undefined,
  onData?: (data: string) => void,
  options?: SpawnOptions,
): Promise<BufferList> & { child: ChildProcess } => {
  const child = spawn(command, args ?? [], options ?? {});
  // ... setup stdout/stderr handlers ...
  
  const promise = new Promise<BufferList>((resolve, reject) => {
    // ... error and close handlers ...
  }) as Promise<BufferList> & { child: ChildProcess };
  
  promise.child = child; // ⭐ This can be used to kill the process
  return promise;
};
```

- **execAsyncRemote** (`execAsync.ts:91-160`):
  - For remote server deployments via SSH
  - Uses `ssh2` Client library
  - The `stream` object from `conn.exec()` has no direct kill capability
  - Would need to run a kill command remotely via SSH

#### Deployment Tracking
**Location**: `packages/server/src/db/schema/deployment.ts`

- **Database Schema** (lines 19-66):
  - `deploymentId`: Primary key
  - `status`: Enum with values "running", "done", "error"
  - `logPath`: Path to deployment logs
  - `applicationId` / `composeId` / `scheduleId`: Foreign keys
  - `startedAt` / `finishedAt`: Timestamps
  - **Missing**: No field to store process PID or job ID

```typescript
export const deploymentStatus = pgEnum("deploymentStatus", [
  "running",
  "done",
  "error",
]);
```

### 2. Scheduled Jobs System

**Location**: `apps/schedules/`

- **Queue** (`queue.ts:1-94`):
  - Separate queue: "backupQueue"
  - Uses BullMQ with repeatable jobs (cron patterns)
  - Similar limitation: no active job killing capability

- **Workers** (`workers.ts:7-28`):
  - Two workers for concurrency
  - Concurrency: 50
  - Calls `runJobs()` from utils

- **Job Execution** (`utils.ts:26-149`):
  - `runCommand()` executes scheduled tasks
  - Uses `spawnAsync` for local execution
  - Uses `execAsyncRemote` for remote servers
  - Creates deployment records with status tracking

### 3. Current Cancel Feature (UI)

**Location**: `apps/dokploy/components/dashboard/application/deployments/cancel-queues.tsx`

```tsx
export const CancelQueues = ({ id, type }: Props) => {
  const { mutateAsync, isLoading } =
    type === "application"
      ? api.application.cleanQueues.useMutation()
      : api.compose.cleanQueues.useMutation();
  
  // Only shown for non-cloud deployments
  if (isCloud) {
    return null;
  }
  
  // AlertDialog with "Cancel Queues" button
  // Currently only removes waiting/delayed jobs
}
```

**Limitation**: This only cancels **pending** deployments in the queue, not actively running ones.

### 4. Deployment Status Display

**Location**: `apps/dokploy/components/dashboard/application/deployments/show-deployments.tsx`

- Displays list of last 10 deployments
- Polls every 1 second: `refetchInterval: 1000`
- Shows status with colored indicators:
  - Running: Yellow
  - Done: Green  
  - Error: Red
- Each deployment shows:
  - Status badge
  - Title and description
  - Action buttons (view logs, rollback if available)
  
**Opportunity**: Add "Kill Process" button here for running deployments

### 5. API Routers

**Application Router** (`apps/dokploy/server/api/routers/application.ts:686-699`):
```typescript
cleanQueues: protectedProcedure
  .input(apiFindOneApplication)
  .mutation(async ({ input, ctx }) => {
    // Authorization check
    await cleanQueuesByApplication(input.applicationId);
  })
```

**Deployment Router** (`apps/dokploy/server/api/routers/deployment.ts:1-75`):
- `all`: Get deployments by application
- `allByCompose`: Get deployments by compose
- `allByServer`: Get deployments by server
- `allByType`: Generic query by type
- **Missing**: No endpoint to kill active deployments

## Implementation Strategy

### Phase 1: Track Active Processes

**Option A: In-Memory Store (Simpler, for single instance)**
Create a process registry that maps deployment IDs to child processes:

```typescript
// packages/server/src/utils/process/registry.ts (NEW FILE)
import type { ChildProcess } from "node:child_process";

const processRegistry = new Map<string, ChildProcess>();

export const registerProcess = (deploymentId: string, child: ChildProcess) => {
  processRegistry.set(deploymentId, child);
  
  // Auto-cleanup on process exit
  child.on("exit", () => {
    processRegistry.delete(deploymentId);
  });
};

export const killProcess = (deploymentId: string): boolean => {
  const child = processRegistry.get(deploymentId);
  if (child && !child.killed) {
    child.kill("SIGTERM"); // Graceful shutdown
    
    // Force kill after timeout
    setTimeout(() => {
      if (!child.killed) {
        child.kill("SIGKILL");
      }
    }, 5000);
    
    return true;
  }
  return false;
};

export const getActiveProcesses = (): string[] => {
  return Array.from(processRegistry.keys());
};
```

**Option B: Database Field (Better for distributed systems)**
Add a `processId` field to the deployments table:

```typescript
// Migration needed
export const deployments = pgTable("deployment", {
  // ... existing fields ...
  processId: text("processId"), // Store PID or BullMQ job ID
});
```

**Recommendation**: Start with Option A (in-memory) since deployments run on a single worker instance. If deploying to multiple servers, use Option B with BullMQ job IDs.

### Phase 2: Modify Deployment Functions

Update all deployment functions to register processes:

```typescript
// packages/server/src/services/application.ts
import { registerProcess } from "../utils/process/registry";

export const deployApplication = async ({ applicationId, titleLog, descriptionLog }) => {
  const application = await findApplicationById(applicationId);
  const deployment = await createDeployment({ applicationId, title, description });
  
  try {
    // ... existing code to build command ...
    
    const buildProcess = spawnAsync(/* ... */);
    
    // ⭐ REGISTER THE PROCESS
    registerProcess(deployment.deploymentId, buildProcess.child);
    
    await buildProcess; // Wait for completion
    await updateDeploymentStatus(deployment.deploymentId, "done");
  } catch (error) {
    await updateDeploymentStatus(deployment.deploymentId, "error");
    throw error;
  }
};
```

Apply similar changes to:
- `deployCompose` (compose.ts)
- `deployRemoteApplication` / `rebuildRemoteApplication`
- `runCommand` in schedules (utils.ts)

### Phase 3: Add BullMQ Active Job Support

Extend the cleanQueues to also handle active jobs:

```typescript
// apps/dokploy/server/queues/queueSetup.ts
export const killActiveDeployment = async (
  applicationId: string, 
  deploymentId: string
) => {
  // 1. Try to kill the child process
  const processKilled = killProcess(deploymentId);
  
  // 2. Also remove the BullMQ job if it exists
  const activeJobs = await myQueue.getJobs(["active"]);
  
  for (const job of activeJobs) {
    if (job?.data?.applicationId === applicationId) {
      try {
        await job.moveToFailed(new Error("Killed by user"), true);
      } catch (error) {
        console.error("Failed to move job to failed:", error);
      }
    }
  }
  
  // 3. Update deployment status
  await updateDeploymentStatus(deploymentId, "error");
  await updateApplicationStatus(applicationId, "error");
  
  return processKilled;
};
```

### Phase 4: Add API Endpoint

Create new mutation in application router:

```typescript
// apps/dokploy/server/api/routers/application.ts
killDeployment: protectedProcedure
  .input(z.object({
    applicationId: z.string(),
    deploymentId: z.string(),
  }))
  .mutation(async ({ input, ctx }) => {
    const application = await findApplicationById(input.applicationId);
    
    // Authorization check
    if (application.project.organizationId !== ctx.session.activeOrganizationId) {
      throw new TRPCError({
        code: "UNAUTHORIZED",
        message: "You are not authorized to kill this deployment",
      });
    }
    
    // Kill the process
    const killed = await killActiveDeployment(
      input.applicationId,
      input.deploymentId
    );
    
    if (!killed) {
      throw new TRPCError({
        code: "NOT_FOUND",
        message: "No active process found for this deployment",
      });
    }
    
    return { success: true };
  })
```

Similarly for compose:

```typescript
// apps/dokploy/server/api/routers/compose.ts
killDeployment: protectedProcedure
  .input(z.object({
    composeId: z.string(),
    deploymentId: z.string(),
  }))
  .mutation(async ({ input, ctx }) => {
    // Similar implementation
  })
```

### Phase 5: Update UI Components

Add a "Kill Deployment" button in the deployments list:

```tsx
// apps/dokploy/components/dashboard/application/deployments/kill-deployment-button.tsx (NEW FILE)
import { AlertDialog, AlertDialogAction, AlertDialogCancel, AlertDialogContent, AlertDialogDescription, AlertDialogFooter, AlertDialogHeader, AlertDialogTitle, AlertDialogTrigger } from "@/components/ui/alert-dialog";
import { Button } from "@/components/ui/button";
import { api } from "@/utils/api";
import { XCircle } from "lucide-react";
import { toast } from "sonner";

interface Props {
  deploymentId: string;
  applicationId?: string;
  composeId?: string;
  type: "application" | "compose";
}

export const KillDeploymentButton = ({ deploymentId, applicationId, composeId, type }: Props) => {
  const { mutateAsync: killApplication, isLoading: isKillingApp } = 
    api.application.killDeployment.useMutation();
    
  const { mutateAsync: killCompose, isLoading: isKillingCompose } = 
    api.compose.killDeployment.useMutation();
  
  const isLoading = isKillingApp || isKillingCompose;
  
  const handleKill = async () => {
    try {
      if (type === "application" && applicationId) {
        await killApplication({ applicationId, deploymentId });
      } else if (type === "compose" && composeId) {
        await killCompose({ composeId, deploymentId });
      }
      toast.success("Deployment killed successfully");
    } catch (error) {
      toast.error(error instanceof Error ? error.message : "Failed to kill deployment");
    }
  };
  
  return (
    <AlertDialog>
      <AlertDialogTrigger asChild>
        <Button 
          variant="destructive" 
          size="sm" 
          isLoading={isLoading}
          className="gap-2"
        >
          <XCircle className="size-4" />
          Kill Process
        </Button>
      </AlertDialogTrigger>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>Kill Running Deployment?</AlertDialogTitle>
          <AlertDialogDescription>
            This will immediately terminate the running deployment process. 
            This action cannot be undone and may leave the deployment in an inconsistent state.
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel>Cancel</AlertDialogCancel>
          <AlertDialogAction onClick={handleKill}>
            Kill Process
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
};
```

Update the deployments list to include the button:

```tsx
// apps/dokploy/components/dashboard/application/deployments/show-deployments.tsx
// In the deployment map section (around line 130-215):

{deployments?.map((deployment, index) => (
  <div key={deployment.deploymentId} className="flex items-center justify-between rounded-lg border p-4 gap-2">
    <div className="flex flex-col">
      {/* existing status display */}
    </div>
    <div className="flex flex-col items-end gap-2">
      {/* existing buttons */}
      
      {/* ⭐ ADD THIS */}
      {deployment.status === "running" && (
        <KillDeploymentButton
          deploymentId={deployment.deploymentId}
          applicationId={type === "application" ? id : undefined}
          composeId={type === "compose" ? id : undefined}
          type={type}
        />
      )}
    </div>
  </div>
))}
```

### Phase 6: Handle Remote Deployments

For remote server deployments (via SSH), the process runs on the remote machine. To kill it:

```typescript
// packages/server/src/utils/process/remote-kill.ts (NEW FILE)
import { execAsyncRemote } from "./execAsync";

export const killRemoteProcess = async (
  serverId: string,
  deploymentId: string
): Promise<boolean> => {
  try {
    // Try to find and kill the process by deployment ID (if stored in process name/args)
    const command = `
      # Find the process (example: by log path pattern)
      PID=$(ps aux | grep "${deploymentId}" | grep -v grep | awk '{print $2}')
      
      if [ -n "$PID" ]; then
        echo "Killing process $PID"
        kill -TERM $PID
        
        # Wait a bit, then force kill if still running
        sleep 5
        kill -9 $PID 2>/dev/null || true
        
        echo "Process killed"
        exit 0
      else
        echo "Process not found"
        exit 1
      fi
    `;
    
    await execAsyncRemote(serverId, command);
    return true;
  } catch (error) {
    console.error("Failed to kill remote process:", error);
    return false;
  }
};
```

Update `killActiveDeployment` to check if it's a remote deployment:

```typescript
export const killActiveDeployment = async (
  applicationId: string, 
  deploymentId: string
) => {
  const application = await findApplicationById(applicationId);
  const deployment = await findDeploymentById(deploymentId);
  
  let processKilled = false;
  
  if (application.serverId) {
    // Remote deployment
    processKilled = await killRemoteProcess(application.serverId, deploymentId);
  } else {
    // Local deployment
    processKilled = killProcess(deploymentId);
  }
  
  // ... rest of the logic
};
```

## Architecture Insights

### Key Design Patterns

1. **Queue-based Architecture**: BullMQ manages deployment jobs with Redis as the backing store
2. **Worker Pattern**: Separate worker processes handle deployment execution
3. **Process Spawning**: Child processes execute actual build/deploy commands
4. **Status Tracking**: Database records track deployment lifecycle
5. **Real-time Updates**: WebSocket connections stream logs to UI

### Current Limitations

1. **No Process Tracking**: Child processes aren't associated with deployment records
2. **Queue-only Cancellation**: Can only cancel queued jobs, not running ones
3. **No Timeout Handling**: Long-running deployments can't be auto-killed
4. **Remote Process Management**: SSH-based deployments harder to kill without PID tracking

### Recommended Improvements

1. **Add Process Registry**: Track active child processes in memory or Redis
2. **Store Job/Process IDs**: Add fields to deployment schema
3. **Implement Timeouts**: Auto-kill deployments exceeding time limits
4. **Better Remote Management**: Store remote PIDs or use Docker container IDs
5. **Graceful Shutdown**: Always try SIGTERM before SIGKILL
6. **Audit Logging**: Log who killed which deployment and when

## Code References

### Core Files

- `apps/dokploy/server/queues/queueSetup.ts:22-42` - Current cleanQueues implementation
- `apps/dokploy/server/queues/deployments-queue.ts:20-122` - Deployment worker
- `packages/server/src/utils/process/spawnAsync.ts:8-58` - Process spawning utility
- `packages/server/src/db/schema/deployment.ts:19-66` - Deployment schema
- `packages/server/src/services/application.ts:256-400` - Application deployment logic
- `packages/server/src/services/compose.ts:193-419` - Compose deployment logic
- `packages/server/src/utils/schedules/utils.ts:26-149` - Scheduled job execution
- `apps/dokploy/components/dashboard/application/deployments/cancel-queues.tsx:22-72` - Current cancel UI
- `apps/dokploy/components/dashboard/application/deployments/show-deployments.tsx:42-228` - Deployments list UI

### Related Services

- `apps/schedules/src/queue.ts:1-94` - Scheduled jobs queue
- `apps/schedules/src/workers.ts:7-28` - Scheduled jobs workers
- `apps/dokploy/server/api/routers/application.ts:686-699` - Application API router
- `apps/dokploy/server/api/routers/compose.ts:219-230` - Compose API router
- `apps/dokploy/server/api/routers/deployment.ts:1-75` - Deployment API router

## Open Questions

1. **Multi-instance Support**: How should process tracking work if deployed across multiple servers?
   - Consider using BullMQ job IDs instead of local PIDs
   - Store in Redis for cross-instance access

2. **Cleanup After Kill**: What should happen to Docker containers/volumes after killing?
   - Should we run cleanup scripts?
   - Mark as "cancelled" vs "error"?

3. **User Permissions**: Should all users be able to kill deployments, or only admins?
   - Current cleanQueues is protected by organization membership
   - Consider adding role-based access control

4. **Scheduled Jobs**: Should scheduled job killing be different from deployments?
   - Same mechanism can apply
   - May want separate UI placement

5. **Cloud Deployments**: How does this work with `IS_CLOUD` flag?
   - Cloud deployments use different queue (`deploy()` directly)
   - May need separate implementation for cloud

6. **Notification**: Should users be notified when a deployment is killed?
   - Email/webhook notifications?
   - Log in deployment history?

## Next Steps

1. **Proof of Concept**: Implement the in-memory process registry
2. **Add API Endpoint**: Create `killDeployment` mutation
3. **Update UI**: Add kill button to running deployments
4. **Test Locally**: Verify it works for local deployments
5. **Handle Remote**: Implement remote process killing via SSH
6. **Add Tests**: Unit and integration tests for kill functionality
7. **Documentation**: Update user docs with new feature

## Related Issues/PRs

*(This section would link to any existing GitHub issues or pull requests related to this feature)*

## Contributors

- Research conducted by: AI Assistant
- For: Dokploy Project

