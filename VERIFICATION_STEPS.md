# Verification Steps for cursor-agent PATH Fix

## Quick Verification

After restarting Vibe Kanban with `pnpm run start:local`:

1. **Check cursor-agent is still installed and in your PATH:**
   ```bash
   which cursor-agent
   # Expected: /home/jesseboudreau/.local/bin/cursor-agent
   
   ls -l ~/.local/bin/cursor-agent
   # Expected: symlink to the actual binary
   ```

2. **Start Vibe Kanban:**
   ```bash
   pnpm run start:local
   ```

3. **Create a task attempt using CursorAgent:**
   - Open the UI at http://localhost:3000
   - Create/open a task
   - Select "CursorAgent" as the executor
   - Start a task attempt with a simple prompt like "List files in the current directory"

4. **Verify success:**
   - The task should NOT show "Executable 'cursor-agent' not found in PATH"
   - The backend logs should NOT contain the error message
   - CursorAgent should successfully start (you may still see auth errors if not logged in, but that's expected and different)
   - The UI should show CursorAgent as "Installation Found" (not "Not Found")

## Expected Outputs

### Before the fix:
- Backend log: `Failed to start task attempt: Executable 'cursor-agent' not found in PATH`
- UI shows: CURSOR_AGENT status as "Not Found"

### After the fix:
- Backend log: Normal CursorAgent startup messages (or auth errors if not logged in)
- UI shows: CURSOR_AGENT status as "Installation Found"
- Task attempts should start successfully

## Testing Auth Flow (Optional)

If cursor-agent requires authentication:
1. The error will now be about authentication, not PATH
2. Run: `cursor-agent login`
3. Complete the browser-based authentication
4. Retry the task attempt

## Debugging if Still Not Working

If you still see PATH errors:

1. **Check PATH in the server process:**
   ```bash
   # Add temporary logging to see what PATH the server has
   echo $PATH
   ```

2. **Verify the server inherited PATH correctly:**
   ```bash
   # Check if ~/.local/bin is in the server's PATH
   # The server inherits from the shell that starts it
   ```

3. **Restart from a fresh shell:**
   ```bash
   # Make sure .bashrc is sourced
   source ~/.bashrc
   cd /home/jesseboudreau/projects/vibe-kanban
   pnpm run start:local
   ```

4. **Check if PATH is being overridden:**
   - Look for any env files (.env, .env.local) that might set PATH
   - Check scripts/start-local.sh for PATH modifications

## Implementation Details

The fix ensures that `ExecutionEnv` (in `crates/executors/src/env.rs`) now:
- Inherits PATH from the parent process when created via `ExecutionEnv::new()`
- Passes this PATH to all executor child processes via `Command.env("PATH", ...)`
- Makes cursor-agent and other locally installed binaries discoverable

This is a permanent fix that persists across restarts.
