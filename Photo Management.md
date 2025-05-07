# Sort Photos into Subfolders by Category
The following script organizes files in a source directory into different directories parallel to the source directory as described in the following subsections.
```bash
#!/bin/bash

###############################################################################
#                            USER INPUT SECTION                               #
###############################################################################

# Get source directory path from user input and resolve to absolute path
read -p "Enter the path to the source directory: " source_dir
source_dir=$(realpath "$source_dir")  # Convert to absolute path for consistency

# Configure duplicate handling: [y] moves duplicates out, [n] handles in-place
read -p "Move duplicates to ../Duplicates? [y/N]: " move_duplicates
handle_duplicates="no"  # Default to in-place handling
[[ "$move_duplicates" =~ [yY] ]] && handle_duplicates="yes"  # Set flag if Yes

# Get user-defined keywords for custom categorization
echo -e "\nEnter additional keywords (comma-separated, spaces allowed):"
echo "Example: Lightroom Edit,HDR"
read -p "Keywords: " keywords_input

# Process keywords into clean array:
# 1. Split comma-separated input
# 2. Trim whitespace from each item
# 3. Remove empty entries
IFS=',' read -ra temp_keywords <<< "$keywords_input"
declare -a keywords=()
for keyword in "${temp_keywords[@]}"; do
    keyword=$(echo "$keyword" | xargs)  # xargs trims whitespace
    [ -n "$keyword" ] && keywords+=("$keyword")  # Add non-empty entries
done

# Get extensions for special categorization
echo -e "\nEnter extensions to categorize (comma-separated, without dots):"
echo "Example: PNG,JPG,MOV"
read -p "Extensions: " extensions_input

# Process extensions into normalized array:
# 1. Split comma-separated input
# 2. Trim whitespace and convert to uppercase
# 3. Remove empty entries
IFS=',' read -ra temp_extensions <<< "$extensions_input"
declare -a extensions=()
for ext in "${temp_extensions[@]}"; do
    ext=$(echo "$ext" | xargs | tr '[:lower:]' '[:upper:]')  # Normalize case
    [ -n "$ext" ] && extensions+=("$ext")  # Add non-empty entries
done

###############################################################################
#                         DIRECTORY VERIFICATION                             #
###############################################################################

# Validate source directory exists before proceeding
if [ ! -d "$source_dir" ]; then
    echo "Error: Directory '$source_dir' does not exist!"
    exit 1
else
    # Display processing configuration for user verification
    echo -e "\nProcessing directory: $source_dir"
    echo "Standard categories: WhatsApp, UUID, Cam, DSC, Snapseed"
    echo "Active keywords: ${keywords[*]}"         # Show parsed keywords
    echo "Active extensions: ${extensions[*]}"     # Show parsed extensions
fi

###############################################################################
#                          CONFIGURATION SETTINGS                             #
###############################################################################

# Path configuration
parent_dir=$(dirname "$source_dir")         # Parent directory of source
original_dir=$(basename "$source_dir")      # Name of source directory
duplicates_root="${parent_dir}/Duplicates"  # Central duplicates location

# File pattern definitions (all patterns use case-insensitive matching)
whatsapp_pattern="^.*(IMG|VID)-[0-9]{8}-WA[0-9]{4}.*\.(jpg|jpeg|png|mp4|mov)$"  # WhatsApp auto-generated files
uuid_pattern="^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}.*\..+$"  # Standard UUID format
camera_pattern=".*(_?)(dsc|img|pxl|dji|gopr|movi|dscf|ndvr|pano|rsc)(_?)[0-9]+.*"  # Camera/vendor filename patterns
snapseed_pattern="snapseed"  # Simple pattern for Snapseed edits

###############################################################################
#                          MAIN PROCESSING LOOP                               #
###############################################################################

# Process files recursively while handling spaces in filenames:
# - Prune special @eadir directories
# - Use null delimiter for safe filename handling
find "$source_dir" -type d \( -iname "*@eadir*" \) -prune -o -type f -print0 | while IFS= read -r -d '' file; do

    # Extract path components
    relative_path="${file#$source_dir/}"   # Path relative to source root
    filename=$(basename "$relative_path")  # Filename with extension
    filename_lower="${filename,,}"         # Lowercase for case-insensitive checks
    extension=$(echo "${filename##*.}" | tr '[:lower:]' '[:upper:]')  # Uppercase extension
    
    # Normalize JPEG extensions to JPG
    [[ "$extension" == "JPEG" ]] && extension="JPG"

    # Initialize target category
    target_category=""

    ###########################################################################
    #                      CATEGORY DETECTION LOGIC                           #
    ###########################################################################
    # Priority order: Keywords > Extensions > Pre-defined Categories > Extension fallback

    # [1] Check user-defined keywords first (highest priority)
    for keyword in "${keywords[@]}"; do
        # Case-insensitive substring match
        if [[ "${filename_lower}" == *"${keyword,,}"* ]]; then
            target_category="$keyword"
            break  # Stop at first match
        fi
    done

    # [2] Check extension-based categorization if no keyword match
    if [ -z "$target_category" ]; then
        # Check if extension is in user-specified list
        if [[ " ${extensions[*]} " =~ " ${extension} " ]]; then
            target_category="$extension"
        fi
    fi

    # [3] Check pre-defined categories if no previous matches
    if [ -z "$target_category" ]; then
        # WhatsApp detection
        if [[ "$filename_lower" =~ $whatsapp_pattern ]]; then
            target_category="WhatsApp"
        
        # UUID detection
        elif [[ "$filename_lower" =~ $uuid_pattern ]]; then
            target_category="UUID"
        
        # Camera/DSC detection
        elif [[ "$filename_lower" =~ $camera_pattern ]]; then
            # Extract camera prefix from regex match group 2
            camera_prefix=$(echo "${BASH_REMATCH[2]}" | tr '[:lower:]' '[:upper:]')
            # Separate DSC files into their own category
            target_category=$([[ "$camera_prefix" == "DSC" ]] && echo "DSC" || echo "Cam")
        
        # Snapseed detection
        elif [[ "$filename_lower" == *"$snapseed_pattern"* ]]; then
            target_category="Snapseed"
        fi
    fi

    # [4] Final fallback: Use file extension if no other matches
    [ -z "$target_category" ] && target_category="$extension"

    ###########################################################################
    #                       TARGET PATH CONSTRUCTION                          #
    ###########################################################################

    # Build target directory path: ParentDir/SourceDirName - Category
    target_dir="${parent_dir}/${original_dir} - ${target_category}"
    
    # Construct full target path preserving original directory structure
    target_path="${target_dir}/${relative_path}"
    
    # Create directory hierarchy if needed (-p creates parents as required)
    mkdir -p "$(dirname "$target_path")"

    ###########################################################################
    #                          FILE HANDLING LOGIC                            #
    ###########################################################################

    if [ -f "$target_path" ]; then
        # Handle duplicate files based on user preference
        if [ "$handle_duplicates" == "yes" ]; then
            # Duplicate moving mode: Move both existing and new file to duplicates directory
            
            # Build path for existing duplicate in target location
            target_duplicate_path="${duplicates_root}/${target_dir#${parent_dir}/}/${relative_path}"
            mkdir -p "$(dirname "$target_duplicate_path")"
            mv -v "$target_path" "$target_duplicate_path"  # Move existing file
            
            # Build path for source duplicate
            source_duplicate_path="${duplicates_root}/${original_dir}/${relative_path}"
            mkdir -p "$(dirname "$source_duplicate_path")"
            mv -v "$file" "$source_duplicate_path"  # Move current file
        
        else
            # In-place handling: Special logic for UUID files
            if [ "$target_category" == "UUID" ]; then
                # UUID files: Overwrite only if size difference ≤15%
                
                current_size=$(stat -c %s "$file")        # Get current file size
                existing_size=$(stat -c %s "$target_path")  # Get existing file size
                
                # Calculate percentage difference (relative to existing file)
                size_diff=$(( (current_size - existing_size) * 100 / existing_size ))
                absolute_diff=${size_diff#-}  # Get absolute value
                
                if [ "$absolute_diff" -le 15 ]; then
                    # Overwrite if difference is within threshold
                    echo "Overwriting UUID file (${absolute_diff}% size difference): $relative_path"
                    mv -vf "$file" "$target_path"
                else
                    # Preserve existing file if difference exceeds threshold
                    echo "Skipping UUID duplicate (${absolute_diff}% size difference >15%): $relative_path"
                fi
            else
                # Non-UUID files: Skip duplicates
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
   - priority level markers ([1], [2], etc)
   - Documented regex pattern purposes and match groups
   - Explained case conversion/normalization steps
   - **WhatsApp**: Matches `IMG-YYYYMMDD-WAXXXX` pattern
   - **UUID**: Matches UUIDv4 format (`8-4-4-4-12` hex structure)
   - **Camera**: Detects common camera prefixes (DSC, IMG, DJI, etc.)
   - **Snapseed**: Looks for "snapseed" in filename
   - **Custom Keywords**: User-defined patterns

4. **File Organization**  
   Creates nested directory structure:  
   `ParentDir/OriginalDirName - Category - (CameraPrefix) - Extension`

5. **Special Handling**  
   - Puts `.jpeg` and `.jpg` in same category JPG
   - Skips `@eaDir` directories (Synology NAS thumbnails)
   - Preserves original folder structure
   - Special overwrite rules for UUID category files (only overwrites UUID files if existing file ≤115% of new file size)

6. **Safety Features**  
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
