# Bash Scripting Basics

##  Introduction

Shell scripting là kỹ năng quan trọng nhất cho DevOps Engineers. Nó giúp tự động hóa các tác vụ lặp đi lặp lại.

## First Script

```bash
#!/bin/bash
# my_first_script.sh

echo "Hello, DevOps World!"
echo "Current directory: $(pwd)"
echo "Current user: $USER"
```

**Make executable and run:**
```bash
chmod +x my_first_script.sh
./my_first_script.sh
```

## 🔤 Variables

```bash
#!/bin/bash

# Variable assignment (no spaces around =)
NAME="DevOps Engineer"
AGE=25
PI=3.14

# Using variables
echo "I am a $NAME"
echo "Age: ${AGE}"

# Command substitution
CURRENT_DATE=$(date +%Y-%m-%d)
echo "Today is: $CURRENT_DATE"

# Read user input
read -p "Enter your name: " USERNAME
echo "Hello, $USERNAME!"

# Environment variables
echo "Home: $HOME"
echo "Path: $PATH"
```

## 🔢 Arrays

```bash
#!/bin/bash

# Define array
FRUITS=("Apple" "Banana" "Orange" "Mango")

# Access elements
echo "First fruit: ${FRUITS[0]}"
echo "All fruits: ${FRUITS[@]}"
echo "Number of fruits: ${#FRUITS[@]}"

# Loop through array
for fruit in "${FRUITS[@]}"; do
    echo "I like $fruit"
done

# Add element
FRUITS+=("Grape")
```

## 🔀 Conditionals

```bash
#!/bin/bash

# if-else
if [ $AGE -gt 18 ]; then
    echo "Adult"
else
    echo "Minor"
fi

# if-elif-else
SCORE=85
if [ $SCORE -ge 90 ]; then
    echo "Grade: A"
elif [ $SCORE -ge 80 ]; then
    echo "Grade: B"
elif [ $SCORE -ge 70 ]; then
    echo "Grade: C"
else
    echo "Grade: F"
fi

# File tests
if [ -f /etc/passwd ]; then
    echo "File exists"
fi

if [ -d /var/log ]; then
    echo "Directory exists"
fi

# String comparison
NAME="DevOps"
if [ "$NAME" == "DevOps" ]; then
    echo "Match!"
fi

# Logical operators
if [ $AGE -gt 18 ] && [ $AGE -lt 65 ]; then
    echo "Working age"
fi

if [ "$ENV" == "prod" ] || [ "$ENV" == "production" ]; then
    echo "Production environment"
fi
```

### **Test Operators**

| Operator | Description | Example |
|----------|-------------|---------|
| `-eq` | Equal | `[ $a -eq $b ]` |
| `-ne` | Not equal | `[ $a -ne $b ]` |
| `-gt` | Greater than | `[ $a -gt $b ]` |
| `-lt` | Less than | `[ $a -lt $b ]` |
| `-ge` | Greater or equal | `[ $a -ge $b ]` |
| `-le` | Less or equal | `[ $a -le $b ]` |
| `-z` | String is empty | `[ -z "$str" ]` |
| `-n` | String not empty | `[ -n "$str" ]` |
| `-f` | File exists | `[ -f file ]` |
| `-d` | Directory exists | `[ -d dir ]` |
| `-r` | Readable | `[ -r file ]` |
| `-w` | Writable | `[ -w file ]` |
| `-x` | Executable | `[ -x file ]` |

## 🔁 Loops

### **For Loop**

```bash
#!/bin/bash

# Basic for loop
for i in 1 2 3 4 5; do
    echo "Number: $i"
done

# Range
for i in {1..10}; do
    echo $i
done

# Step
for i in {0..100..10}; do
    echo $i
done

# Files
for file in /var/log/*.log; do
    echo "Processing: $file"
done

# C-style
for ((i=1; i<=10; i++)); do
    echo "Count: $i"
done
```

### **While Loop**

```bash
#!/bin/bash

# While loop
COUNT=1
while [ $COUNT - le 5 ]; do
    echo "Count: $COUNT"
    ((COUNT++))
done

# Read file line by line
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/hosts

# Infinite loop
while true; do
    echo "Running..."
    sleep 5
done
```

### **Until Loop**

```bash
#!/bin/bash

COUNT=0
until [ $COUNT -eq 5 ]; do
    echo "Count: $COUNT"
    ((COUNT++))
done
```

## Functions

```bash
#!/bin/bash

# Define function
function greet() {
    echo "Hello, $1!"
}

# Call function
greet "DevOps"

# Function with return
function add() {
    local result=$(($1 + $2))
    echo $result
}

SUM=$(add 5 10)
echo "Sum: $SUM"

# Function with multiple params
function deploy_app() {
    local app_name=$1
    local environment=$2
    local version=$3
    
    echo "Deploying $app_name v$version to $environment..."
}

deploy_app "myapp" "production" "1.0.0"
```

## 📥 Input/Output

```bash
#!/bin/bash

# Read from user
read -p "Enter environment (dev/prod): " ENV
echo "Selected: $ENV"

# Read with default
read -p "Enter port [8080]: " PORT
PORT=${PORT:-8080}
echo "Port: $PORT"

# Read password (hidden)
read -sp "Enter password: " PASSWORD
echo

# Redirect output
echo "Log message" > output.txt      # Overwrite
echo "Another log" >> output.txt     # Append

# Redirect errors
command 2> error.log                 # Errors only
command > output.log 2>&1            # Both stdout & stderr
command &> all.log                   # Both (shorthand)

# Discard output
command > /dev/null 2>&1
```

## Practical Examples

### **1. System Backup Script**

```bash
#!/bin/bash

# backup.sh - Backup important directories

BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="backup_${DATE}.tar.gz"

echo "Starting backup..."
tar -czf "${BACKUP_DIR}/${BACKUP_FILE}" \
    /etc \
    /home \
    /var/www \
    2>/dev/null

if [ $? -eq 0 ]; then
    echo "Backup completed: ${BACKUP_FILE}"
else
    echo "Backup failed!"
    exit 1
fi

# Delete backups older than 7 days
find $BACKUP_DIR -name "backup_*.tar.gz" -mtime +7 -delete
echo "Old backups cleaned up"
```

### **2. Service Health Check**

```bash
#!/bin/bash

# healthcheck.sh - Check service status

SERVICE="nginx"

if systemctl is-active --quiet $SERVICE; then
    echo "$SERVICE is running"
else
    echo "$SERVICE is down"
    echo "Attempting to restart..."
    systemctl restart $SERVICE
    
    sleep 5
    
    if systemctl is-active --quiet $SERVICE; then
        echo "$SERVICE restarted successfully"
    else
        echo "Failed to restart $SERVICE"
        # Send alert
        echo "Service $SERVICE is down!" | mail -s "ALERT" admin@example.com
        exit 1
    fi
fi
```

### **3. Deployment Script**

```bash
#!/bin/bash

# deploy.sh - Deploy application

set -e  # Exit on error

APP_NAME="myapp"
ENVIRONMENT=${1:-dev}
VERSION=${2:-latest}
DEPLOY_DIR="/var/www/${APP_NAME}"

echo "Deploying ${APP_NAME} v${VERSION} to ${ENVIRONMENT}..."

# Pull latest code
cd $DEPLOY_DIR
git fetch origin
git checkout $VERSION
git pull origin $VERSION

# Install dependencies
if [ -f "package.json" ]; then
    npm install --production
elif [ -f "requirements.txt" ]; then
    pip install -r requirements.txt
fi

# Run database migrations
python manage.py migrate

# Restart application
systemctl restart $APP_NAME

echo "Deployment completed successfully!"
```

### **4. Log Analyzer**

```bash
#!/bin/bash

# analyze_logs.sh - Analyze web server logs

LOG_FILE="/var/log/nginx/access.log"
REPORT_FILE="/tmp/log_report_$(date +%Y%m%d).txt"

echo "=== Log Analysis Report ===" > $REPORT_FILE
echo "Date: $(date)" >> $REPORT_FILE
echo >> $REPORT_FILE

# Total requests
TOTAL=$(wc -l < $LOG_FILE)
echo "Total Requests: $TOTAL" >> $REPORT_FILE

# Top 10 IP addresses
echo >> $REPORT_FILE
echo "Top 10 IP Addresses:" >> $REPORT_FILE
awk '{print $1}' $LOG_FILE | sort | uniq -c | sort -rn | head -10 >> $REPORT_FILE

# Top 10 URLs
echo >> $REPORT_FILE
echo "Top 10 URLs:" >> $REPORT_FILE
awk '{print $7}' $LOG_FILE | sort | uniq -c | sort -rn | head -10 >> $REPORT_FILE

# Status code distribution
echo >> $REPORT_FILE
echo "Status Codes:" >> $REPORT_FILE
awk '{print $9}' $LOG_FILE | sort | uniq -c | sort -rn >> $REPORT_FILE

# Error count (4xx and 5xx)
ERRORS=$(awk '$9 ~ /^[45]/ {count++} END {print count}' $LOG_FILE)
echo >> $REPORT_FILE
echo "Total Errors (4xx/5xx): $ERRORS" >> $REPORT_FILE

cat $REPORT_FILE
```

## Best Practices

1. **Always use shebang**
   ```bash
   #!/bin/bash
   ```

2. **Use set flags**
   ```bash
   set -e  # Exit on error
   set -u  # Error on undefined variable
   set -o pipefail  # Pipeline fails if any command fails
   ```

3. **Quote variables**
   ```bash
   # Good
   if [ "$VAR" == "value" ]; then
   
   # Bad
   if [ $VAR == "value" ]; then
   ```

4. **Check command success**
   ```bash
   if command; then
       echo "Success"
   else
       echo "Failed"
       exit 1
   fi
   ```

5. **Use functions for reusability**

6. **Add comments**

7. **Use meaningful variable names**

8. **Handle errors gracefully**

---

**Tiếp theo**: [Process Management →](../process-management/processes.md)
