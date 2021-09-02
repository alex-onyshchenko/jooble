1. Created script that sanitize file before archiving.

```bash
#!/bin/bash

set -e

log_file="$1"

declare -a RM_LINE_PATTERNS=("192.168.0.100" "127.0.0.1" "^00:00.*/reinit$")
REPLACE_PATTERN="\$[a-fA-F0-9]{32}\$\S{2,3}@\S+\.\S+"
REPLACE_STR="*****"

while IFS= read -r line
do
  keep=1
  for pattern in "${RM_LINE_PATTERNS[@]}"
  do
    if [[ "$line" =~ "$pattern" ]]; then
      keep=0
    fi
  done
  [[ $keep == 1 ]] && echo "$line" | sed -e "s/$REPLACE_PATTERN/$REPLACE_STR/g"
done <$log_file > o
mv o $log_file
```

Here we loop over lines in the log file and check if the line meets regex if so we don't copy the line to temporary file.
Also we replace sensitive data with stars.
In the end we overwrite original file.

2. Created script that rotates log file.

```bash
#!/bin/bash

set -e

ARCHIVE_DIR="./archive"

log_file="$1"

name=$(basename $log_file)
timestamp=$(date '+%Y-%m-%d_%H:%M:%S')

./sanitize.sh $log_file

tar -cf "$ARCHIVE_DIR/${name}-${timestamp}.tar.gz" $log_file

i=0
for filename in $(ls -tU $ARCHIVE_DIR/$name-*.tar.gz); do
  if [[ $i -ge 10 ]]; then
    scp -i  /.ssh/backuper.pub "$filename" "backuper@192.168.0.43:/var/log/storage/$name" && rm -f "$filename"
  fi
  i=$(( $i + 1 ))
done
```

Here we call sanitize script to process log file. Then we archive it to the archive directory.
In the end we loop over sorted list of same type log files and copy all except last 10 archives.
Then we sent over secure copy old logs to the remote server and remove sent files.

3. Created script that generate cron config.

```bash
#!/bin/bash

set -e

get_abs_filename() {
  echo "$(cd "$(dirname "$1")" && pwd)/$(basename "$1")"
}

ROTATE_SCRIPT_PATH=$(get_abs_filename rotate.sh)
LOG_DIR="$(pwd)"

for filename in *.log; do
  filename_path="$LOG_DIR/$filename"
  if [[ "$filename" == "access.log" ]];then
    printf '*/15  * * * * %s %s\n' "$ROTATE_SCRIPT_PATH" "$filename_path"
  fi
  printf '0 0 * * * %s %s\n' "$ROTATE_SCRIPT_PATH" "$filename_path"
done
```

The output of this script will be as following:

```
*/15  * * * * /Users/parol1/Desktop/Projects/jooble/rotate.sh /Users/parol1/Desktop/Projects/jooble/access.log
0 0 * * * /Users/parol1/Desktop/Projects/jooble/rotate.sh /Users/parol1/Desktop/Projects/jooble/access.log
0 0 * * * /Users/parol1/Desktop/Projects/jooble/rotate.sh /Users/parol1/Desktop/Projects/jooble/chunga.log
0 0 * * * /Users/parol1/Desktop/Projects/jooble/rotate.sh /Users/parol1/Desktop/Projects/jooble/custom.log
0 0 * * * /Users/parol1/Desktop/Projects/jooble/rotate.sh /Users/parol1/Desktop/Projects/jooble/error.log
0 0 * * * /Users/parol1/Desktop/Projects/jooble/rotate.sh /Users/parol1/Desktop/Projects/jooble/seo.log
```

It will run rotate script when needed. We need to copy this output to the cron config.
