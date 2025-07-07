# Batch-archiving-script
Scripts to run archiving steps on multiple directories at once

## 🖥️ Setup
```bash
screen -S archive_dirs
```
```bash
mkdir -p /sc/arion/projects/BiNGS/$USER/archiving

# File containing the list of folders to be archived
folder_list="/sc/arion/projects/BiNGS/$USER/archiving/to_be_archived.txt"
# Create or empty the list of folders to be archived
> "$folder_list"

# Output log file for this archiving run
output_log="/sc/arion/projects/BiNGS/$USER/archiving/archiving_check_status_logs_20250707.txt"
# Create or empty the list of folders to be archived
> "$output_log"
# Set permission for archiving_bings user
setfacl -R -m user:archiving_bings:rwx $output_log

```
Add paths of folders to be archived. My to_be_archived.txt looks like this:

/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/240823_VH01621_32_AAFMNLMM5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/240903_VH01621_34_AAG5233M5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/241002_VH01621_35_AAFCFCLM5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/241030_VH01621_37_AAG77MHM5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/241108_VH01621_38_AAGCJKVM5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/241220_VH01621_43_AAGCL3LM5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/250116_VH01621_44_AACCGJVHV
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/250425_VH01621_54_AAGJVLGM5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/250618_VH01621_65_AAH22J7M5

### Step 1: Compress
```bash
ml R/4.1.0

# Loop through each folder in the list and compress
while IFS= read -r folder_name; do
  cd "$folder_name"
  echo "$folder_name"
  Rscript /sc/arion/projects/BiNGS/bings_analysis/code/R/utilities/tsm_archiving_tar.R compress "$folder_name"
done < "$folder_list" 
```

### Step 2: Set permission for archiving_bings user
```bash
# Loop through each folder in the list
while IFS= read -r folder_name; do
  # Check if the file already has the user:archiving_bings permissions
  if ! getfacl "$folder_name" | grep -q "user:archiving_bings:rwx"; then
    # Apply the setfacl command if permission does not exist
    setfacl -R -m user:archiving_bings:rwx "$folder_name"
  else
    echo "Skipping $folder_name: user:archiving_bings already has rwx"
  fi
done < "$folder_list"
```

### Step 3: Find filelist paths
```bash
# Make a new txt with filelist paths
filelists="/sc/arion/projects/BiNGS/$USER/archiving/to_be_archived_filelists.txt"

# Create or empty the output file
> "$filelists"

# Read each line from the input file
while IFS= read -r dir_path; do
    # Use a wildcard to search for .filelist files in the directory
    filelist=$(find "$dir_path" -name "$(basename "$dir_path")_*.tar.gz.filelist" 2>/dev/null)
    
    # If filelist is found, append it to the output file
    if [[ -n $filelist ]]; then
        echo "$filelist" >> "$filelists"
    else
        echo "No .filelist found for $dir_path"
    fi
done < "$folder_list"
```
