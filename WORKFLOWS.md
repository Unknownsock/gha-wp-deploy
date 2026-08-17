# Workflow files

Copy each block below into the new repo at the path shown above it (use
GitHub's web "Add file → Create new file" if you're not pushing from a local
clone). The same content also exists as real files in this folder
(`.github/workflows/deploy.yml`, `.github/workflows/remote-cleanup.yml`) if
you'd rather push the folder directly with git.

## `.github/workflows/deploy.yml`

```yaml
name: Deploy WordPress theme via FTP (reusable)

on:
    workflow_call:
        inputs:
            environment:
                description: 'staging | production (used to look up config in deploy.json and pick secret names)'
                required: true
                type: string
            node-version:
                required: false
                type: string
                default: '22'
            build-command:
                required: false
                type: string
                default: 'npm run prod'
            config-file:
                description: 'Path to deploy.json in the caller repo'
                required: false
                type: string
                default: '.github/config/deploy.json'
            job-timeout-minutes:
                required: false
                type: number
                default: 18
            deploy-timeout-minutes:
                required: false
                type: number
                default: 8

jobs:
    deploy:
        runs-on: ubuntu-latest
        timeout-minutes: ${{ inputs.job-timeout-minutes }}
        env:
            FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true

        steps:
            - name: Checkout caller repository
              uses: actions/checkout@v5
              with:
                  fetch-depth: 2 # Enough for single-commit pushes

            - name: Fetch push range
              if: github.event.before != '0000000000000000000000000000000000000000'
              run: git fetch --deepen=50 || git fetch --unshallow

            - name: Setup Node.js
              uses: actions/setup-node@v5
              with:
                  node-version: ${{ inputs.node-version }}
                  cache: 'npm'

            - name: Install dependencies
              run: npm ci

            - name: Build assets
              run: ${{ inputs.build-command }}

            - name: Deploy via FTP
              timeout-minutes: ${{ inputs.deploy-timeout-minutes }}
              env:
                  ENVIRONMENT: ${{ inputs.environment }}
                  CONFIG_FILE: ${{ inputs.config-file }}
                  BEFORE_SHA: ${{ github.event.before }}
                  STAGING_FTP_USERNAME: ${{ secrets.STAGING_FTP_USERNAME }}
                  STAGING_FTP_PASSWORD: ${{ secrets.STAGING_FTP_PASSWORD }}
                  PRODUCTION_FTP_USERNAME: ${{ secrets.PRODUCTION_FTP_USERNAME }}
                  PRODUCTION_FTP_PASSWORD: ${{ secrets.PRODUCTION_FTP_PASSWORD }}
              run: |
                  sudo apt-get install -y -qq lftp jq

                  set -e

                  TMPFILES=()
                  cleanup() { rm -f "${TMPFILES[@]}"; }
                  trap cleanup EXIT

                  WORKSPACE_DIR=$(pwd)

                  if [ ! -f "$CONFIG_FILE" ]; then
                    echo "❌ Config file not found: $CONFIG_FILE"
                    exit 1
                  fi

                  echo "🌐 Environment: $ENVIRONMENT"
                  echo ""

                  # Check if deployment is enabled
                  ENABLED=$(jq -r ".${ENVIRONMENT}.enabled" "$CONFIG_FILE")
                  if [ "$ENABLED" != "true" ]; then
                    echo "⏭️  Deployment disabled for $ENVIRONMENT in deploy.json"
                    exit 0
                  fi

                  REMOTE_PATH=$(jq -r ".${ENVIRONMENT}.remotePath" "$CONFIG_FILE")
                  FTP_HOST=$(jq -r ".${ENVIRONMENT}.ftpHost" "$CONFIG_FILE")
                  PROTOCOL=$(jq -r ".${ENVIRONMENT}.protocol // \"ftp\"" "$CONFIG_FILE")
                  UPLOAD_ALL=$(jq -r ".${ENVIRONMENT}.uploadAll // false" "$CONFIG_FILE")
                  CLEAN=$(jq -r ".${ENVIRONMENT}.clean // false" "$CONFIG_FILE")
                  CHUNK_SIZE=$(jq -r ".${ENVIRONMENT}.chunkSize // 0" "$CONFIG_FILE")

                  # Resolve credentials
                  ENV_UPPER=$(echo "$ENVIRONMENT" | tr '[:lower:]' '[:upper:]')
                  USERNAME_VAR="${ENV_UPPER}_FTP_USERNAME"
                  PASSWORD_VAR="${ENV_UPPER}_FTP_PASSWORD"
                  USERNAME="${!USERNAME_VAR}"
                  PASSWORD="${!PASSWORD_VAR}"

                  if [ -z "$FTP_HOST" ]; then
                    echo "❌ ftpHost is not set in deploy.json for $ENVIRONMENT"
                    exit 1
                  fi

                  if [ -z "$USERNAME" ] || [ -z "$PASSWORD" ]; then
                    echo "❌ Missing credentials. Expected secrets: $USERNAME_VAR / $PASSWORD_VAR"
                    exit 1
                  fi

                  echo "   Host:     $FTP_HOST"
                  echo "   Path:     $REMOTE_PATH"
                  echo "   Protocol: $PROTOCOL"
                  echo ""

                  # Build lftp settings block
                  lftp_config() {
                    local protocol="$1"
                    echo "set ssl:verify-certificate no"
                    echo "set net:timeout 15"
                    echo "set net:max-retries 3"
                    echo "set net:reconnect-interval-base 3"
                    if [ "$protocol" = "sftp" ]; then
                      echo "set sftp:auto-confirm yes"
                    else
                      echo "set ftp:passive-mode on"
                    fi
                  }

                  # Open connection — credentials passed via separate user command to safely
                  # handle special characters (], [, spaces, etc.) in passwords
                  lftp_open() {
                    local protocol="$1" username="$2" password="$3" host="$4"
                    local safe_user safe_pass
                    safe_user="${username//\\/\\\\}"; safe_user="${safe_user//\"/\\\"}"
                    safe_pass="${password//\\/\\\\}"; safe_pass="${safe_pass//\"/\\\"}"
                    if [ "$protocol" = "sftp" ]; then
                      echo "open sftp://$host"
                    else
                      echo "open $host"
                    fi
                    echo "user \"$safe_user\" \"$safe_pass\""
                  }

                  # Detect changed files
                  echo "🔍 Detecting files..."
                  echo "Current commit: $(git rev-parse --short HEAD)"

                  # Marker file on the remote holding the SHA that was last deployed
                  # successfully. Used instead of the push before/after range so that any
                  # number of failed runs in between get swept up automatically on the next
                  # success, rather than silently leaving the server behind.
                  MARKER_FILE=".deploy-sha"

                  update_marker() {
                    local sha short_sha marker_tmp marker_script
                    sha=$(git rev-parse HEAD)
                    short_sha=$(git rev-parse --short HEAD)
                    echo "📝 Updating deploy marker..."
                    marker_tmp=$(mktemp); TMPFILES+=("$marker_tmp")
                    printf '%s' "$sha" > "$marker_tmp"
                    marker_script=$(mktemp); TMPFILES+=("$marker_script")
                    {
                      lftp_config "$PROTOCOL"
                      lftp_open "$PROTOCOL" "$USERNAME" "$PASSWORD" "$FTP_HOST"
                      echo "cd $REMOTE_PATH || exit 1"
                      echo "put $marker_tmp -o $MARKER_FILE"
                    } > "$marker_script"
                    if timeout 20s lftp -f "$marker_script"; then
                      echo "✓ Marker updated to $short_sha"
                    else
                      echo "⚠️  Failed to update deploy marker (next run will fall back to diff heuristics)"
                    fi
                  }

                  # Build theme path list from config array, falling back to all of themes/
                  THEME_PATHS=()
                  while IFS= read -r theme; do
                    THEME_PATHS+=("themes/$theme/")
                  done < <(jq -r ".${ENVIRONMENT}.themes // [] | .[]" "$CONFIG_FILE")

                  if [ ${#THEME_PATHS[@]} -eq 0 ]; then
                    THEME_PATHS=("themes/")
                  fi

                  echo "   Themes:   ${THEME_PATHS[*]}"
                  echo ""

                  # Clean mode: wipe each remote theme directory entirely before uploading,
                  # so anything that accumulated on the server outside of git (stray files
                  # from old manual FTP uploads, orphaned build chunks, etc.) can't linger.
                  # Forces a full re-upload afterwards regardless of uploadAll, since there's
                  # nothing left on the remote to diff against.
                  if [ "$CLEAN" = "true" ]; then
                    echo "🧨 Clean mode enabled — wiping remote theme directories before upload"
                    CLEAN_SCRIPT=$(mktemp); TMPFILES+=("$CLEAN_SCRIPT")
                    {
                      lftp_config "$PROTOCOL"
                      lftp_open "$PROTOCOL" "$USERNAME" "$PASSWORD" "$FTP_HOST"
                      echo "cd $REMOTE_PATH || exit 1"
                      for theme_path in "${THEME_PATHS[@]}"; do
                        theme_name="${theme_path#themes/}"; theme_name="${theme_name%/}"
                        [ -z "$theme_name" ] && continue
                        echo "echo '🗑️  Removing $theme_name/'"
                        echo "rm -r -f $theme_name"
                        echo "mkdir -f -p $theme_name"
                      done
                    } > "$CLEAN_SCRIPT"

                    if timeout 60s lftp -f "$CLEAN_SCRIPT"; then
                      echo "✓ Remote theme directories wiped"
                    else
                      echo "❌ Failed to wipe remote theme directories"
                      exit 1
                    fi
                    echo ""
                    UPLOAD_ALL="true"
                  fi

                  LAST_SHA=""
                  if [ "$UPLOAD_ALL" != "true" ]; then
                    echo "🔎 Checking remote deploy marker..."
                    MARKER_TMP=$(mktemp); TMPFILES+=("$MARKER_TMP")
                    MARKER_SCRIPT=$(mktemp); TMPFILES+=("$MARKER_SCRIPT")
                    {
                      lftp_config "$PROTOCOL"
                      lftp_open "$PROTOCOL" "$USERNAME" "$PASSWORD" "$FTP_HOST"
                      echo "cd $REMOTE_PATH || exit 1"
                      echo "get $MARKER_FILE -o $MARKER_TMP"
                    } > "$MARKER_SCRIPT"

                    if timeout 20s lftp -f "$MARKER_SCRIPT" >/dev/null 2>&1 && [ -s "$MARKER_TMP" ]; then
                      LAST_SHA=$(tr -d '[:space:]' < "$MARKER_TMP")
                      if git cat-file -e "${LAST_SHA}^{commit}" 2>/dev/null; then
                        echo "✓ Last deployed SHA: $(git rev-parse --short "$LAST_SHA")"
                      else
                        echo "⚠️  Marker SHA not found in local history (force-push/rebase?) — falling back"
                        LAST_SHA=""
                      fi
                    else
                      echo "ℹ️  No deploy marker on remote yet (first run?) — falling back"
                    fi
                  fi

                  if [ "$UPLOAD_ALL" = "true" ]; then
                    DIFF_MODE="all"
                  elif [ -n "$LAST_SHA" ]; then
                    DIFF_MODE="marker_sha"
                  elif [ -n "$BEFORE_SHA" ] && git cat-file -e "${BEFORE_SHA}^{commit}" 2>/dev/null; then
                    DIFF_MODE="before_sha"
                  elif git rev-parse HEAD~1 >/dev/null 2>&1; then
                    DIFF_MODE="head_minus_1"
                  else
                    DIFF_MODE="all"
                  fi

                  DELETED=""
                  case "$DIFF_MODE" in
                    all)
                      CHANGED=$(git ls-files "${THEME_PATHS[@]}")
                      echo "📦 Upload all mode — including all tracked theme files"
                      ;;
                    marker_sha)
                      CHANGED=$(git diff --name-only --diff-filter=ACMR "$LAST_SHA" HEAD -- "${THEME_PATHS[@]}")
                      DELETED=$(git diff --name-only --diff-filter=D "$LAST_SHA" HEAD -- "${THEME_PATHS[@]}")
                      echo "Last deployed SHA: $(git rev-parse --short "$LAST_SHA")"
                      echo "📋 Comparing last successful deploy ($LAST_SHA → HEAD)"
                      ;;
                    before_sha)
                      CHANGED=$(git diff --name-only --diff-filter=ACMR "$BEFORE_SHA" HEAD -- "${THEME_PATHS[@]}")
                      DELETED=$(git diff --name-only --diff-filter=D "$BEFORE_SHA" HEAD -- "${THEME_PATHS[@]}")
                      echo "Before SHA: $(git rev-parse --short "$BEFORE_SHA")"
                      echo "📋 Comparing push range ($BEFORE_SHA → HEAD)"
                      ;;
                    head_minus_1)
                      CHANGED=$(git diff --name-only --diff-filter=ACMR HEAD~1 HEAD -- "${THEME_PATHS[@]}")
                      DELETED=$(git diff --name-only --diff-filter=D HEAD~1 HEAD -- "${THEME_PATHS[@]}")
                      echo "Previous commit: $(git rev-parse --short HEAD~1)"
                      echo "📋 Comparing HEAD~1 to HEAD"
                      ;;
                  esac

                  # Always include gitignored build output (built fresh by Vite, never in git)
                  # - matched generically via git's own ignore rules rather than a hardcoded
                  #   filename pattern, so it doesn't go stale again if the build tool or its
                  #   output naming changes. Covers assets/.vite/manifest.json, hashed
                  #   app/block JS+CSS chunks, etc.
                  BUILT_LIST=$(mktemp); TMPFILES+=("$BUILT_LIST")
                  for theme_path in "${THEME_PATHS[@]}"; do
                    git ls-files --others --ignored --exclude-standard -- "${theme_path}assets/" 2>/dev/null >> "$BUILT_LIST" || true
                  done
                  BUILT=$(sort -u "$BUILT_LIST")

                  if [ -n "$BUILT" ]; then
                    echo "🔨 Including built assets"
                    CHANGED=$(printf '%s\n%s' "$CHANGED" "$BUILT" | sort -u | grep -v '^$')
                  fi

                  if [ -z "$CHANGED" ] && [ -z "$DELETED" ]; then
                    echo "✅ No files to upload"
                    update_marker
                    exit 0
                  fi

                  TOTAL_FILES=$(if [ -n "$CHANGED" ]; then echo "$CHANGED" | wc -l | tr -d '[:space:]'; else echo 0; fi)
                  DELETED_COUNT=$(if [ -n "$DELETED" ]; then echo "$DELETED" | wc -l | tr -d '[:space:]'; else echo 0; fi)
                  echo "📁 Files to upload: $TOTAL_FILES"
                  [ "$DELETED_COUNT" -gt 0 ] && echo "🗑️  Files to delete: $DELETED_COUNT"
                  echo ""

                  # Test connection — retried a few times since transient host-side blips
                  # (brief FTP daemon hiccups, momentary connection-limit trips, etc.) shouldn't
                  # fail the whole deploy on their own.
                  echo "🔌 Testing connection to $FTP_HOST..."
                  FTP_TEST=$(mktemp); TMPFILES+=("$FTP_TEST")
                  {
                    lftp_config "$PROTOCOL"
                    lftp_open "$PROTOCOL" "$USERNAME" "$PASSWORD" "$FTP_HOST"
                    echo "cd $REMOTE_PATH || exit 1"
                    echo "quit"
                  } > "$FTP_TEST"

                  CONNECT_ATTEMPTS=3
                  CONNECTED=false
                  for attempt in $(seq 1 "$CONNECT_ATTEMPTS"); do
                    if timeout 30s lftp -f "$FTP_TEST" 2>&1; then
                      CONNECTED=true
                      break
                    fi
                    if [ "$attempt" -lt "$CONNECT_ATTEMPTS" ]; then
                      echo "⚠️  Connection attempt $attempt/$CONNECT_ATTEMPTS failed — retrying in 10s..."
                      sleep 10
                    fi
                  done

                  if [ "$CONNECTED" != "true" ]; then
                    echo "❌ Cannot connect to $FTP_HOST or access $REMOTE_PATH (after $CONNECT_ATTEMPTS attempts)"
                    exit 1
                  fi
                  echo "✓ Connected"
                  echo ""

                  # Use chunked upload if chunkSize is set, otherwise upload all in one connection
                  if [ "$CHUNK_SIZE" -gt 0 ] 2>/dev/null; then
                    TOTAL_CHUNKS=$(( (TOTAL_FILES + CHUNK_SIZE - 1) / CHUNK_SIZE ))
                    echo "📦 Chunked upload: $TOTAL_CHUNKS chunks of up to $CHUNK_SIZE files (reconnects between each)"
                  else
                    TOTAL_CHUNKS=1
                    CHUNK_SIZE=$TOTAL_FILES
                  fi
                  echo ""

                  CHUNK_NUM=0
                  OFFSET=0
                  OVERALL=0

                  while [ "$OFFSET" -lt "$TOTAL_FILES" ]; do
                    CHUNK_NUM=$((CHUNK_NUM + 1))
                    CHUNK_FILES=$(echo "$CHANGED" | tail -n +$((OFFSET + 1)) | head -n "$CHUNK_SIZE")
                    CHUNK_COUNT=$(echo "$CHUNK_FILES" | wc -l | tr -d '[:space:]')

                    [ "$TOTAL_CHUNKS" -gt 1 ] && echo "🔄 Chunk $CHUNK_NUM/$TOTAL_CHUNKS ($CHUNK_COUNT files)..."

                    FTP_SCRIPT=$(mktemp); TMPFILES+=("$FTP_SCRIPT")
                    {
                      lftp_config "$PROTOCOL"
                      lftp_open "$PROTOCOL" "$USERNAME" "$PASSWORD" "$FTP_HOST"
                      echo "cd $REMOTE_PATH || exit 1"

                      while IFS= read -r file; do
                        OVERALL=$((OVERALL + 1))
                        REMOTE_FILE="${file#themes/}"
                        DIR=$(dirname "$REMOTE_FILE")
                        LOCAL_FILE="$WORKSPACE_DIR/$file"
                        [ "$DIR" != "." ] && [ "$DIR" != "" ] && echo "mkdir -f -p $DIR"
                        echo "echo '[$OVERALL/$TOTAL_FILES] $file'"
                        echo "put $LOCAL_FILE -o $REMOTE_FILE"
                      done <<< "$CHUNK_FILES"
                    } > "$FTP_SCRIPT"

                    CHUNK_TIMEOUT=$(( CHUNK_COUNT * 5 + 30 ))
                    if timeout "${CHUNK_TIMEOUT}s" lftp -f "$FTP_SCRIPT"; then
                      [ "$TOTAL_CHUNKS" -gt 1 ] && echo "✓ Chunk $CHUNK_NUM complete"
                    else
                      LFTP_EXIT=$?
                      echo ""
                      echo "❌ Upload failed at chunk $CHUNK_NUM (exit code: $LFTP_EXIT)"
                      exit 1
                    fi

                    OFFSET=$((OFFSET + CHUNK_SIZE))
                  done

                  # Remove files on the remote that were deleted from git, so stale files
                  # (old class definitions, renamed includes, etc.) don't linger on the
                  # server and get picked up by glob-based includes.
                  if [ "$DELETED_COUNT" -gt 0 ]; then
                    echo ""
                    echo "🗑️  Deleting $DELETED_COUNT removed file(s)..."

                    DELETE_SCRIPT=$(mktemp); TMPFILES+=("$DELETE_SCRIPT")
                    {
                      lftp_config "$PROTOCOL"
                      lftp_open "$PROTOCOL" "$USERNAME" "$PASSWORD" "$FTP_HOST"
                      echo "cd $REMOTE_PATH || exit 1"

                      DEL_NUM=0
                      while IFS= read -r file; do
                        DEL_NUM=$((DEL_NUM + 1))
                        REMOTE_FILE="${file#themes/}"
                        echo "echo '[$DEL_NUM/$DELETED_COUNT] rm $file'"
                        echo "rm -f $REMOTE_FILE"
                      done <<< "$DELETED"
                    } > "$DELETE_SCRIPT"

                    DELETE_TIMEOUT=$(( DELETED_COUNT * 5 + 30 ))
                    if timeout "${DELETE_TIMEOUT}s" lftp -f "$DELETE_SCRIPT"; then
                      echo "✓ Deletions complete"
                    else
                      echo ""
                      echo "⚠️  Some deletions may have failed (exit code: $?) - check manually if needed"
                    fi
                  fi

                  echo ""
                  update_marker

                  echo ""
                  echo "✅ Deploy complete! ($TOTAL_FILES uploaded, $DELETED_COUNT deleted)"
```

## `.github/workflows/remote-cleanup.yml`

```yaml
name: Remote Cleanup (delete a file on FTP) (reusable)

on:
    workflow_call:
        inputs:
            environment:
                description: 'staging | production'
                required: true
                type: string
            target_path:
                description: 'Path to delete, relative to the theme root (e.g. "hot")'
                required: true
                type: string
            config-file:
                description: 'Path to deploy.json in the caller repo'
                required: false
                type: string
                default: '.github/config/deploy.json'

jobs:
    delete-remote-file:
        runs-on: ubuntu-latest
        timeout-minutes: 5
        steps:
            - name: Checkout caller repository
              uses: actions/checkout@v5

            - name: Delete remote file
              env:
                  ENVIRONMENT: ${{ inputs.environment }}
                  TARGET_PATH: ${{ inputs.target_path }}
                  CONFIG_FILE: ${{ inputs.config-file }}
                  STAGING_FTP_USERNAME: ${{ secrets.STAGING_FTP_USERNAME }}
                  STAGING_FTP_PASSWORD: ${{ secrets.STAGING_FTP_PASSWORD }}
                  PRODUCTION_FTP_USERNAME: ${{ secrets.PRODUCTION_FTP_USERNAME }}
                  PRODUCTION_FTP_PASSWORD: ${{ secrets.PRODUCTION_FTP_PASSWORD }}
              run: |
                  sudo apt-get install -y -qq lftp jq

                  set -e

                  ENVIRONMENT="${ENVIRONMENT:?ENVIRONMENT is required}"
                  TARGET_PATH="${TARGET_PATH:?TARGET_PATH is required}"

                  # Guardrails: never allow deleting the whole remote root or traversing out
                  # of it.
                  if [ -z "$TARGET_PATH" ] || [ "$TARGET_PATH" = "/" ] || [ "$TARGET_PATH" = "." ]; then
                    echo "❌ Refusing to delete an empty/root path"
                    exit 1
                  fi
                  case "$TARGET_PATH" in
                    ..|../*|*/../*|*/..)
                      echo "❌ TARGET_PATH must not contain '..'"
                      exit 1
                      ;;
                  esac

                  if [ ! -f "$CONFIG_FILE" ]; then
                    echo "❌ Config file not found: $CONFIG_FILE"
                    exit 1
                  fi

                  REMOTE_PATH=$(jq -r ".${ENVIRONMENT}.remotePath" "$CONFIG_FILE")
                  FTP_HOST=$(jq -r ".${ENVIRONMENT}.ftpHost" "$CONFIG_FILE")
                  PROTOCOL=$(jq -r ".${ENVIRONMENT}.protocol // \"ftp\"" "$CONFIG_FILE")
                  THEME=$(jq -r ".${ENVIRONMENT}.themes[0] // \"\"" "$CONFIG_FILE")

                  ENV_UPPER=$(echo "$ENVIRONMENT" | tr '[:lower:]' '[:upper:]')
                  USERNAME_VAR="${ENV_UPPER}_FTP_USERNAME"
                  PASSWORD_VAR="${ENV_UPPER}_FTP_PASSWORD"
                  USERNAME="${!USERNAME_VAR}"
                  PASSWORD="${!PASSWORD_VAR}"

                  if [ -z "$FTP_HOST" ] || [ "$FTP_HOST" = "null" ]; then
                    echo "❌ ftpHost is not set in deploy.json for $ENVIRONMENT"
                    exit 1
                  fi
                  if [ -z "$USERNAME" ] || [ -z "$PASSWORD" ]; then
                    echo "❌ Missing credentials. Expected secrets: $USERNAME_VAR / $PASSWORD_VAR"
                    exit 1
                  fi

                  FULL_REMOTE="$REMOTE_PATH/$THEME/$TARGET_PATH"

                  echo "🌐 Environment: $ENVIRONMENT"
                  echo "   Host:   $FTP_HOST"
                  echo "   Target: $FULL_REMOTE"
                  echo ""

                  FTP_SCRIPT=$(mktemp)
                  trap 'rm -f "$FTP_SCRIPT"' EXIT

                  {
                    echo "set ssl:verify-certificate no"
                    echo "set net:timeout 15"
                    echo "set net:max-retries 3"
                    echo "set net:reconnect-interval-base 3"
                    if [ "$PROTOCOL" = "sftp" ]; then
                      echo "set sftp:auto-confirm yes"
                      echo "open sftp://$FTP_HOST"
                    else
                      echo "set ftp:passive-mode on"
                      echo "open $FTP_HOST"
                    fi
                    safe_user="${USERNAME//\\/\\\\}"; safe_user="${safe_user//\"/\\\"}"
                    safe_pass="${PASSWORD//\\/\\\\}"; safe_pass="${safe_pass//\"/\\\"}"
                    echo "user \"$safe_user\" \"$safe_pass\""
                    echo "cd $REMOTE_PATH/$THEME || exit 1"
                    echo "rm -f $TARGET_PATH"
                  } > "$FTP_SCRIPT"

                  if timeout 30s lftp -f "$FTP_SCRIPT"; then
                    echo "✓ Delete command sent for $TARGET_PATH (no error means it's gone or was already absent)"
                  else
                    echo "❌ Delete failed"
                    exit 1
                  fi
```

## Caller workflows (go in each *project* repo, not this one)

Replace `<owner>` with the GitHub account/org this repo ends up under, and
`@v1` with whatever tag you push (see README → Versioning).

`.github/workflows/deploy-main.yml`:

```yaml
name: Deploy to Live
on:
    push:
        branches: [main]
    workflow_dispatch:

jobs:
    deploy-live:
        uses: <owner>/gha-wp-deploy/.github/workflows/deploy.yml@v1
        with:
            environment: production
        secrets: inherit
```

`.github/workflows/deploy-staging.yml`:

```yaml
name: Deploy to Staging
on:
    push:
        branches: [feature/reskin] # adjust per project
    workflow_dispatch:

jobs:
    deploy-staging:
        uses: <owner>/gha-wp-deploy/.github/workflows/deploy.yml@v1
        with:
            environment: staging
        secrets: inherit
```

`.github/workflows/remote-cleanup.yml`:

```yaml
name: Remote Cleanup (delete a file on FTP)
on:
    workflow_dispatch:
        inputs:
            environment:
                description: 'Target environment'
                required: true
                type: choice
                options:
                    - staging
                    - production
            target_path:
                description: 'Path to delete, relative to the theme root (e.g. "hot")'
                required: true
                default: 'hot'

jobs:
    delete-remote-file:
        uses: <owner>/gha-wp-deploy/.github/workflows/remote-cleanup.yml@v1
        with:
            environment: ${{ inputs.environment }}
            target_path: ${{ inputs.target_path }}
        secrets: inherit
```

`secrets: inherit` passes the caller repo's own `STAGING_FTP_USERNAME` /
`STAGING_FTP_PASSWORD` / `PRODUCTION_FTP_USERNAME` / `PRODUCTION_FTP_PASSWORD`
straight through to the reusable workflow — no renaming needed, and this repo
never has to know the actual credentials.

## Creating and tagging the repo

```sh
git init
git add -A
git commit -m "Initial reusable deploy workflows"
git branch -M main
git remote add origin https://github.com/<owner>/gha-wp-deploy.git
git push -u origin main
git tag v1.0.0
git tag v1
git push origin v1.0.0 v1
```
