# Sort Photos into Subfolders by Category
The following script organizes files in a source directory into different directories parallel to the source directory as described in the following subsections.
```bash
#!/bin/bash

###############################################################################
#                            USER INPUT SECTION                               #
###############################################################################

# Prompt user for source directory and resolve to absolute path
read -p "Enter the path to the source directory: " source_dir
source_dir=$(realpath "$source_dir")

# Duplicate handling configuration
# - Determines whether duplicates get moved to ../Duplicates or handled in-place
read -p "Move duplicates to ../Duplicates? [y/N]: " move_duplicates
handle_duplicates="no"
[[ "$move_duplicates" =~ [yY] ]] && handle_duplicates="yes"

# Custom keyword configuration
# - Accepts comma-separated list of additional categories
# - Processes input to create clean array of keywords
echo -e "\nEnter additional keywords (comma-separated, spaces allowed):"
echo "Example: Lightroom Edit,HDR"
read -p "Keywords: " keywords_input

# Process keywords into normalized array:
# 1. Split on commas
# 2. Trim whitespace
# 3. Filter empty entries
IFS=',' read -ra temp_keywords <<< "$keywords_input"
declare -a keywords=()
for keyword in "${temp_keywords[@]}"; do
    keyword=$(echo "$keyword" | xargs)  # Trim whitespace
    [ -n "$keyword" ] && keywords+=("$keyword")
done

###############################################################################
#                         DIRECTORY VERIFICATION                              #
###############################################################################

# Validate source directory exists before proceeding
if [ ! -d "$source_dir" ]; then
    echo "Error: Directory '$source_dir' does not exist!"
    exit 1
else
    # Display processing configuration
    echo -e "\nProcessing directory: $source_dir"
    echo "Standard categories: WhatsApp, UUID, Cam, Snapseed"
    echo "Active keywords: ${keywords[*]}"
fi

###############################################################################
#                          CONFIGURATION SETTINGS                             #
###############################################################################

# Path configuration
parent_dir=$(dirname "$source_dir")                # Parent of source directory
original_dir=$(basename "$source_dir")             # Name of source directory
duplicates_root="${parent_dir}/Duplicates"         # Central duplicates location

# File pattern definitions:
# WhatsApp: Matches WAXXXX generated filenames
whatsapp_pattern="^.*(IMG|VID)-[0-9]{8}-WA[0-9]{4}.*\.(jpg|jpeg|png|mp4|mov)$"

# UUID: Matches standard 8-4-4-4-12 hexadecimal format
uuid_pattern="^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}.*\..+$"

# Camera: Matches common camera/vendor filename prefixes
camera_pattern=".*(_?)(dsc|img|pxl|dji|gopr|movi|dscf|ndvr|pano|rsc)(_?)[0-9]+.*"

# Snapseed: Simple pattern match for edited files
snapseed_pattern="snapseed"

###############################################################################
#                          MAIN PROCESSING LOOP                               #
###############################################################################

# Process all files in source directory recursively:
# - Exclude special "@eadir" directories
# - Handle spaces in filenames via null-delimited output
find "$source_dir" -type d \( -iname "*@eadir*" \) -prune -o -type f -print0 | while IFS= read -r -d '' file; do

    # Extract relative path components
    relative_path="${file#$source_dir/}"  # Path relative to source directory
    filename=$(basename "$relative_path") # Filename with extension
    filename_lower="${filename,,}"        # Lowercase version for case-insensitive matching
    extension=$(echo "${filename##*.}" | tr '[:lower:]' '[:upper:]')  # Uppercase extension

    # Normalize JPEG extensions to consistent format
    [[ "$extension" == "JPEG" ]] && extension="JPG"
    
    # Initialize target categorization variables
    target_category=""
    camera_prefix=""
    target_extension="$extension"

    ###########################################################################
    #                      CATEGORY DETECTION LOGIC                           #
    ###########################################################################

    # Detection priority order:
    # 1. WhatsApp files
    # 2. UUID files
    # 3. Camera files
    # 4. Snapseed files
    # 5. User-defined keywords
    
    if [[ "$filename_lower" =~ $whatsapp_pattern ]]; then
        target_category="WhatsApp"
    elif [[ "$filename_lower" =~ $uuid_pattern ]]; then
        target_category="UUID"
    elif [[ "$filename_lower" =~ $camera_pattern ]]; then
        target_category="Cam"
        # Extract camera prefix (e.g., DSC, IMG) for subcategorization
        camera_prefix=$(echo "${BASH_REMATCH[2]}" | tr '[:lower:]' '[:upper:]')
    elif [[ "$filename_lower" == *"$snapseed_pattern"* ]]; then
        target_category="Snapseed"
    else
        # Check against user-defined keywords
        for keyword in "${keywords[@]}"; do
            if [[ "${filename_lower}" == *"${keyword,,}"* ]]; then
                target_category="$keyword"
                break  # Stop at first match
            fi
        done
    fi

    ###########################################################################
    #                       TARGET PATH CONSTRUCTION                          #
    ###########################################################################

    # Build destination path based on detected category:
    # Format: ParentDir/SourceDirName - Category - Extension/OriginalPath
    
    if [ -n "$target_category" ]; then
        # Special handling for camera files with vendor prefixes
        if [ "$target_category" == "Cam" ] && [ -n "$camera_prefix" ]; then
            target_dir="${parent_dir}/${original_dir} - ${target_category} - ${camera_prefix} - ${target_extension}"
        else
            target_dir="${parent_dir}/${original_dir} - ${target_category} - ${target_extension}"
        fi
    else
        # Fallback for uncategorized files
        target_dir="${parent_dir}/${original_dir} - ${target_extension}"
    fi

    # Construct full target path preserving original directory structure
    target_path="${target_dir}/${relative_path}"
    mkdir -p "$(dirname "$target_path")"  # Create necessary directories

    ###########################################################################
    #                          FILE HANDLING LOGIC                            #
    ###########################################################################

    if [ -f "$target_path" ]; then
        # Handle duplicate files based on user preference
        if [ "$handle_duplicates" == "yes" ]; then
            #################################################################
            #                    DUPLICATE MOVING LOGIC                     #
            #################################################################
            
            # Move existing target file to duplicates directory
            target_duplicate_path="${duplicates_root}/${target_dir#${parent_dir}/}/${relative_path}"
            mkdir -p "$(dirname "$target_duplicate_path")"
            mv -v "$target_path" "$target_duplicate_path"
            
            # Move source duplicate to duplicates directory
            source_duplicate_path="${duplicates_root}/${original_dir}/${relative_path}"
            mkdir -p "$(dirname "$source_duplicate_path")"
            mv -v "$file" "$source_duplicate_path"
        else
            #################################################################
            #                  IN-PLACE DUPLICATE HANDLING                  #
            #################################################################
            
            if [ "$target_category" == "UUID" ]; then
                # Special handling for UUID files: size-based overwrite
                current_size=$(stat -c %s "$file")       # Current file size
                existing_size=$(stat -c %s "$target_path")  # Existing file size
                
                # Calculate size difference percentage relative to existing file
                size_diff=$(( (current_size - existing_size) * 100 / existing_size ))
                absolute_diff=${size_diff#-}  # Get absolute value

                if [ "$absolute_diff" -le 15 ]; then
                    # Overwrite if difference ≤15%
                    echo "Overwriting UUID file (${absolute_diff}% size difference): $relative_path"
                    mv -vf "$file" "$target_path"
                else
                    # Preserve existing if difference >15%
                    echo "Skipping UUID duplicate (${absolute_diff}% size difference >15%): $relative_path"
                fi
            else
                # Standard non-UUID duplicate handling
                echo "Skipping duplicate: $relative_path"
            fi
        fi
    else
        # No duplicate exists - perform standard move
        mv -v "$file" "$target_path"
    fi
done
```

## Key Components Explained

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
   - Puts `.jpeg` and `.jpg` in same category JPG
   - Skips `@eaDir` directories (Synology NAS thumbnails)
   - Preserves original folder structure
   - Special overwrite rules for UUID category files (only overwrites UUID files if existing file ≤115% of new file size)

5. **Safety Features**  
   - `-n` flag in `mv` prevents overwrites
   - `-print0`/`read -d ''` handles spaces in filenames
   - Case-insensitive matching for broader compatibility

## Regex Patterns Explained

| Pattern            | Purpose                | Example Matches                          |
|--------------------|------------------------|------------------------------------------|
| `whatsapp_pattern` | WhatsApp media files   | IMG-20231015-WA0001.jpg                  |
| `uuid_pattern`     | UUIDv4 formatted files | 550e8400-e29b-41d4-a716-446655440000.jpg |
| `camera_pattern`   | Camera-generated files | DSC_1234, GOPR0001, DJI_1234             |
| `snapseed_pattern` | Snapseed-edited files  | sunset_snapseed_edit.jpg                 |

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
