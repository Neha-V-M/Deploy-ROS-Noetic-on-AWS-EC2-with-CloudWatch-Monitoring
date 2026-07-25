# Project 2 Notes — Deploy ROS Noetic on AWS EC2 + CloudWatch Monitoring

## Requirements (from problem statement)
- Ubuntu 20.04 LTS
- General purpose t2.micro (used t3.micro instead — see note below)
- 30GB storage
- Prerequisite: AWS account

## 1. Finding the right AMI
- **Ubuntu 20.04 wasn't in the Quick Start AMI list** anymore — AWS rotates which LTS versions show there by default. Had to go to **Browse more AMIs** and search manually.

- **Correct AMI found under Community AMIs**, searching `ubuntu-focal-20.04`, then narrowing to:
  `ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-20250603`
  - Owner: `099720109477` (Canonical's official AWS account — verify this to avoid untrusted community uploads)
  - Architecture: x86_64
  - Free tier eligible, no Marketplace subscription cost
  - Plain description: "Canonical, Ubuntu, 20.04, amd64 focal image" — no EKS/Pro/Minimal prefix

## 2. Instance type: t2.micro not available
- **t2.micro showed as not eligible** under the current AWS plan (see note on AWS's billing model below) — had to switch to **t3.micro** instead.
- t3.micro has the same 1 vCPU / 1GB RAM as t2.micro, just a newer processor generation — functionally fine as a substitute for "general purpose micro."
- **Known risk with only 1GB RAM:** `ros-noetic-desktop-full` is a large install (simulators + perception packages) and could hit out-of-memory issues on such a small instance. Didn't need a swap file this time, but kept it in mind as a fallback if the install had stalled or failed.

## 3. Launch configuration
- Instance type: t3.micro
- Storage: changed from default to **30GB** (under "Configure storage" in the launch wizard)
- **IAM role attached at launch this time** (not after). Role included `AmazonSSMManagedInstanceCore` (for Session Manager) — later needed `CloudWatchAgentServerPolicy` too for the CloudWatch agent to actually deliver data.
- Security group: only needed SSH (port 22, source Anywhere) — no HTTP rule needed this time since ROS doesn't serve a public webpage . (Session Manager is used for actual connection, so this SSH rule is more of a backup than a necessity.)
- Auto-assign public IP: **enabled** — needed even though connecting via Session Manager, because the ROS install steps download packages from the internet (packages.ros.org, GitHub for the GPG key), which requires the instance to reach the internet through the VPC's internet gateway.

## 4. Verify OS version
```
lsb_release -a
```
Confirmed: Ubuntu 20.04.6 LTS, codename `focal` — matches what the ROS install commands expect.

## 5. Install ROS Noetic

**Add ROS package repository:**
```
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
```
- This command produces **no visible output on success** — that's normal, not a sign of failure.
- Ran it twice by accident — harmless, just overwrote the file with identical content both times.
- Verified it worked by checking: `cat /etc/apt/sources.list.d/ros-latest.list`

**Install curl and add the GPG key:**
```
sudo apt install curl -y
curl -s https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | sudo apt-key add -
```
- `apt-key` is a deprecated method on newer systems but still functions correctly on 20.04 — a deprecation warning here is expected, not an error.

**Update and install:**
```
sudo apt update
sudo apt install ros-noetic-desktop-full -y
```
- This is a large install — took a significant amount of time (20-30 min range), completely normal given the package size.

## 6. Environment setup — shell mismatch confusion
- **Major point of confusion:** ran `source /opt/ros/noetic/setup.bash` and got `sh: 9: source: not found`.
  - Cause: the terminal session was running under `sh` (specifically Ubuntu's default `dash`), which doesn't support the `source` command — that's a `bash`-specific builtin.
  - **Wrong fix tried first:** using `. /opt/ros/noetic/setup.bash` (the POSIX dot-equivalent) — this still failed with cascading errors (`Bad substitution`, `builtin: not found`, `Can't open /setup.sh`), because the *script itself* uses bash-only syntax internally, so running it under any non-bash shell breaks regardless of which "source" method is used.
  - **Actual fix:** type `bash` first to switch into an actual bash shell, THEN run `source /opt/ros/noetic/setup.bash` — this worked cleanly.
- **Made it permanent** (so future terminal sessions don't need manual sourcing):
```
echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc
```
Ran this from inside the bash shell (not the default sh) to keep things consistent.

## 7. Verify the ROS install
```
roscore
```
- Success output included: `started roslaunch server...`, `ROS_MASTER_URI=http://ip-10-0-165:11311/`, `process[master]: started with pid [...]`, `started core service [/rosout]` — no errors anywhere.
- This is the key screenshot/proof for the assignment.
- Pressed `Ctrl+C` to stop it once confirmed working.

## 8. CloudWatch agent setup

**Why:** EC2 only reports basic metrics (CPU, network) automatically. Memory and disk usage require installing a separate CloudWatch agent on the instance.

**Install:**
```
sudo apt install unzip -y
curl -O https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb
```
- **Hit the exact "Permission denied" issue  — the `curl -O` command (no `sudo`) tried to write the downloaded file into a folder the current user didn't own.
  - **Fix:** `cd ~` first, then retry the curl command, since home directory is always writable by the logged-in user.
  - **Key lesson on when sudo is/isn't needed:** `sudo apt install` commands install system-wide and don't care about the current folder (root can write anywhere). Plain commands like `curl -O` write to the current folder as your regular user — if that folder isn't yours, it fails. The fix for the second kind is to `cd` somewhere you own, not to reflexively add `sudo` to everything (bad practice — removes the safety boundary that limits mistakes to your own files instead of the whole system).

**Configure and start:**
```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c default -s
```
- Ran cleanly: "Configuration validation succeeded", created a systemd service symlink.
- Requires `CloudWatchAgentServerPolicy` attached to the instance's IAM role — without it, the agent can run locally but fail to actually deliver data to CloudWatch.

**Verify agent is running:**
```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status -m ec2
```
Returned `"status": "running"`, `"configstatus": "configured"` — confirmed working locally.

**Check agent logs (used when troubleshooting "no data showing"):**
```
sudo tail -n 50 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```
Showed the agent's actual receiver config: `telegraf_mem`, `telegraf_disk` → exporting via `awscloudwatch` — confirmed it's specifically monitoring memory and disk (the metrics EC2 doesn't track by default), not CPU/network which are already covered separately.

## 9. Finding the data in the CloudWatch console 

- To see agent-specific data (memory, disk), needed to go to the **separate CloudWatch console** (not EC2), not the "Overview" landing page either.

  1. Left sidebar → **Metrics** (expandable section)
  2. Click **"Classic metrics"** underneath it
  3. Look for the **CWAgent** namespace tile
  4. Drill down through dimension folders to find the specific instance, then select a metric like `disk_used_percent` or `mem_used_percent`
- Found `disk_used_percent` plotting real data (~100% for a loop3/squashfs snap-related mount — normal, since snap package mounts are small read-only filesystems that read near-full by design; the `nvme`/`ext4`/`/` row is the more meaningful root filesystem metric to check instead).
- This confirmed the whole CloudWatch setup was working correctly the entire time.

## General lessons
- Attach IAM roles at launch, not after — avoided the Project 1 stop/start issue entirely this time.
- `sudo` is for system-wide changes only, not a blanket fix for every permission error — check *where* you are (`pwd`, `cd ~`) before reaching for `sudo` on a file-write command.
- A command producing no output isn't necessarily broken — some succeed silently (e.g. the `echo > file` repo-adding command).
- When something seems broken in the AWS console, double-check you're looking at the right *page/section* before assuming the underlying setup failed — default metrics vs. agent-specific metrics live in genuinely different places.
- Shell type matters: `sh`/`dash` vs `bash` are not interchangeable — a script written for bash can fail in confusing, cascading ways if sourced under the wrong shell, even using the "equivalent" dot syntax.
- AWS periodically rotates which AMI versions appear in the Quick Start list — an older LTS still being required by a project doesn't mean it's been removed entirely, just relocated to "Browse more AMIs" / Community AMIs.

