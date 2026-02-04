# WordCount MapReduce on Amazon EMR

## Project Overview
A distributed WordCount implementation using Hadoop MapReduce on Amazon EMR with Python streaming.

## Repository Structure
```
.
├── mapper.py
├── reducer.py
└── README.md
```

## Dataset Information

**Source**: Simple English Wikipedia Dump  
**URL**: https://github.com/LGDoor/Dump-of-Simple-English-Wiki  
**File**: `corpus.tgz` (contains `corpus.txt`)  
**Size**: 32.8 MB (32,821,626 bytes)  
**Content**: Plain text from Simple English Wikipedia articles

## Complete Setup and Execution Commands

### 1. Connect to EMR Master Node
```bash
ssh -i "vockey.pem" hadoop@<MASTER_PUBLIC_DNS>
```

### 2. Download Dataset and Prepare HDFS
```bash
wget https://github.com/LGDoor/Dump-of-Simple-English-Wiki/raw/refs/heads/master/corpus.tgz
tar -xvzf corpus.tgz
hdfs dfs -mkdir -p /user/hadoop/input
hdfs dfs -mkdir -p /user/hadoop/output
hdfs dfs -put corpus.txt /user/hadoop/input/
hdfs dfs -ls /user/hadoop/input/
```

### 3. Create Mapper Script
```bash
nano mapper.py
```
**Content:**
```python
import sys

for line in sys.stdin:
    words = line.strip().split()
    for word in words:
        print(f"{word}\t1")
```
Save with: Ctrl+O, Enter, Ctrl+X

### 4. Create Reducer Script
```bash
nano reducer.py
```
**Content:**
```python
import sys

current_word = None
current_count = 0

for line in sys.stdin:
    word, count = line.strip().split('\t', 1)
    count = int(count)
    
    if current_word == word:
        current_count += count
    else:
        if current_word:
            print(f"{current_word}\t{current_count}")
        current_word = word
        current_count = count

if current_word:
    print(f"{current_word}\t{current_count}")
```
Save with: Ctrl+O, Enter, Ctrl+X

### 5. Run MapReduce Job (Basic)
```bash
hadoop jar /usr/lib/hadoop-mapreduce/hadoop-streaming.jar \
  -input /user/hadoop/input/ \
  -output /user/hadoop/output/ \
  -mapper mapper.py \
  -reducer reducer.py \
  -file mapper.py \
  -file reducer.py
```

### 6. Check Results
```bash
hdfs dfs -ls /user/hadoop/output/
hdfs dfs -cat /user/hadoop/output/part-00000 | head -20
```

### 7. For 2 Core vs 4 Core Experiment
**First run with 2 cores:**
```bash
hadoop jar /usr/lib/hadoop-mapreduce/hadoop-streaming.jar \
  -input /user/hadoop/input/ \
  -output /user/hadoop/output2nodes/ \
  -mapper mapper.py \
  -reducer reducer.py \
  -file mapper.py \
  -file reducer.py
```

**Then resize cluster to 4 cores via AWS Console and run again:**
```bash
hadoop jar /usr/lib/hadoop-mapreduce/hadoop-streaming.jar \
  -input /user/hadoop/input/ \
  -output /user/hadoop/output/ \
  -mapper mapper.py \
  -reducer reducer.py \
  -file mapper.py \
  -file reducer.py
```

**Verification:**
```bash
hdfs dfs -ls /user/hadoop/output
hdfs dfs -cat /user/hadoop/output/part-00000 | head
```

### 8. Performance Comparison Commands
```bash
# Check job metrics (look for time spent values)
# After 2 core run:
Total time spent by all maps in occupied slots (ms)=424344576
Total time spent by all reduces in occupied slots (ms)=187683840

# After 4 core run:
Total time spent by all maps in occupied slots (ms)=347409408
Total time spent by all reduces in occupied slots (ms)=314496000
```

## Performance Results Summary
- **Map phase**: Improved by 18.1% (424,344,576 ms → 347,409,408 ms)
- **Reduce phase**: Worsened by 67.6% (187,683,840 ms → 314,496,000 ms)
- **Overall improvement**: Only 2.6% faster despite doubling cores
- **Bottleneck**: Single reducer configuration causes network congestion
