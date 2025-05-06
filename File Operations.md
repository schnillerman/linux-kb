# Move ALL content of a directory (incl. hidden files) to current directory:
```bash
rsync -a --progress --remove-source-files --include='.*' /QUELLPFAD/ . && find /QUELLPFAD -type d -empty -delete
```
## How it works:
1. **`rsync`-command**:
   - `-a`: archive mode (preserves rights, timestamps, etc.)
   - `--progress`: shows progress
   - `--remove-source-files`: deletes source files after succesful transfer
   - `--include='.*'`: includes hidden files
   - `/QUELLPFAD/ .`: moves **all** files from source path (incl. root) into current directory

2. **`find`-command**:
   - deletes empty directories in source dir after moving

---

## Example:
- **Source path**: `/mnt/external/backup`
- **Hidden file**: `/mnt/external/backup/.important_config`
- **Subfolder**: `/mnt/external/backup/photos/.hidden_album`

Command moves:
- `.important_config` → current directory
- `.hidden_album` → `./photos/.hidden_album`
- All other files/directories

---

> [!WARNING]
> - **Create backup first**!
> - `rsync` does **not** overwrite existing files (except with `--ignore-existing`).
> - test with `--dry-run` first:
>   ```bash
>   rsync -a --dry-run --include='.*' /QUELLPFAD/ .
>   ```

# Delete all folders containing keyword from current direcctory
  ```bash
find . -type d -iname "*@eadir*" -print | while read -r ordner; do
    echo "Lösche: $ordner"
    rm -rf "$ordner"
done
```
