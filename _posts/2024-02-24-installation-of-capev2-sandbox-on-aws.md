---
title: Deploying CAPEv2 on AWS - A Comprehensive Guide
date: 2024-02-25 16:39:00 +0300
categories: [Malware Analysis, CAPEv2, Cuckoo]
tags: [malware analysis, capev2, cuckoo]
image: /assets/img/posts/installation-of-capev2-sandbox-on-aws/cover.png
---
## Introduction
CAPEv2, an open-source automated malware analysis system, stands at the forefront of innovative solutions for dissecting and comprehensively understanding malware behavior. <br> 
Developed as an evolution of Cuckoo Sandbox, CAPEv2 has established itself as a powerful tool for automatically executing and analyzing files within an isolated  environments.


### Key Features and Capabilities:
- **Comprehensive Analysis Results:**
  - CAPEv2 automatically runs and analyzes files, providing detailed insights into malware behavior.
  - Analysis results encompass traces of win32 API calls, created and deleted files, memory dumps, network traffic traces, and screenshots of the Windows desktop during malware execution.
- **Rich History and Development:**
  - Originating from the Cuckoo Sandbox project in 2010, CAPEv2 is a product of continuous development and enhancements.
  - Notable milestones include releases, contributions, and the establishment of the Cuckoo Foundation in March 2014.
- **Versatility in Use Cases:**
  - Designed to analyze various file types, including Windows executables, DLL files, PDFs, Microsoft Office documents, URLs, HTML files, PHP scripts, and more.
  - Modular design enables integration into larger frameworks, showcasing its adaptability to diverse analysis needs.
- **Modular Architecture:**
  - CAPE Sandbox comprises central management software and isolated virtual machines (Guest machines) for analysis.
  - The Host machine manages the entire analysis process, while the Guest machines provide secure environments for executing and analyzing malware samples.
- **Limitless Customization:**
  - CAPE's modularity and scripting capabilities empower users to customize and extend its functionality, making it a versatile tool for varied use cases.

## Why deploy CAPE on AWS?
As of February 25, 2024, there has been a noticeable scarcity of open-source malware analysis automated sandbox projects designed to seamlessly integrate with AWS.<br> 
Traditionally, sandbox projects like CAPE and Cuckoo, on which CAPE is based, were primarily installed on bare metal servers.<br> However, this approach came with limitations and dependencies on specific hardware.<br> The breakthrough in leveraging AWS for malware analysis lies in its on-demand infrastructure and pay-as-you-go model, eliminating the need for specialized hardware and allowing users to pay only for the resources they consume.
One notable milestone in this paradigm shift was the initiative undertaken by the <a href="https://research.checkpoint.com/2019/cuckoo-system-on-aws/" target="_blank">Checkpoint Research team in 2019. </a> <br> 
They introduced a project that utilized AWS for deploying the Cuckoo system, providing a groundbreaking alternative to bare metal installations. <br> Unfortunately, the Checkpoint <a href="https://github.com/CheckPointSW/Cuckoo-AWS/" target="_blank">project. </a>  appears to be unmaintained.

In 2022, the CAPE community seized the opportunity to enhance their project's capabilities by integrating the AWS deployment option. The CAPE project, building upon the foundation laid by Checkpoint's initiative, introduced modifications that enabled the utilization of Amazon EC2 instances as guest machines. This offered an alternative of the traditional use of KVM or other virtualization methods, as AWS does not support nested virtualization unless a costly bare metal server is employed.

### How it works?
CAPE operates with a host responsible for managing analysis machinery where malware executes and an Amazon Machine Image (AMI) that requires building. Upon malware submission, the CAPE host initiates an AMI instance, conducts analysis, and retrieves results through an HTTP agent installed on the AMI. This process enables comprehensive behavioral analysis reports on the malware.

> Acknowledgment: It is worth mentioning that although CAPEv2 is highly maintained, the AWS module, being a community module, is not maintained by the core developers.<br> During my installation, I encountered errors that were resolved thanks to plutusrt. He provided assistance with the live error, identified the bug, and addressed it. More details can be found <a href="https://github.com/kevoreilly/CAPEv2/pull/1980/" target="_blank">here. </a>
{: .prompt-tip }

> Warning: AWS Policy Guidelines
Sandbox analysis must occur in a secure AWS account and VPC.<br>
Inbound traffic is restricted to customer-owned IP addresses.<br>
Outbound internet access from within AWS, including via proxies, is prohibited.<br>
Simulation services (e.g., INetSim) must reside in the same VPC as the malware.<br><br>
Consider alternate solutions if internet access is required during testing, as AWS may not be suitable due to these limitations.
{: .prompt-danger }

## Overview of the Deployment Process:

**Preparing the CAPEv2 Guest Machine VMDK:**
- Set up a local Windows 10 VM for CAPE analysis, configure it for optimal performance, and export the VMDK file encapsulating the VM state.

**Configuring S3 Bucket, VM import roles, AWS-CLI, and converting the VMDK to AMI:**
- Upload the VMDK to an AWS S3 bucket, streamline the process with AWS-CLI, and convert the VMDK to an AMI.
- Create users and roles for secure VM import processes.

**Setting and Configuring VPC:**
- Establish a dedicated VPC for malware analysis, implementing security groups for isolation.
- Fine-tune network settings to meet specific malware analysis requirements.

**Installing and Configuring CAPEv2 Host:**
- Install the CAPEv2 server on an Ubuntu 22.04 LTS instance within the AWS environment.
- Configure the CAPE conf file to ensure optimal performance and compatibility.

> Prerequisites:<br>
Basic familiarity with AWS is required.<br>
A dedicated AWS account with an administrative IAM user<br>
It is highly recommended to read the <a href="https://capev2.readthedocs.io/en/latest/" target="_blank">CAPEv2 documentation </a> to understand the concepts.<br> It will help troubleshoot any issues you might encounter.
{: .prompt-info }

## Step-by-step Installation Guide:

### Preparing the Guest machine VMDK


Step-by-step Installation Guide
Preparing the Guest machine VMDK

1\. **Download and install the Win10_21H2_English_x64 ISO**<br>
   CAPEv2 supports Windows 7, and Win10_21H2_English_x64.<br>As the Win10_21H2 version is considered End-of-Life (EOL), it is no longer available for download from Microsoft. <br>To obtain the ISO, I recommend using the Wayback Machine on archive.org. Here is the download link:
   
   `https://ia902708.us.archive.org/11/items/win-10-21-h-2-english-x-64_20220529/Win10_21H2_English_x64.iso`

2\. **Verify the integrity of the ISO using:**

   Windows:
```text
Get-FileHash .\win-10-21-h-2-english-x-64_20220529\Win10_21H2_English_x64.iso
```
   Unix:
```bash
sha256sum  win-10-21-h-2-english-x-64_20220529/Win10_21H2_English_x64.iso on 
```

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image40.png){: width="700" height="400" }

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image46.png){: width="700" height="400" }


3\. **Using that ISO create a new Virtual Machine on Vmware workstation pro:**

  > To avoid automatic updates, leave the machine in host-only mode during the installation until you disable Windows updates.
  {: .prompt-warning }

4\. **Disable Windows updates:**

```text
C:\Windows\system32>services.msc
```

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image10.png){: width="700" height="400" }

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image9.png){: width="700" height="400" }

5\. **Install Python 3.10.6 (32-bit).:**
```text
wget https://www.python.org/ftp/python/3.10.6/python-3.10.6.exe -o python-3.10.6.exe
```
![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image11.png){: width="700" height="400" }

6\. **Verify the installation:**
```text
C:\Windows\system32>python --version
Python 3.10.6
```
7\. **Install Pip and Pillow:**
```text
python -m pip install --upgrade pip
python -m pip install Pillow==9.5.0
```
8\. **Disable Windows Firewall.**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image14.png){: width="700" height="400" }

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image19.png){: width="700" height="400" }
_Credit: https://capev2.readthedocs.io/en/latest/installation/guest/network.html_

9\.**Enable RDP with no password:**
```
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
```
```
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 0
```
```
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' -Name 'LimitBlankPasswordUse' -Value 0
```
```
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' -Name 'LimitBlankPasswordUse' -Value 0
```

10\.**Disable Noisy Network Services:**

**Tredo**
```
netsh interface teredo set state disabled
```
**Link Local Multicast Name Resolution (LLMNR)**

Open the Group Policy editor by typing gpedit.msc into the Start Menu search box, and press Enter. Then navigate to Computer Configuration> Administrative Templates> Network> DNS Client, and open Turn off Multicast Name Resolution.
Set the policy to enabled.

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image41.png){: width="700" height="400" }

**Network Connectivity Status Indicator, Error Reporting, etc**

Windows has many diagnostic tools such as Network Connectivity Status Indicator and Error Reporting, that reach out to Microsoft servers over the Internet. Fortunately, these can all be disabled with one Group Policy change.

Open the Group Policy editor by typing gpedit.msc into the Start Menu search box, and press Enter. Then navigate to Computer Configuration> Administrative Templates> System> Internet Communication Management, and open Restrict Internet Communication.

Set the policy to enabled.

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image24.png){: width="700" height="400" }

11\.**Disable Microsoft Defender:**
```
Set-MpPreference -DisableRealtimeMonitoring $true -DisableScriptScanning $true -DisableBehaviorMonitoring $true -DisableIOAVProtection $true -DisableIntrusionPreventionSystem $true
```
12\.**Disable App and Browser Control:**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image6.png){: width="700" height="400" }

13\.**Download and execute the "disable-defender.ps1" PowerShell script.**
` https://github.com/jeremybeaume/tools/blob/master/disable-defender.ps1`

```
Set-ExecutionPolicy Unrestricted
```
```
.\disable-defender.ps1
```

14\.**Reboot:**
```
Restart-Computer -Force
```
15\.**Download and install the CAPEv2 agent on the guest machine, following the CAPEv2 documentation:**

**Download the agent:**<br>
Choose your own Python name for the agent; the name is irrelevant as long as you avoid any mention of CAPE or anything blatant to evade anti-VM detection algorithms.
```
wget https://raw.githubusercontent.com/kevoreilly/CAPEv2/master/agent/agent.py -o randomprogramname.pyw
```
**Install the agent using the CAPEv2 <a href="https://capev2.readthedocs.io/en/latest/installation/guest/agent.html" target="_blank">guide </a>.**

> Optional:<br>
Find and install Microsoft Office 2010 Pro Plus with SP2 September 2020 (x86).
{: .prompt-info }

The guest is now ready, let's export and upload it to AWS.

### Configuring S3 Bucket, VM import roles, AWS-CLI, and converting the VMDK to AMI:

1\.**Importing the VMDK to AWS**
On vmware export by using the export to OVF option this will create and OVF and A VMDK files on a desired locations

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image38.png){: width="700" height="400" }

2\.**Create a bucket or use an existing one:**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image39.png){: width="700" height="400" }

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image21.png){: width="700" height="400" }

Leave all default configurations and save the bucket.

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image25.png){: width="700" height="400" }


3\.**Create access key and configure the AWS-CLI:**

**Go to IAM > Users > choose your own user** <br>

Create access key

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image8.png){: width="700" height="400" }

Choose command line interface

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image13.png){: width="700" height="400" }

Create access key

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image43.png){: width="700" height="400" }

4\.**Follow the AWS <a href="https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html" target="_blank">guide </a>to install the AWS-CLI (platform-dependent).**



5\.**Configure the AWS CLI.**
```
aws configure
```
You will be prompted to enter the access key ID, the secret access key, default region, and default output format (by default JSON).

6\.**Upload the VMDK file to S3.**
```
aws s3 cp .\Windows-Guest-disk1.vmdk s3://machinery-import/
```
This can take a while.<br> Meanwhile, we will configure our role for creating the AMI and the "containers.json" file that is required to create the bucket.

7\.**Creating the role:**

Refer to the detailed AWS guide for creating this role.

4\.**Follow the AWS <a href="https://docs.aws.amazon.com/vm-import/latest/userguide/required-permissions.html#vmimport-role" target="_blank">guide </a>to create the role.**

8\.**Create the "containers.json" file.**
```json
[
  {
    "Description": "CAPEv2-Guest-AMI",
    "Format": "vmdk",
    "Url": "s3://YourBucketName/YourVmdkName.vmdk"
  }
]

```
9\.**Convert the VMDK from S3 to an AMI once the S3 upload is complete.**
```
aws ec2 import-image --description "CAPEv2 Guest VM" --disk-containers "file://C:\Users\DevSecOps\containers.json"
```
To check the status of the machine, use the ImportTaskId value from the JSON.
```
aws ec2 describe-import-image-tasks --import-task-ids "TASK ID"
```

### Setting and Configuring VPC:

1\.**Using the AWS console, search for VPC:**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image1.png){: width="700" height="400" }

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image37.png){: width="700" height="400" }

2\.**Click on "Create VPC."**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image5.png){: width="700" height="400" }


> Use the same configuration as shown in the pictures below:<br>
{: .prompt-info }

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image15.png){: width="700" height="400" }

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image26.png){: width="700" height="400" }


3\.**Set up the security groups.**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image2.png){: width="700" height="400" }

**Click on "Create Security Group."**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image35.png){: width="700" height="400" }

**Create the first security group** CAPEv2Host <br> The rules should allow inbound SSH and web access on port 8000 from your own IP.<br> Leave the outbound rules as default.
![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image4.png){: width="700" height="400" }

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image27.png){: width="700" height="400" }


### Installing and Configuring CAPEv2 Host:

1\.**Create a Ubuntu Server 22.04 using EC2 with the following specifications:**
- AMI: ami-0905a3c97561e0b69 (64-bit (x86)) / ami-0a1b36900d715a3ad (64-bit (Arm))
- Virtualization: HVM
- Instance Type: T2.xlarge
- Storage: 100 GB gp2
- VPC: Our Created VPC
- Security Group:CAPEv2Host
- SSH key: Required

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image31.png){: width="700" height="400" }

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image33.png){: width="700" height="400" }

2\.**Connect to the instance using SSH**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image22.png){: width="700" height="400" }

Click on that instance

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image36.png){: width="700" height="400" }

Choose "Connect"

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image16.png){: width="700" height="400" }

3\.**Connecting to the instance and running the following commands:**

**Elevate to root:**
```bash
sudo -i
```
**Update and Upgrade packages**
```bash
apt-get update && apt-get upgrade -y
```
**Exit root**
```bash
exit 
```
**Download the CAPEv2 bash script installer**
```bash
wget https://raw.githubusercontent.com/kevoreilly/CAPEv2/master/installer/cape2.sh
```
**Make the bash script executable**
```bash
chmod +xr cape2.sh
```
**Execute the script as root**
```bash
sudo ./cape2.sh all cape | tee cape.log
```
**Once The installation has completed move to the CAPEv2 folder**
```bash
cd /opt/CAPEv2
```
**Install dependencies using poetry(as a non-root user)**
```bash
poetry install
```

**Command to fix a DB error**
```bash
sudo -u postgres -H sh -c "psql -d \"cape\" -c \"ALTER DATABASE cape OWNER TO cape;\""
```
**Reboot**
```bash
sudo reboot
```
**Reconnect to the instance and navigate to the CAPEv2 directory.**
```bash
cd /opt/CAPEv2
```
**Verify that the dependencies were installed, and the virtual environment was created**
```bash
poetry show
```
```bash
poetry env list
```
**Update the Dependencies**
```bash
sudo poetry update
```
**Install additional dependencies**
```bash
sudo -u cape poetry run pip3 install https://github.com/CAPESandbox/peepdf/archive/20eda78d7d77fc5b3b652ffc2d8a5b0af796e3dd.zip#egg=peepdf==0.4.2
```
```bash
sudo -u cape poetry run pip install -U git+https://github.com/DissectMalware/batch_deobfuscator
```
```bash
sudo -u cape poetry run pip install -U git+https://github.com/CAPESandbox/httpreplay
```
**Get your internal server IP**
```bash
ip a
```
**Configure The Cuckoo.conf file so it will look like this, replace `<REPLACE WITH YOUR IP>` with your ip**
```bash
sudo nano conf/cuckoo.conf
```
```text
[cuckoo]

# Which category of tasks do you want to analyze?
categories = static, pcap, url, file

# If turned on, Cuckoo will delete the original file after its analysis
# has been completed.
delete_original = off

# Archives are not deleted by default, as it extracts and "original file" become extracted file
delete_archive = on

# If turned on, Cuckoo will delete the copy of the original file in the
# local binaries repository after the analysis has finished. (On *nix this
# will also invalidate the file called "binary" in each analysis directory,
# as this is a symlink.)
delete_bin_copy = off

# Specify the name of the machinery module to use, this module will
# define the interaction between Cuckoo and your virtualization software
# of choice.
machinery = aws

# Enable screenshots of analysis machines while running.
machinery_screenshots = off

# Specify if a scaling bounded semaphore should be used by the scheduler for tasking the VMs.
# This is only applicable to auto-scaling machineries such as Azure and AWS.
# There is a specific configuration key in each machinery that is used to initialize the semaphore.
# For Azure, this configuration key is "total_machines_limit"
# For AWS, this configuration key is "dynamic_machines_limit"
scaling_semaphore = off
# A configurable wait time between updating the limit value of the scaling bounded semaphore
scaling_semaphore_update_timer = 10
# Allow more than one task scheduled to be assigned at once for better scaling
# A switch to allow batch task assignment, a method that can more efficiently assign tasks to available machines
batch_scheduling = off
# The maximum number of tasks assigned to machines per batch, optimal value dependent on deployment
max_batch_count = 20

# Enable creation of memory dump of the analysis machine before shutting
# down. Even if turned off, this functionality can also be enabled at
# submission. Currently available for: VirtualBox and libvirt modules (KVM).
memory_dump = off

# When the timeout of an analysis is hit, the VM is just killed by default.
# For some long-running setups it might be interesting to terminate the
# moinitored processes before killing the VM so that connections are closed.
terminate_processes = off

# Enable automatically re-schedule of "broken" tasks each startup.
# Each task found in status "processing" is re-queued for analysis.
reschedule = off

# Fail "unserviceable" tasks as they are queued.
# Any task found that will never be analyzed based on the available analysis machines
# will have its status set to "failed".
fail_unserviceable = on

# Limit the amount of analysis jobs a Cuckoo process goes through.
# This can be used together with a watchdog to mitigate risk of memory leaks.
max_analysis_count = 0

# Limit the number of concurrently executing analysis machines.
# This may be useful on systems with limited resources.
# Set to 0 to disable any limits.
max_machines_count = 10

# Limit the amount of VMs that are allowed to start in parallel. Generally
# speaking starting the VMs is one of the more CPU intensive parts of the
# actual analysis. This option tries to avoid maxing out the CPU completely.
# This configuration option is only relevant for machineries that have a set
# amount of VMs and are restricted by CPU usage.
# If you are using an auto-scaling machinery such as Azure or AWS,
# set this value to 0.
max_vmstartup_count = 5

# Minimum amount of free space (in MB) available before starting a new task.
# This tries to avoid failing an analysis because the reports can't be written
# due out-of-diskspace errors. Setting this value to 0 disables the check.
# (Note: this feature is currently not supported under Windows.)
freespace = 0
# Process tasks, but not reach out of memory
freespace_processing = 15000

# Temporary directory containing the files uploaded through Cuckoo interfaces
# (web.py, api.py, Django web interface).
tmppath = /tmp

# Delta in days from current time to set the guest clocks to for file analyses
# A negative value sets the clock back, a positive value sets it forward.
# The default of 0 disables this option
# Note that this can still be overridden by the per-analysis clock setting
# and it is not performed by default for URL analysis as it will generally
# result in SSL errors
daydelta = 0

# Path to the unix socket for running root commands.
rooter = /tmp/cuckoo-rooter

# Enable if you want to see a DEBUG log periodically containing backlog of pending tasks, locked vs unlocked machines.
# NOTE: Enabling this feature adds 4 database calls every 10 seconds.
periodic_log = off

# Max filename length for submissions, before truncation. 196 is arbitrary.
max_len = 196

# If it is greater than this, call truncate the filename further for sanitizing purposes.
# Length truncated to is controlled by sanitize_to_len.
#
# This is to prevent long filenames such as files named by hash.
sanitize_len = 32
sanitize_to_len = 24

[resultserver]
# The Result Server is used to receive in real time the behavioral logs
# produced by the analyzer.
# Specify the IP address of the host. The analysis machines should be able
# to contact the host through such address, so make sure it's valid.
# NOTE: if you set resultserver IP to 0.0.0.0 you have to set the option
# `resultserver_ip` for all your virtual machines in machinery configuration.
ip = <REPLACE WITH YOUR IP>

# Specify a port number to bind the result server on.
port = 2042

# Force the port chosen above, don't try another one (we can select another
# port dynamically if we can not bind this one, but that is not an option
# in some setups)
force_port = yes

pool_size = 0

# Should the server write the legacy CSV format?
# (if you have any custom processing on those, switch this on)
store_csvs = off

# Maximum size of uploaded files from VM (screenshots, dropped files, log)
# The value is expressed in bytes, by default 100MB.
upload_max_size = 100000000

# To enable trimming of huge binaries go to -> web.conf -> general -> enable_trim
# Prevent upload of files that passes upload_max_size?
do_upload_max_size = no

[processing]
# Set the maximum size of analyses generated files to process. This is used
# to avoid the processing of big files which may take a lot of processing
# time. The value is expressed in bytes, by default 200MB.
analysis_size_limit = 200000000

# Enable or disable DNS lookups.
resolve_dns = on

# Enable or disable reverse DNS lookups
# This information currently is not displayed in the web interface
reverse_dns = off

# Enable PCAP sorting, needed for the connection content view in the web interface.
sort_pcap = on

[database]
# Specify the database connection string.
# Examples, see documentation for more:
# sqlite:///foo.db
# postgresql://foo:bar@localhost:5432/mydatabase
# mysql://foo:bar@localhost/mydatabase
# If empty, default is a SQLite in db/cuckoo.db.
# SQLite doens't support database upgrades!
# For production we strongly suggest go with PostgreSQL
connection = postgresql://cape:SuperPuperSecret@localhost:5432/cape
# If you use PostgreSQL: SSL mode
# https://www.postgresql.org/docs/current/libpq-ssl.html#LIBPQ-SSL-SSLMODE-STATEMENTS
psql_ssl_mode = disable

# Database connection timeout in seconds.
# If empty, default is set to 60 seconds.
timeout =

# Log all SQL statements issued to the database.
log_statements = off

[timeouts]
# Set the default analysis timeout expressed in seconds. This value will be
# used to define after how many seconds the analysis will terminate unless
# otherwise specified at submission.
default = 200

# Set the critical timeout expressed in (relative!) seconds. It will be added
# to the default timeout above and after this timeout is hit
# Cuckoo will consider the analysis failed and it will shutdown the machine
# no matter what. When this happens the analysis results will most likely
# be lost.
critical = 60

# Maximum time to wait for virtual machine status change. For example when
# shutting down a vm. Default is 300 seconds.
vm_state = 300

[tmpfs]
# only if you using volatility to speedup IO
# mkdir -p /mnt/tmpfs
# mount -t tmpfs -o size=50g ramfs /mnt/tmpfs
# chown cape:cape /mnt/tmpfs
#
# vim /etc/fstab
# tmpfs       /mnt/tmpfs tmpfs   nodev,nosuid,noexec,nodiratime,size=50g   0 0
#
# Add crontab with
# @reboot chown cape:cape /mnt/tmpfs -R
enabled = off
path = /mnt/tmpfs/
# in mb
freespace = 2000

[cleaner]
# Invoke cleanup if <= of free space detected. see/set freespace/freespace_processing
enabled = no
# set any value to 0 to disable it. In days
binaries_days = 5
tmp_days = 5
# Remove analysis folder
analysis_days = 5
# Delete mongo data
mongo = no
```
3\. **Create a IAM user for CAPE**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image3.png){: width="700" height="400" }

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image12.png){: width="700" height="400" }

4\.**4. Choose Attach policies directly and Create a policy**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image45.png){: width="700" height="400" }

- Choose Json and paste the following policy
- Replace the values of the VPC with the VPC you created and the value of the AMI with the AMI ID you created.

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": [
				"ec2:DescribeInstances",
				"ec2:ModifyInstanceAttribute",
				"ec2:RunInstances",
				"ec2:TerminateInstances",
				"ec2:CreateTags"
			],
			"Resource": "*",
			"Condition": {
				"StringEquals": {
					"ec2:Region": "eu-west-1"
				}
			}
		},
		{
			"Effect": "Allow",
			"Action": "ec2:RunInstances",
			"Resource": "*",
			"Condition": {
				"StringEquals": {
					"ec2:Region": "eu-west-1",
					"ec2:Vpc": "<Replace with your VPC>",
					"ec2:ImageId": "<Replace with your AMI>>"
				}
			}
		}
	]
}
```

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image28.png){: width="700" height="400" }

**Hit Next:**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image7.png){: width="700" height="400" }

**Choose policy name**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image20.png){: width="700" height="400" }

**Choose policy name**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image18.png){: width="700" height="400" }

**Refresh the previous page and attach the policy**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image30.png){: width="700" height="400" }

**Create user**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image29.png){: width="700" height="400" }

5\.**Navigate to the newly created user and click on "Create access key."**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image32.png){: width="700" height="400" }

**Choose CLI  and hit next**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image42.png){: width="700" height="400" }

**Copy the access and secret key, write it down, but don't save it.**


6\.**Create a security group for the CAPE guest**
The group should never allow any access to the internet. One disadvantage in AWS is that according to AWS policy when configuring a sandbox, the machines are not allowed to have internet access, even through a proxy.

The outbound rules should allow access only to the CAPEv2Host security group. The inbound rules should allow all traffic from the CAPEv2Host security group and RDP access from your own IP (for debugging purposes).

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image17.png){: width="700" height="400" }

7\.**Get back to the CAPEhost security group and allow all TCP traffic from the newly created guest security group.**

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/image23.png){: width="700" height="400" }

8\.**Reconnect to the Cape Host Ubuntu server**

- **install boto3**

```bash
cd /opt/CAPEv2/
```
```bash
sudo -u cape poetry run pip3 install boto3
```

- **Configure the AWS machinery conf file**

```bash
sudo nano conf/aws.conf
```
- **Change the following:**
- Add region name  > `region_name =`
- Add availability zone >  `availability_zone =`

> Warning: When it comes to writing access keys to the EC2 instance, it's important to note that this is considered a bad practice. However, currently, there is no other solution. Keep in mind to monitor the CAPE user's actions. This is also the reason the CAPE user is allowed to perform actions only on the specific VPC and doesn’t have full EC2 access.
{: .prompt-danger }

- Add AWS access key id > `aws_access_key_id =`
- Add AWS secret access key >  `aws_secret_access_key = `
- Change dynamic_machines_limit > `dynamic_machines_limit = 1`
- change image_id  to your real ami ID > `image_id`
- Change the instance_type to t3.large (can be any other size as well, but it will impact performance). > `instance_type`
- Change subnet id to the subnet id of your CAPE host > `subnet_id`
- Change security group to the guest security group > `security_groups`
- Replace `#interface`  with `interface = eth0`
- Delete everything after `arch = x64`

**Restart the cape service**
```bash
sudo systemctl restart cape.service
```

```bash
journalctl -u cape.service -f
```

If everything was installed and configured successfully, you should be able to see this:
![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/photo_5_2024-02-25_20-10-48.jpg){: width="700" height="400" }


### Demo Time!

For this demo, I have created a basic Msfvenom executable reverse shell and uploaded it to CAPE for analysis.

- generating the executble

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/photo_4_2024-02-25_20-10-48.jpg){: width="700" height="400" }


- Cape Service log after submission.

![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/photo_3_2024-02-25_20-10-48.jpg){: width="700" height="400" }
![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/photo_8_2024-02-25_20-10-48.jpg){: width="700" height="400" }

- Analysis results:
![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/photo_2_2024-02-25_20-10-48.jpg){: width="700" height="400" }
![Desktop View](/assets/img/posts/installation-of-capev2-sandbox-on-aws/photo_10_2024-02-25_20-10-48.jpg){: width="700" height="400" }






<bv>In conclusion, CAPEv2 emerges as a powerful tool in the realm of malware analysis, offering detailed insights and flexibility in deployment. Leveraging AWS enhances its scalability and cost-effectiveness, making it an ideal solution for organizations seeking efficient malware analysis capabilities. By following the deployment process outlined in this guide and referring to CAPEv2 documentation, users can effectively harness its capabilities to bolster their security posture and combat evolving cyber threats.

<h3>References:</h3>

https://research.checkpoint.com/2019/cuckoo-system-on-aws/<br>
https://github.com/CheckPointSW/Cuckoo-AWS<br>
https://capev2.readthedocs.io/en/latest/index.html<br>
https://docs.aws.amazon.com/vm-import/latest/userguide/required-permissions.html#vmimport-role<br>
https://d1.awsstatic.com/events/aws-reinforce-2022/NIS231_Malware-analysis-with-AWS.pdf<br>
