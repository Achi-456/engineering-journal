# Daily DevOps Journal — December 27, 2025

## Topic
Basic project setup commands: `mkdir`, `touch`, `date`, and hidden files.

## Key Points
- `mkdir -p` (or `--parents`) avoids errors when a directory already exists and is useful for creating nested structures.
- Use the `$HOME` environment variable to refer to the user’s home directory instead of hardcoding absolute paths.
- Files starting with a dot (e.g., `.filename.ext`) are hidden on Unix-like systems; use `ls -a` to list them.
- The `date` command supports formatted timestamps, e.g. `+%F` (YYYY-MM-DD) and `%T` (HH:MM:SS). Use `>>` to append to a file.

## Examples
```bash
# Create nested directories safely
mkdir -p project/src/components

# Use HOME variable (Unix/macOS; on Windows PowerShell use $HOME as well)
mkdir -p "$HOME/projects/demo"

# Create a hidden file
touch .env.local
ls -a

# Append a timestamp to a log file
# Date format: YYYY-MM-DD_HH:MM:SS
printf "$(date +%F_%T)\n" >> activity.log
```

## Notes
- On Windows PowerShell, `$HOME` is available and points to the user profile directory (e.g., `C:\Users\\<User>`).
- Hidden files (dotfiles) behavior applies to Unix-like systems. Windows File Explorer may require enabling “Hidden items” to view hidden files.
- Prefer appending logs with `>>` to preserve history.