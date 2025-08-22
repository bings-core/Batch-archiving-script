# Batch-archiving-script
Scripts to run archiving steps on multiple directories at once

You can copy over the scripts from the path below and run piece-by-piece in your screen session:

/sc/arion/projects/BiNGS/bings_analysis/projects/2025/bings_pipelines/bings_pipeline_batch_archiving/batch_archiving_script.sh

## 🖥️ Setup
```bash
screen -S archive_dirs
```
```bash
mkdir -p /sc/arion/projects/BiNGS/$USER/archiving

# Make a file containing the list of folders to be archived
folder_list="/sc/arion/projects/BiNGS/$USER/archiving/to_be_archived.txt"
# Create or empty the list of folders to be archived
> "$folder_list"

# Make a file to store filelist paths
filelists="/sc/arion/projects/BiNGS/$USER/archiving/to_be_archived_filelists.txt"
# Create or empty the output file
> "$filelists"

# ⚠️ Create a unique log file name each time to avoid overwriting in your next archiving run. ⚠️
# Make a file to store output logs for this archiving run
output_log="/sc/arion/projects/BiNGS/$USER/archiving/archiving_check_status_logs_20250707.txt"
# Create or empty the output log txt
> "$output_log"
# Set permission for archiving_bings user
setfacl -R -m user:archiving_bings:rwx $output_log

ml R/4.1.0
```
Add paths of folders to be archived. The other two txt files will remain empty for now.

My to_be_archived.txt looks like this:

/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/240823_VH01621_32_AAFMNLMM5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/240903_VH01621_34_AAG5233M5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/241002_VH01621_35_AAFCFCLM5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/241030_VH01621_37_AAG77MHM5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/241108_VH01621_38_AAGCJKVM5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/241220_VH01621_43_AAGCL3LM5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/250116_VH01621_44_AACCGJVHV
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/250425_VH01621_54_AAGJVLGM5
/sc/arion/projects/BiNGS/NextSeq_Data/COGIT_NS2k/Bernstein/250618_VH01621_65_AAH22J7M5

### Compress
```bash
# Loop through each folder in the list and compress
while IFS= read -r folder_name; do
  cd "$folder_name"
  echo "$folder_name"
  Rscript /sc/arion/projects/BiNGS/bings_analysis/code/R/utilities/tsm_archiving_tar.R compress "$folder_name"
done < "$folder_list" 
```

### Set permission for archiving_bings user
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

### Find filelist paths
```bash
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

### Switch to archiving_bings user
```bash
/opt/collab/bin/cologin archiving_bings

ml R/4.1.0

# ⚠️ Change ulukag01 in the paths below based on your minerva username. ⚠️
folder_list="/sc/arion/projects/BiNGS/ulukag01/archiving/to_be_archived.txt"
filelists="/sc/arion/projects/BiNGS/ulukag01/archiving/to_be_archived_filelists.txt"
# ⚠️ Change the output log path below based on your minerva username and unique log file name. ⚠️
output_log="/sc/arion/projects/BiNGS/ulukag01/archiving/archiving_check_status_logs_20250707.txt"
```

### Delete content
```bash
# Loop through each file in the list
while IFS= read -r tar_filelist_path; do
  Rscript /sc/arion/projects/BiNGS/bings_analysis/code/R/utilities/tsm_archiving_tar.R delete_tar_contents "${tar_filelist_path}"
  echo "Deleted contents of: $tar_filelist_path"
done < "$filelists"
```

### Archive
```bash
# Loop through each file in the list
while IFS= read -r tar_filelist_path; do
  Rscript /sc/arion/projects/BiNGS/bings_analysis/code/R/utilities/tsm_archiving_tar.R archive "${tar_filelist_path}"
  echo "Archived contents of: $tar_filelist_path"
done < "$filelists"
```

### Check status
```bash
# Loop through each file in the list
while IFS= read -r tar_filelist_path; do
  # Save the output of the Rscript command to the log file
  echo "Processing: $tar_filelist_path" >> "$output_log"
  Rscript /sc/arion/projects/BiNGS/bings_analysis/code/R/utilities/tsm_archiving_tar.R status "${tar_filelist_path}" >> "$output_log" 2>&1
  echo "Completed: $tar_filelist_path" >> "$output_log"
done < "$filelists"
```

#### ‼️ Check your $output_log file to make sure everything is successfully archived before deleting local tars. ‼️
#### ‼️ If anything has failed to be archived, remove their .filelist path from $filelists and then continue. ‼️

### Delete local copy
```bash
while IFS= read -r tar_filelist_path; do
  Rscript /sc/arion/projects/BiNGS/bings_analysis/code/R/utilities/tsm_archiving_tar.R delete_local_tar  "${tar_filelist_path}"
done < "$filelists"
```

### Exit the environment
```bash
exit # log out of archiving_bings user
exit # quit screen session
```

### Final reminder

Don’t forget to log the newly archived directories in the [BiNGS_Archiving_Logs.xlsx](https://mtsinai-my.sharepoint.com/:x:/g/personal/deniz_demircioglu_mssm_edu/EQGNb5S7pbZLsOl2YNnztHQB_UZBapieLhwLwjtqfohhtw?e=6Tpa3f).

 
### Retrieving archived dirs
#### Create or empty a file containing the list of folders to be retrieved
```bash
retrieve_list="/sc/arion/projects/BiNGS/$USER/archiving/to_be_retrieved.txt"
> "$retrieve_list"
```

Add paths of folders to be retrieved. All of them either need to be archived before 8/1/2025 or all of them are archived after 8/1/2025. 

My to_be_retrieved.txt looks like this:

/sc/arion/projects/BiNGS/bings_omics/data/bings/2023/ernestoguccione/EG12_slim/EG12_slim_20240913_15h58m12s.tar.gz.filelist
/sc/arion/projects/BiNGS/bings_omics/data/bings/2023/ernestoguccione/EG18_slim/EG18_slim_20240913_18h53m36s.tar.gz.filelist
/sc/arion/projects/BiNGS/bings_omics/data/bings/2023/ernestoguccione/EG19_slim/EG19_slim_20240913_20h51m07s.tar.gz.filelist
/sc/arion/projects/BiNGS/bings_omics/data/bings/2023/ernestoguccione/Exp5_slim/Exp5_slim_20240913_23h28m40s.tar.gz.filelist

#### Switch to archiving_bings user
```bash
/opt/collab/bin/cologin archiving_bings

ml R/4.1.0

# ⚠️ Change ulukag01 in the paths below based on your minerva username. ⚠️
retrieve_list="/sc/arion/projects/BiNGS/ulukag01/archiving/to_be_retrieved.txt"
```

#### Retrieve if all of them are archived before 8/1/2025
```bash
# Loop through each file in the list
while IFS= read -r tar_filelist_path; do
  Rscript /sc/arion/projects/BiNGS/bings_analysis/code/R/utilities/tsm_archiving_tar.R retrieve "${tar_filelist_path}" --legacy
  echo "Retrieved contents of: $tar_filelist_path"
done < "$retrieve_list"
```

#### Retrieve if all of them are archived after 8/1/2025
```bash
# Loop through each file in the list
while IFS= read -r tar_filelist_path; do
  Rscript /sc/arion/projects/BiNGS/bings_analysis/code/R/utilities/tsm_archiving_tar.R retrieve "${tar_filelist_path}"
  echo "Retrieved contents of: $tar_filelist_path"
done < "$retrieve_list"
```
