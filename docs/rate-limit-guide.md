# Evernote Backup Rate Limit & Update Handling Guide

This guide covers how evernote-backup handles rate limits and updates that occur in Evernote between sync sessions.

## Rate Limit Handling

### Q: When I get "Rate limit reached. Restart program", what command should I use to retry?

**A:** Simply re-run the same sync command:

```bash
evernote-backup sync
```

### Q: Is my stop point recorded? How does resume work?

**A:** Yes, your stop point is recorded automatically at multiple levels:

1. **USN (Update Sequence Number)** - Main sync progress stored in database config
2. **Individual note content** - Tracks which notes still need content downloaded (`raw_note IS NULL`)  
3. **Linked notebooks** - Separate USN tracking for each shared notebook

The program is designed to automatically resume from exactly where it left off, so no special resume commands are needed.

### Q: What options can help avoid frequent rate limits?

**A:** Use these mitigation options:

```bash
evernote-backup sync --max-download-workers 2 --download-cache-memory-limit 100
```

- `--max-download-workers`: Reduce parallel downloads (default: 4, sane limit: 10)
- `--download-cache-memory-limit`: Reduce memory usage (default: 200MB)
- `--max-chunk-results`: Controls sync chunk size (default: 100)

## Update Handling Between Sync Sessions

### Q: How does the app handle updates that occur in Evernote between sync sessions?

**A:** The app uses a **USN (Update Sequence Number)** mechanism to track changes incrementally. It only fetches changes since the last sync by requesting chunks from the last processed USN to the current remote USN.

### Q: How are new/updated notes detected and processed?

**A:** Through this process:

1. **Detection**: Compare remote USN with local USN stored in database
2. **Fetch**: Request sync chunks from `current_usn` to `remote_usn`  
3. **Store**: Add notes to local database with `raw_note=NULL` initially
4. **Download**: Later download full note content for notes with `raw_note=NULL`
5. **Compress**: Store compressed (lzma + pickle) note content in database

### Q: How are deleted notes handled?

**A:** Two types of deletions:

- **Soft Deletes (Trash)**: Notes have an `is_active` field - when trashed, this becomes `False`
- **Hard Deletes (Expunged)**: Notes are permanently deleted from local database when they appear in `chunk.expungedNotes`

### Q: How are notebook changes handled?

**A:** Notebook changes (renames, deletions, new notebooks) are handled via:

- **Sync Processing**: Uses `SyncChunkFilter` with `includeNotebooks=True`
- **Storage**: `REPLACE` SQL operations automatically handle updates
- **Linked Notebooks**: Have separate USN tracking and are synced independently

### Q: How does the USN mechanism work?

**A:** Key components:

- **Main Account USN**: Stored in config table, tracks overall account changes
- **Linked Notebook USN**: Separate USN for each linked notebook in `notebooks_linked.usn` field
- **Sync Process**: 
  1. Get current remote USN
  2. Compare with local USN
  3. Iterate through chunks from `after_usn`
  4. Update local USN after each chunk

### Q: Are there conflicts if notes are modified both locally and remotely?

**A:** No conflicts occur because evernote-backup is a **read-only backup tool**. It does not modify remote notes, so:

- No conflicts arise from local modifications
- Remote changes simply overwrite local copies
- `replace into` SQL operations handle updates by overwriting existing data

## Technical Details

### Database Progress Tracking

The program uses several database fields to track synchronization progress:

**Config Table Values:**
- `USN`: Tracks the main sync state for user notebooks and notes
- `last_connection_tasks`: Tracks sync state for tasks and reminders (v2 sync API)
- `user`: Stores the authenticated user ID to prevent cross-user sync issues

**Linked Notebook Tracking:**
- `notebooks_linked.usn`: Individual USN tracking for each linked notebook

**Note Download State:**
- `notes.raw_note`: NULL indicates note metadata exists but content hasn't been downloaded yet

### Two-Level Resume Strategy

#### Level 1: Metadata Sync Resume
- Uses USN tracking via Evernote's `getFilteredSyncChunk` API
- Automatically resumes from last processed USN
- Processes notebooks, note metadata, expunges

#### Level 2: Content Download Resume
- Uses `notes.raw_note IS NULL` to identify incomplete downloads
- Downloads only missing note content
- Failed downloads are skipped and retried on next sync

### Error Recovery

- **Network Errors**: Retried up to 5 times
- **Failed Note Downloads**: Skipped and retried on next sync with warning:
  ```
  Note '{note_title}' will be skipped for this run and retried during the next sync.
  ```
- **Rate Limit Errors**: Immediate termination with retry guidance showing time remaining
- **Corrupt Notes**: Marked for re-download

### Task/Reminder Sync Resume

When `--include-tasks` is enabled:
- Uses separate sync mechanism (`iter_sync_chunks_v2()`)
- Tracks tasks and reminders with timestamp-based incremental sync
- Handles expunged tasks/reminders

### Linked/Shared Notebook Updates

- **Authentication**: Each linked notebook requires separate authentication
- **USN Tracking**: Each linked notebook has its own USN tracking
- **Independent Sync**: Synced independently via `iter_linked_notebook_sync_chunks()`
- **Graceful Handling**: Unshared notebooks are handled with warnings

## Key Takeaways

1. **Automatic Resume**: Simply re-running `evernote-backup sync` resumes from the exact stop point
2. **USN Tracking**: Core sync progress is tracked via Evernote's Update Sequence Numbers
3. **Granular Progress**: Note content downloads are tracked individually via `raw_note IS NULL`
4. **Rate Limit Friendly**: The design assumes rate limits will occur and provides graceful recovery
5. **No Conflict Resolution**: The app is read-only, so no conflicts arise from local changes
6. **Incremental Sync**: Only fetches changes since last sync, making it efficient for large accounts

The program is specifically designed to be resilient to interruptions and rate limits, with the database state serving as the single source of truth for what has been completed and what remains to be processed.