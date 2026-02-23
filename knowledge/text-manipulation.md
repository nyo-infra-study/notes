# Linux Text Processing Commands

Text processing commands are essential for manipulating log files, configuration files, and data streams in Linux. These tools are particularly useful for debugging, CI/CD pipelines, and handling large amounts of data.

Source: [Earthly - Linux Text Processing Commands](https://earthly.dev/blog/linux-text-processing-commands)

## Core Concepts

### Standard Streams

- **stdin** (Standard Input): Input to a program (keyboard, file, or pipe)
- **stdout** (Standard Output): Normal output from a program
- **stderr** (Standard Error): Error messages from a program

### Pipes and Redirection

**Pipes** (`|`): Connect output of one command to input of another
```bash
echo "Hello World" | tr [a-z] [A-Z]
# Output: HELLO WORLD
```

**Redirection Operators**:
- `>` - Redirect output to file (overwrite)
- `>>` - Append output to file
- `<` - Read input from file
- `2>` - Redirect errors to file
- `2>>` - Append errors to file

```bash
# Save output
curl https://example.com > page.html

# Append to file
echo "new line" >> file.txt

# Capture errors
kubectl apply -f config.yaml 2> errors.log
```

## Essential Commands

### cut - Extract Columns

Extract specific fields from delimited text.

```bash
# Get first field (tab-delimited by default)
gh gist list | cut -f 1

# CSV: extract name column
cut -d ',' -f 1 data.csv

# Extract range of fields
cut -d ',' -f 2-4 data.csv
```

### sort - Order Lines

Arrange lines in specific order.

```bash
# Ascending order (default)
sort names.txt

# Descending order
sort -r names.txt

# Ignore case
sort -f names.txt

# Save to file
sort names.txt -o sorted.txt
```

### uniq - Remove Duplicates

Filter out repeated adjacent lines.

```bash
# Remove adjacent duplicates
uniq file.txt

# Remove all duplicates (requires sort)
sort file.txt | uniq

# Count occurrences
uniq -c file.txt

# Show only unique lines
sort file.txt | uniq -u
```

### grep - Search Patterns

Search for patterns in text.

```bash
# Basic search
grep "error" app.log

# Case-insensitive
grep -i "error" app.log

# Recursive search
grep -r "TODO" src/

# Show line numbers
grep -n "error" app.log

# Invert match (exclude)
grep -v "debug" app.log
```

### sed - Stream Editor

Perform text transformations.

```bash
# Replace first occurrence per line
sed 's/k8s/kubernetes/' file.md

# Replace all occurrences (global)
sed 's/k8s/kubernetes/g' file.md

# Delete lines matching pattern
sed '/debug/d' app.log

# Replace in-place
sed -i 's/old/new/g' file.txt
```

### awk - Pattern Scanning

Advanced text processing and data extraction.

```bash
# Print specific column
awk '{print $2}' file.txt

# CSV: print second field
awk -F ',' '{print $2}' data.csv

# Sum column values
awk -F ',' '{sum+=$2} END {print sum}' data.csv

# Skip header row
awk -F ',' 'NR>1 {print $2}' data.csv

# Filter by condition
awk -F ',' '$2 > 20 {print $1}' data.csv
```

### tr - Translate Characters

Transform or delete characters.

```bash
# Uppercase to lowercase
echo "Hello World" | tr [A-Z] [a-z]

# Alternative syntax
echo "Hello" | tr [:upper:] [:lower:]

# Delete characters
echo "abc123" | tr -d [:digit:]

# Squeeze repeated characters
echo "hello    world" | tr -s ' '
```

### wc - Word Count

Count lines, words, and characters.

```bash
# Count all (lines, words, characters)
wc file.txt

# Count lines only
wc -l file.txt

# Count words only
wc -w file.txt

# Count characters only
wc -c file.txt
```

### tac - Reverse Lines

Print file in reverse order (opposite of cat).

```bash
# Reverse file content
tac file.txt

# Useful for reading logs (newest first)
tac /var/log/app.log
```

### tee - Split Output

Write to both stdout and file simultaneously.

```bash
# View and save output
docker inspect image | jq . | tee output.json

# Append mode
command | tee -a file.txt
```

### xargs - Build Commands

Pass output as arguments to another command.

```bash
# Delete multiple files
ls *.tmp | xargs rm

# Interactive mode (confirm each)
ls *.tmp | xargs -p rm

# One argument at a time
gh gist list | cut -f 1 | xargs -n 1 gh gist delete

# Parallel execution
find . -name "*.log" | xargs -P 4 gzip
```

## Practical Examples

### Log Analysis

```bash
# Count error occurrences
grep -i "error" app.log | wc -l

# Find unique error messages
grep "ERROR" app.log | sort | uniq -c | sort -rn

# Extract timestamps of errors
grep "ERROR" app.log | cut -d ' ' -f 1-2

# View latest logs first
tac app.log | grep "ERROR" | head -20
```

### CSV Processing

```bash
# Extract specific columns
cut -d ',' -f 1,3 data.csv

# Calculate average of column
awk -F ',' 'NR>1 {sum+=$2; count++} END {print sum/count}' data.csv

# Filter rows by condition
awk -F ',' '$2 > 100 {print}' data.csv
```

### Text Transformation

```bash
# Convert repository name to lowercase (for Docker tags)
echo $GITHUB_REPOSITORY | tr '[:upper:]' '[:lower:]'

# Replace all occurrences in multiple files
find . -name "*.md" | xargs sed -i 's/old-term/new-term/g'

# Remove blank lines
sed '/^$/d' file.txt
```

### Data Extraction

```bash
# Extract IDs from command output
gh gist list | cut -f 1

# Get unique IP addresses from logs
grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' access.log | sort | uniq

# Count HTTP status codes
awk '{print $9}' access.log | sort | uniq -c | sort -rn
```

## Modern Alternatives

These Rust-based tools offer improved performance and user experience:

### ripgrep (rg) - Better grep

```bash
# Recursive search with line numbers (default)
rg "pattern"

# Case-insensitive
rg -i "pattern"

# Search specific file types
rg -t py "import"
```

### bat - Better cat

```bash
# Syntax highlighting and line numbers
bat file.py

# Show non-printable characters
bat -A file.txt
```

### fd - Better find

```bash
# Simple file search
fd pattern

# Search by extension
fd -e py

# Execute command on results
fd -e log -x gzip
```

## Regular Expressions

Common regex patterns for text processing:

```bash
# Match any characters: *
grep "k.*s" file.txt  # matches k8s, kubernetes, etc.

# Match word boundaries: \b
grep "\bword\b" file.txt

# Match start of line: ^
grep "^Error" file.txt

# Match end of line: $
grep "failed$" file.txt

# Match digits: [0-9] or \d
grep "[0-9]\{3\}" file.txt  # 3 digits

# Match email pattern
grep -E "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" file.txt
```

## Use Cases in Our Infrastructure

### Kubernetes Logs

```bash
# Get pod logs and filter errors
kubectl logs pod-name | grep ERROR | tee errors.log

# Count restarts by pod
kubectl get pods -o wide | awk '{print $4}' | sort | uniq -c

# Extract container IDs
kubectl get pods -o json | jq -r '.items[].status.containerStatuses[].containerID' | cut -d '/' -f 3
```

### CI/CD Pipelines

```bash
# Extract build version from file
grep "version" package.json | cut -d '"' -f 4

# Validate configuration files
find . -name "*.yaml" | xargs -n 1 yamllint

# Generate lowercase image tags
echo $IMAGE_NAME | tr '[:upper:]' '[:lower:]'
```

### Configuration Management

```bash
# Find all TODO comments
rg "TODO|FIXME" --type yaml

# Replace environment variables
sed 's/${ENV}/production/g' config.template > config.yaml

# Extract unique hostnames from configs
grep -h "host:" *.yaml | cut -d ':' -f 2 | sort | uniq
```

---

Content was rephrased for compliance with licensing restrictions.
