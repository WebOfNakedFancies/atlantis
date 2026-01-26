# Branch Mode MVP Implementation

## Overview
This document describes the MVP implementation for triggering Atlantis plans directly on branches without requiring a Pull Request.

## Implementation Approach
We implemented **Option 1: Minimal Change (Synthetic PR)** using `PR=0` as a sentinel value to indicate branch-based plans.

## How It Works

### API Usage
To trigger a plan on a branch without a PR, simply omit the `PR` field or set it to `0`.

**Note**: The `Ref` parameter should be a **branch name** (e.g., `"main"`, `"develop"`). Commit SHAs are not currently supported.

```bash
curl --request POST 'https://<ATLANTIS_HOST_NAME>/api/plan' \
--header 'X-Atlantis-Token: <ATLANTIS_API_SECRET>' \
--header 'Content-Type: application/json' \
--data-raw '{
    "Repository": "owner/repo-name",
    "Ref": "main",
    "Type": "Github",
    "Paths": [{
      "Directory": ".",
      "Workspace": "default"
    }]
}'
```

### Key Changes Made

1. **API Controller** (`server/controllers/api_controller.go`)
   - Modified `apiParseAndValidate()` to handle `PR=0` case
   - When `PR=0`, uses `Ref` for both head and base branches
   - Added conditional VCS status updates (skipped when `PR=0`)
   - Added TODO comments marking areas for future refactoring

2. **VCS Status Updates**
   - Skipped when `PR=0` to avoid errors
   - Applies to both plan and apply endpoints
   - Logged as "branch mode (PR=0)" in debug messages

3. **Documentation** (`runatlantis.io/docs/api-endpoints.md`)
   - Updated parameter descriptions
   - Added branch mode examples
   - Documented limitations

## Known Limitations

### 1. Merge Strategy Incompatibility (FIXED)
- **Issue**: When Atlantis is configured with merge checkout strategy (default), it expects to merge a PR branch into a base branch
- **Impact**: Branch mode (PR=0) tried to "merge" a branch with itself (e.g., merge `main` into `main`), causing git errors
- **Solution**: The code now automatically uses branch checkout strategy when `PR=0`, bypassing the merge logic
- **Code location**: `working_dir.go` checks `c.pr.Num == 0` to force branch strategy in both `updateToRef()` and `forceClone()`

### 2. Locking Collisions
- Multiple concurrent branch plans all use `PR=0` as the lock key
- If running plans on different branches simultaneously, they may interfere
- **Workaround**: Serialize branch-based plans or use PR-based workflow for concurrent plans
- **Future Fix**: Implement branch-specific locking (Option 2)

### 3. No VCS Status Updates
- Commit status checks are not updated when using branch mode
- Users won't see plan status in their VCS UI
- **Workaround**: Check Atlantis logs/UI directly
- **Future Fix**: Add branch-based commit status support

### 4. Lock Cleanup
- `UnlockByPull(repo, 0)` unlocks ALL branch-mode locks for that repo
- May affect unrelated branch plans
- **Workaround**: Ensure plans complete before starting new ones
- **Future Fix**: Use branch-specific lock keys

### 5. Commit SHA Not Supported
- **Limitation**: The `Ref` parameter only accepts branch names, not commit SHAs
- **Issue**: Using a SHA as `Ref` will fail during git operations
- **Workaround**: Use branch names or tags only
- **Future Fix**: Implement SHA support with fallback clone logic

## Future Improvements (Option 2)

When this feature proves valuable, consider refactoring to explicit branch mode:

1. Add `BranchMode bool` field to `APIRequest`
2. Implement branch-based locking (key: `repo/branch` instead of `repo/PR`)
3. Add branch-specific commit status support
4. Create separate storage for branch plans vs PR plans
5. Add branch-based lock management UI

## Testing

### Manual Testing
```bash
# Test plan on main branch
curl -X POST http://localhost:4141/api/plan \
  -H "X-Atlantis-Token: $API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "Repository": "org/repo",
    "Ref": "main",
    "Type": "Github",
    "Paths": [{"Directory": ".", "Workspace": "default"}]
  }'

# Test apply on main branch
curl -X POST http://localhost:4141/api/apply \
  -H "X-Atlantis-Token: $API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "Repository": "org/repo",
    "Ref": "main",
    "Type": "Github",
    "Paths": [{"Directory": ".", "Workspace": "default"}]
  }'
```

### Expected Behavior
- ✅ Plan executes successfully
- ✅ Repo is cloned at specified ref
- ✅ Terraform plan runs
- ✅ Locks are acquired and released
- ❌ VCS commit status NOT updated (expected)
- ❌ Concurrent branch plans may conflict (known limitation)

## Migration Path

If this feature is successful and you want to properly productionize it:

1. **Phase 1 (Current)**: Use MVP with documented limitations
2. **Phase 2**: Gather user feedback and usage patterns
3. **Phase 3**: Implement Option 2 if validated (1-2 weeks effort)
4. **Phase 4**: Add advanced features (scheduled branch plans, webhooks, etc.)

## Code Locations

- API Controller: `server/controllers/api_controller.go`
- Documentation: `runatlantis.io/docs/api-endpoints.md`
- Locking: `server/core/locking/locking.go`
- Working Dir: `server/events/working_dir.go`

## Support

For issues or questions:
1. Check logs for "branch mode" messages
2. Verify PR=0 or PR field is omitted
3. Ensure single branch plan at a time (to avoid lock conflicts)
4. Review TODO comments in code for context
