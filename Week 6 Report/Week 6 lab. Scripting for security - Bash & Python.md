
First we check if our VM is online or not:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112190559.png>)

And yes it is. Now I download the zip my instructor gave me a link to and unzip it:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112190932.png>)

Now I check whether or not Python is installed on my system. I am not checking bash, as it is installed on every system:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112191027.png>)

## Part A - Bash
### Task A0-A1

 Now as I have further tasks to complete, I will first look at the data, so I understand the way it is structured:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112191505.png>)

As my first task is to turn over a thousand log lines into a short report that names the source IPs behind the failed logins and how hard each one hit, I now check the way failed logins are structured:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112191835.png>)

As we see there are 286 failed attempts to log into the system. Also, let us check the authentication failure count:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112192247.png>)

And we got 258 authentication failures. Now let's compare the output of the two `grep` commands I just wrote:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112192742.png>)

And if we look closely, the sshd entries overlap. Also, the second `grep` command, gave different types of output, and the three most common types are:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112193256.png>)

The one command, whose matches have nothing to do with SSH at all, is the `grep "authentication failure"...`.

If I added the two counts together I would not get the real count of the failed logins, because as we have seen earlier, their sshd entries mostly overlap.

And the conclusion is that I must specify the keyword as much as possible, so that I do not get unnecessary data alongside the data I need to search for.

### Task A2-A4

My next task is to pull out the attacker IP, from the failed lines, extracting just the source IP with awk. For that:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112194842.png>)

I added some commands as well, so to shorten the output amount, and show the number of attempts each IP address made to log in and failed. And the one in the first place has failed 173 times. My next task to identify whether the first-place IP address is a documentation/test range or a real network, and for that I searched on the web, and it showed that it is reserved and does not belong to the public internet:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112195332.png>)

And by looking at the difference between the bottom 3 IP addresses attempts, and the top ones, the difference an attack and typos are clear, by the amount of their failed login attempts.

Failed logins are only noise until I get an answer to the real question. Next, I list every successful login:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112200028.png>)

And now I check whether any of the attackers got in:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112195841.png>)

And as we see, the IP address that hit the hardest managed to log into the system. Why only this IP? Because the filter showed only this one, and in addition to this, we can see that the actual people, who should be able to log in, use one subnet, which is 10.0.0.0/16. We see that from failed login attempts as well.

Now as I found an attacker managed to log into the system, I check what they did next:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112200825.png>)

And, some great news. They did nothing after logging in, just stayed several minutes, and disconnected. To double check, I look at the end of the full log, that contains their IP, and see if I have not missed anything:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112200943.png>)

And everything is correct.

### Task A5-A6

Now, my task is to make a reusable script, which should take the log file as its argument, print a title, the total count, and the top 5 IP ranking:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112201249.png>)

I first create the file and open it in editor. This way it will do both at once. For my next step I start writing the script:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112202445.png>)

Now it is time to run it. And the result is:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112202504.png>)

The same as in Task A3. Now my task is to save a dated report:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112202715.png>)

Now there are some challenges. The first one is to add a Top-5 attacked usernames section to report.sh. For that:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112203433.png>)

Here is the final code:
```bash
#!/bin/bash

myLog=$1

echo -e "Welcome, this is the automated script"

totalCount=$(grep -c "Failed password" "$myLog")
echo -e "\nThe total count of failed login attempts is: $totalCount\n"

echo "And the top 5 ranking IP addresses that tried to login, but failed are:"
awk '$6 == "Failed" && $7 == "password" {print $(NF-3)}' "$myLog" | sort | uniq -c | sort -rn | head -5

echo -e "\nThe top attacked usernames are:"
awk '$6 == "Failed" && $7 == "password" {print $(NF-5)}' "$myLog" | sort | uniq -c | sort -nr | head -5
```

I added the last two lines to the script, and now I run it:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112203507.png>)

Counting the hidden attempts:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112203606.png>)

There are 14 collapsed lines, and the total count of failed logins that it will give as a result is:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112203702.png>)

As we can see 12 lines says "message repeated 2 times" and 2 lines say "message repeated 3 times": 12 x 2 + 2 x 3 = 30. So my correct amount of `Failed passwords` is 286 + 30 = 316.

To conclude Part A, the top attacker is 203.0.113.7 with 173 failures, which after managing to log into the system, did nothing and later just logged out. Additionally, the total amount of attacks is 316.

## Part B - Python: find the alarms in a SIEM export

I will have to work with events.csv this time. Counting and grouping structured data is where Python beats Bash. As it is provided, this is a SIEM export of the same night and the same servers as Part A — so I can, and should, cross-check what I found in Part A against it. I have to build detect.py from scratch to print only the IPs over an alarm threshold.

### Task B1

My first task is to write a code that opens events.csv with DictReader and print each row's user and result:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112205347.png>)

Now, I run it, and this is the result:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112205437.png>)

### Task B2

My next task is to tally the failures per IP. For that, I will first remove the `print(...)` statement from the previous task, and then:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112205948.png>)

### Task B3

Now my task is to raise the alarms, for every IP with count over 50:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112210532.png>)

And when we run the program, the result is:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112210548.png>)

The program printed four IP addresses. If we check it with the very first part, we can see that the 203.0.113.7 and 198.51.100.23 both appeared here as well. And also we can see that the attempts count are higher than in the previous checking. To answer the question, no, the counts do not match, as the CSV count is bigger. report.sh only counted failed SSH logins on web01, but the CSV covers all hosts, and both 203.0.113.7 and 198.51.100.23 also have failed SSH attempts on vpn01. 

As it is provided, one of the IPs failed 38 times. My task is to find it, look at its first and last attempts, and validate whether or not these failed attempts were legitimate. So I change the function, to get exactly that IP address:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112211807.png>)

run the file:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112211840.png>)

and now grep it, to read the timestamp to understand if this IP address made safe login attempts, or it was an attack as well:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112211957.png>)

And as we can see, the difference is 2 minutes, so most probably, it was an automated script, hence, it was not a safe login attempt.

Now my task is to change the threshold to 30 and run it again:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112212221.png>)

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112212428.png>)

As we can see, the IP address that I just  found manually appeared here as well. Also in the task it is mentioned, that there is one more IP address near the threshold, and to understand if the login attempts were safe or not. For that:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112212536.png>)

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112212546.png>)

As we can see, one more IP address has appeared. Now I will check the timestamp, to understand if the login attempts were safe or not:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112212645.png>)

The difference is just 30 seconds, hence, it is an attack as well.

The count alone misses a rule, which is if the attempts are valid or not, and it should be understandable by the time difference of login attempts. 

To check if any alarmed IP has `success` as `result` value, I do (so that I don't have to type it by hand everytime, I will just change the python code):

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112220106.png>)

Here is the final code:

```python
import sys, csv

myLog = sys.argv[1]
fails = dict()
success = []

def countCmp(numArg):
	return numArg > 20

with open(myLog) as f:
	for row in csv.DictReader(f):
		if row['result'] == 'fail':
			fails[row['src_ip']] = fails.get(row['src_ip'], 0) + 1
		elif row['result'] == 'success':
			success.append(row['src_ip'])

for ip, count in fails.items():
	cnt = countCmp(count)
	if (cnt):
		print(f"ALARM, THE IP ADDRESS {ip} HAS MADE {count} ATTEMPTS TO CONNECT")
	if (cnt and ip in success):
		print(f"THE IP ADDRESS {ip} MANAGED TO LOG IN AS WELL")
```

And the result is:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112220127.png>)

And it is the same IP address that appeared in the previous answer as well. And to be exact it was in **Task A4**.

In conclusion, the `events.csv` provided a critical information, that the IP address 203.0.113.7 managed to log into the system as well.

## Part C - Automating the enumeration, with AI

### Task C1-C2


![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112234158.png>)
![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112234300.png>)
![PowerShell Get-LocalUser output](<Week6/Pasted image 20261112234321.png>)

and in order not to do this all by hand, the script should do this all. Consequently, the next task is to write a script that runs my `report.sh`, on every `*.log` in `logs/` automatically. And specifically for this task I will create another file called `report2.sh` (copy of an old file, which I am going to modify):

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261113000045.png>)

Here is the final code:

```bash
#!/bin/bash

for x in logs/*.log; do
	echo "Checking "$x" log file"
	totalCount=$(grep -c "Failed password" "$x")
	echo -e "\nThe total count of failed login attempts is: $totalCount\n"

	echo "And the top 5 ranking IP addresses that tried to login, but failed are:"
	awk '$6 == "Failed" && $7 == "password" {print $(NF-3)}' "$x" | sort | uniq -c | sort -rn | head -5

	echo -e "\nThe top attacked usernames are:"
	awk '$6 == "Failed" && $7 == "password" {print $(NF-5)}' "$x" | sort | uniq -c | sort -nr | head -5
done
```

And when we run it, we get:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261113000107.png>)
![PowerShell Get-LocalUser output](<Week6/Pasted image 20261113000117.png>)

### Task C3

The next task is to build a fleet report with AI, which goes over every log in logs/ and prints a single combined summary: which IP attacked which hosts, and how many times on each. And one necessary thing to mention, is that one must never use a real line from the logs, but make one up as a reference for the prompt:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261114132044.png>)

And I got the following result:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261114132214.png>)
![PowerShell Get-LocalUser output](<Week6/Pasted image 20261114132242.png>)

Here is the final code:

```bash
#!/usr/bin/env bash

dir="${1:-.}"
min="${2:-10}"

if [[ ! -d "$dir" ]]; then
  echo "Error: '$dir' is not a directory" >&2
  exit 1
fi

shopt -s nullglob
files=("$dir"/*.log)

if (( ${#files[@]} == 0 )); then
  echo "No .log files found in '$dir'" >&2
  exit 1
fi

result=$(
  awk -v min="$min" '
    /Failed password for/ {
      host = $4
      ip = ""
      for (i = 1; i < NF; i++) {
        if ($i == "from") { ip = $(i + 1); break }
      }
      if (ip != "") {
        per[ip SUBSEP host]++
        total[ip]++
      }
    }
    END {
      for (k in per) {
        split(k, a, SUBSEP)
        if (total[a[1]] >= min)
          printf "%s\t%s\t%d\t%d\n", a[1], a[2], per[k], total[a[1]]
      }
    }
  ' "${files[@]}" | sort -t$'\t' -k1,1V -k3,3nr |
  awk -F'\t' '
    $1 != prev {
      printf "%s%s  (total: %d)\n", (NR > 1 ? "\n" : ""), $1, $4
      prev = $1
    }
    { printf "    %-20s %d\n", $2, $3 }
  '
)

if [[ -z "$result" ]]; then
  echo "No IP made $min or more failed attempts."
else
  echo "$result"
fi
```

And when I run the script, the output is the following:

![PowerShell Get-LocalUser output](<Week6/Pasted image 20261114132419.png>)

where `.` means the current directory and the number following that is the threshold, which shows the minimum number of attacks that were carried out by an IP address. As we can see 203.0.113.7 attacked every server, and it tops the other IP addresses. 