# Backblaze B2 Compatibility Fix - Add --disable-tagging Flag

## Problem Description

When using pmoxs3backuproxy with Backblaze B2, backups would complete successfully but file-level restores would fail with the error:

```
unable to parse raw blob - wrong magic (500)
```

### Root Cause

The issue was caused by corruption of `index.json.blob` files. Investigation revealed that:

1. **Backblaze B2 does not support S3 object tagging APIs**
   - `GetObjectTagging` returns empty tags (no error, but no data)
   - `PutObjectTagging` is explicitly rejected with an error

2. **pmoxs3backuproxy was calling tagging APIs without proper error handling**
   - When `ReadTags()` failed, errors were silently ignored: `existingTags, _ := ss.ReadTags(*C.Client)`
   - This led to nil map operations and corrupted blob files
   - The corruption manifested as tagging XML being written to index.json.blob instead of actual blob content:
     ```xml
     <Tagging><TagSet><Tag><Key>note</Key><Value>cmVwb3NvbGl0ZQ</Value></Tag></TagSet></Tagging>
     ```

3. **Impact**
   - VM backups completed successfully (chunks and index files were created)
   - Configuration file backups worked (qemu-server.conf.blob, etc.)
   - Only index.json.blob was corrupted
   - File-level restore operations failed completely
   - Full VM restores were likely affected as well

## Solution

Added a `--disable-tagging` flag to completely disable all S3 object tagging operations for providers that don't support it.

### Implementation Details

#### 1. Command-Line Flag Addition

**Files Modified:**
- `cmd/pmoxs3backuproxy/main.go` (line 124)
- `cmd/garbagecollector/main.go` (line 84)

```go
disableTagging := flag.Bool("disable-tagging", false, "Disable S3 object tagging (required for Backblaze B2)")
```

#### 2. Server Configuration

**File:** `cmd/pmoxs3backuproxy/types.go` (line 59)

Added `DisableTagging` field to Server struct:
```go
type Server struct {
    // ... existing fields ...
    LookupTypeFlag    string
    DisableTagging    bool
}
```

#### 3. Core Function Updates

**File:** `internal/s3pmoxcommon/s3pmoxcommon.go`

**ReadTags() - Skip API calls when disabled:**
```go
func (S *Snapshot) ReadTags(c minio.Client, disableTagging bool) (map[string]string, error) {
    if disableTagging {
        s3backuplog.DebugPrint("Tagging disabled, returning empty tags")
        return make(map[string]string), nil
    }
    // ... existing GetObjectTagging code ...
}
```

**ListSnapshots() - Skip reading UserTags when disabled:**
```go
func ListSnapshots(c minio.Client, datastore string, returnCorrupted bool, disableTagging bool) ([]Snapshot, error) {
    // ... existing code ...
    if !disableTagging {
        if object.UserTags["protected"] == "true" {
            existing_S.Protected = true
        }
        if object.UserTags["note"] != "" {
            note, _ := base64.RawStdEncoding.DecodeString(object.UserTags["note"])
            existing_S.Comment = string(note)
        }
    }
    // ... rest of code ...
}
```

**GetLatestSnapshot() - Pass disableTagging parameter:**
```go
func GetLatestSnapshot(c minio.Client, ds string, id string, time uint64, disableTagging bool) (*Snapshot, error) {
    snapshots, err := ListSnapshots(c, ds, false, disableTagging)
    // ... rest of code ...
}
```

#### 4. API Endpoint Updates

**File:** `cmd/pmoxs3backuproxy/main.go`

**PUT /notes endpoint - Skip PutObjectTagging when disabled:**
```go
if s.DisableTagging {
    s3backuplog.WarnPrint("Tagging disabled - notes feature unavailable for snapshot: %s", ss.S3Prefix())
    w.WriteHeader(http.StatusOK)
    return
}
```

**PUT /protected endpoint - Skip PutObjectTagging when disabled:**
```go
if s.DisableTagging {
    s3backuplog.WarnPrint("Tagging disabled - protection feature unavailable for snapshot: %s", ss.S3Prefix())
    w.Header().Add("Content-Type", "application/json")
    resp, _ := json.Marshal(Response{Data: ss})
    w.Write(resp)
    return
}
```

#### 5. Function Call Updates

Updated all function calls throughout the codebase to pass the `disableTagging` parameter:
- All `ReadTags()` calls now pass `s.DisableTagging` or `disableTagging`
- All `ListSnapshots()` calls now pass `s.DisableTagging` or `disableTagging`
- Both `GetLatestSnapshot()` calls now pass `s.DisableTagging`

## Files Modified

1. **cmd/pmoxs3backuproxy/main.go**
   - Added `--disable-tagging` flag
   - Updated Server initialization with DisableTagging field
   - Modified PUT /notes endpoint to skip tagging when disabled
   - Modified PUT /protected endpoint to skip tagging when disabled
   - Updated all ReadTags(), ListSnapshots(), and GetLatestSnapshot() calls

2. **cmd/pmoxs3backuproxy/types.go**
   - Added `DisableTagging bool` field to Server struct

3. **internal/s3pmoxcommon/s3pmoxcommon.go**
   - Updated `ListSnapshots()` signature and implementation
   - Updated `GetLatestSnapshot()` signature and implementation
   - Updated `ReadTags()` signature and implementation

4. **cmd/garbagecollector/main.go**
   - Added `--disable-tagging` flag
   - Updated ListSnapshots() call

5. **README.md**
   - Documented the new --disable-tagging flag for both tools
   - Added usage note about S3 providers without tagging support
   - Explained feature degradation when tagging is disabled

## Testing Results

### Test Environment
- S3 Provider: Backblaze B2
- Test VM: "reposilite"
- Operations tested:
  - Full VM backup
  - Incremental VM backup
  - File-level restore
  - Garbage collection

### Before Fix
- ❌ index.json.blob corrupted with tagging XML
- ❌ File-level restore failed with "wrong magic (500)" error
- ❌ Other blob files worked correctly, only index.json.blob affected

### After Fix (with --disable-tagging flag)
- ✅ Backups complete successfully
- ✅ index.json.blob contains correct blob content
- ✅ File-level restore works perfectly
- ✅ Garbage collector runs without errors
- ✅ All backup operations function correctly

### Known Limitations with --disable-tagging
- Notes feature unavailable (returns warning, doesn't save notes)
- Protection feature unavailable (returns warning, doesn't protect from garbage collection)
- Garbage collector treats all backups as unprotected
- These features require S3 object tagging support and cannot work on B2

## Usage Instructions

### For pmoxs3backuproxy

```bash
./pmoxs3backuproxy \
  -endpoint s3.us-west-004.backblazeb2.com \
  -bind 127.0.0.1:8007 \
  -usessl \
  -disable-tagging \
  -debug
```

### For garbagecollector

```bash
./garbagecollector \
  -endpoint s3.us-west-004.backblazeb2.com \
  -accesskey YOUR_KEY_ID \
  -secretkey YOUR_SECRET_KEY \
  -bucket YOUR_BUCKET \
  -usessl \
  -disable-tagging \
  -retention 60 \
  -debug
```

**Important:** Remember that boolean flags in Go must not have values after them:
- ✅ Correct: `-usessl -disable-tagging`
- ✅ Also correct: `-usessl=true -disable-tagging=true`
- ❌ Wrong: `-usessl true -disable-tagging true` (treats "true" as positional argument)

## Build Instructions

Built using Docker with Go 1.22:

```bash
docker run --rm -v "$(pwd)":/workspace -w /workspace golang:1.22 \
  go build -buildvcs=false -o pmoxs3backuproxy ./cmd/pmoxs3backuproxy

docker run --rm -v "$(pwd)":/workspace -w /workspace golang:1.22 \
  go build -buildvcs=false -o garbagecollector ./cmd/garbagecollector
```

## Commit History

1. **Initial implementation commit** (e632bf5)
   - Added --disable-tagging flag
   - Modified all tagging-related functions
   - Updated API endpoints

2. **Documentation commit** (pending)
   - Updated README.md with usage instructions
   - Updated .gitignore to exclude binaries
   - Added this CLAUDE.md file

## Future Considerations

1. **Auto-detection:** Could potentially detect B2 or other providers and automatically disable tagging, but current approach with explicit flag is safer and more predictable.

2. **Alternative storage for notes/protection:** Could implement a separate metadata storage mechanism (e.g., separate JSON files in S3) for providers without tagging support, but this would add complexity.

3. **Testing:** Consider adding CI tests specifically for --disable-tagging mode to ensure compatibility doesn't break in future updates.

## Credits

Implementation developed collaboratively between user (Steven Hayes) and Claude (Anthropic) on 2026-01-12.

Issue identified and root cause analyzed through examination of:
- Corrupted index.json.blob files
- Backblaze B2 API documentation
- pmoxs3backuproxy source code
- Error handling patterns in ReadTags() calls
