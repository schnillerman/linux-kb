# Sort Photos into Subfolders by Category
```bash
#!/bin/bash

# Interactive directory selection
read -p "Enter the path to the source directory: " source_dir
source_dir=$(realpath "$source_dir")

# Interactive keyword input
echo -e "\nEnter additional keywords (comma-separated, spaces allowed):"
echo "Example: Lightroom Edit,HDR"
read -p "Keywords: " keywords_input

# Process keywords
IFS=',' read -ra temp_keywords <<< "$keywords_input"
declare -a keywords=()
for keyword in "${temp_keywords[@]}"; do
    keyword=$(echo "$keyword" | xargs)  # Trim whitespace
    [ -n "$keyword" ] && keywords+=("$keyword")
done

# Verify directory existence
if [ ! -d "$source_dir" ]; then
    echo "Error: Directory '$source_dir' does not exist!"
    exit 1
else
    echo -e "\nProcessing directory: $source_dir"
    echo "Standard categories: WhatsApp, UUID, Cam, Snapseed"
    echo "Active keywords: ${keywords[*]}"
fi

# Configuration
parent_dir=$(dirname "$source_dir")
original_dir=$(basename "$source_dir")
whatsapp_pattern="^.*(IMG|VID)-[0-9]{8}-WA[0-9]{4}.*\.(jpg|jpeg|png|mp4|mov)$"
uuid_pattern="^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}\..+$"
camera_pattern=".*(_?)(dsc|img|pxl|dji|gopr|movi|dscf|ndvr|pano|rsc)(_?)[0-9]+.*"
snapseed_pattern="snapseed"

# Main processing loop
find "$source_dir" -type d \( -iname "*@eadir*" \) -prune -o -type f -print0 | while IFS= read -r -d '' file; do
    relative_path="${file#$source_dir/}"
    filename=$(basename "$relative_path")
    filename_lower="${filename,,}"  # Convert to lowercase
    extension=$(echo "${filename##*.}" | tr '[:lower:]' '[:upper:]')
    
    # Normalize JPEG extensions
    [[ "$extension" == "JPEG" ]] && extension="JPG"
    
    target_category=""
    camera_prefix=""
    target_extension="$extension"

    # 1) WhatsApp detection
    if [[ "$filename_lower" =~ $whatsapp_pattern ]]; then
        target_category="WhatsApp"
    
    # 2) UUID detection
    elif [[ "$filename_lower" =~ $uuid_pattern ]]; then
        target_category="UUID"
    
    # 3) Camera detection with prefix extraction
    elif [[ "$filename_lower" =~ $camera_pattern ]]; then
        target_category="Cam"
        # Extract camera prefix and convert to uppercase
        camera_prefix=$(echo "${BASH_REMATCH[2]}" | tr '[:lower:]' '[:upper:]')
    
    # 4) Snapseed detection
    elif [[ "$filename_lower" == *"$snapseed_pattern"* ]]; then
        target_category="Snapseed"
    
    # 5) Keyword matching
    else
        for keyword in "${keywords[@]}"; do
            if [[ "${filename_lower}" == *"${keyword,,}"* ]]; then
                target_category="$keyword"
                break
            fi
        done
    fi

    # Target path logic
    if [ -n "$target_category" ]; then
        if [ "$target_category" == "Cam" ] && [ -n "$camera_prefix" ]; then
            target_dir="${parent_dir}/${original_dir} - ${target_category} - ${camera_prefix} - ${target_extension}"
        else
            target_dir="${parent_dir}/${original_dir} - ${target_category} - ${target_extension}"
        fi
    else
        target_dir="${parent_dir}/${original_dir} - ${target_extension}"
    fi

    target_path="${target_dir}/${relative_path}"
    mkdir -p "$(dirname "$target_path")"
    mv -nv "$file" "$target_path"
done
```

## Key Components Explained:

1. **Interactive Inputs**  
   - Collects source directory path and optional keywords
   - Processes keywords into an array

2. **File Categorization**  
   - **WhatsApp**: Matches `IMG-YYYYMMDD-WAXXXX` pattern
   - **UUID**: Matches UUIDv4 format (`8-4-4-4-12` hex structure)
   - **Camera**: Detects common camera prefixes (DSC, IMG, DJI, etc.)
   - **Snapseed**: Looks for "snapseed" in filename
   - **Custom Keywords**: User-defined patterns

3. **File Organization**  
   Creates nested directory structure:  
   `ParentDir/OriginalDirName - Category - (CameraPrefix) - Extension`

4. **Special Handling**  
   - Converts `.jpeg` → `.jpg` for consistency
   - Skips `@eaDir` directories (Synology NAS thumbnails)
   - Preserves original folder structure

5. **Safety Features**  
   - `-n` flag in `mv` prevents overwrites
   - `-print0`/`read -d ''` handles spaces in filenames
   - Case-insensitive matching for broader compatibility

## Regex Patterns Explained:

| Pattern          | Purpose                                      | Example Matches              |
|------------------|----------------------------------------------|-------------------------------|
| `whatsapp_pattern` | WhatsApp media files                       | IMG-20231015-WA0001.jpg       |
| `uuid_pattern`     | UUIDv4 formatted files                     | 550e8400-e29b-41d4-a716-446655440000.jpg |
| `camera_pattern`   | Camera-generated files                     | DSC_1234, GOPR0001, DJI_1234 |
| `snapseed_pattern` | Snapseed-edited files                      | sunset_snapseed_edit.jpg      |

This script provides a sophisticated way to organize mixed media files while preserving metadata and directory structures.

# Identify photos with non-authentic EXIF data (manipulated by, e.g., WhatsApp)
## **What WhatsApp Does to EXIF Data**  
1. **`DateTimeOriginal` is Overwritten**  
   WhatsApp sets this tag to the **time the file was sent or saved** on the device, *not* the original capture date.  

2. **Other EXIF Tags Are Removed**  
   - `GPS` coordinates (latitude/longitude)  
   - `Make` (camera manufacturer)  
   - `Model` (camera model)  
   - `Software` (editing tools)  

---

## **Why `exiftool -if '$datetimeoriginal'` Fails**  
- **Manipulated files** still have the `DateTimeOriginal` tag but with **incorrect values**.  
- The command only checks for the *presence* of the tag, not its *authenticity*.  

---

## **Improved Method: Identifying Real vs. WhatsApp Photos**  
### **1. Check for Missing EXIF Tags**  
WhatsApp files typically lack camera metadata. This command lists files without `Make` or `Model` tags (suspicious for WhatsApp):  
```bash  
exiftool -q -if 'not defined $make or not defined $model' -r /path/to/folder  
```  

### **2. Search for WhatsApp-Specific Filenames**  
WhatsApp renames media files to the pattern `IMG-YYYYMMDD-WAXXXX.*`:  
```bash  
find /path/to/folder -type f -iregex ".*IMG-[0-9]{8}-WA[0-9]{4}.*"  
```  

### **3. Compare `DateTimeOriginal` with Filesystem Timestamp**  
Real photos often have similar values for:  
- `DateTimeOriginal` (EXIF)  
- `FileModifyDate` (filesystem)  

**Command to Check Discrepancies** (≥1 hour difference):  
```bash  
exiftool -q -r -if '  
    $datetimeoriginal &&  
    (($filemodifydate - $datetimeoriginal) > 3600 ||  
    ($datetimeoriginal - $filemodifydate) > 3600  
' /path/to/folder  
```  

### **4. Check for Missing GPS Data**  
Original photos often include GPS coordinates:  
```bash  
exiftool -q -if 'not defined $gpslatitude' -r /path/to/folder  
```  

---

## **Example: Automated Classification Script**  
```bash  
#!/bin/bash  

# Real Photos  
exiftool -q -r -ext jpg -ext heic \  
    -if '$make && $model && $gpslatitude && $datetimeoriginal' \  
    -p "Real Photo: $directory/$filename" /path/to/folder  

# WhatsApp Photos  
exiftool -q -r -ext jpg -ext heic \  
    -if 'not defined $make && not defined $model && not defined $gpslatitude' \  
    -p "Suspicious (WhatsApp?): $directory/$filename" /path/to/folder  
```  

---

## **Summary**  
| **Feature**               | **Real Photos**                | **WhatsApp Photos**           |  
|---------------------------|---------------------------------|--------------------------------|  
| `DateTimeOriginal`        | Actual capture date            | Timestamp of sending/saving   |  
| `Make`/`Model`            | Present (e.g., "Canon EOS R")  | Missing                        |  
| `GPSLatitude`/`Longitude` | Usually present                | Always missing                |  
| Filename                  | Camera-specific (e.g., DSC_1234)| `IMG-YYYYMMDD-WAXXXX`         |  

Using these methods, you can reliably identify WhatsApp-altered files! 📸🔍
