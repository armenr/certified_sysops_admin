# AWS Certified SysOps Administrator - Associate

Welp, here go my notes. Most of this is knowledge I already possess. These are notes to memorialize my formal study as it pertains to the certification curriculum. Refreshers never hurt.

## Monitoring, Metrics & Analysis

### Basics

* Host Level Metrics consist of:
  * CPU
  * Network
  * Disk
  * Status Check
* RAM Utilization is a custom metric - there are various approaches to writing scripts which will post this to the CloudWatch API for you.
* CloudWatch metrics are stored for 2 weeks, by default. The GetMetrics API will expose data older than 2 weeks. 3rd party monitoring tools/platforms utilize this API to provide historical data older than 2 weeks.
* CloudWatch data for terminated instances is available for up to two weeks after that instance was terminated. 

### Metric Granularity
* Default EC2 monitoring intervals are @ 5 minutes. "Detailed monitoring" will provide 1 minute polling intervals, but is subject to charges.
* The granularity of specific metrics depend on their appendant services.
  * Most services support 1 minute intervals
  * Some services poll at 3 or 5 minute defaults
* **Custom metrics** (like RAM utilization) will produce statistical results at a frequency of *up to* one minute.
  * **AWS documentation:** "Amazon CloudWatch metrics provide statistical results at a frequency up to one minute. This includes custom metrics. You can send custom metrics to Amazon CloudWatch as frequently as you like, but statistics will only be available at one minute granularity. You can also request statistics at a lower frequency, for example five minutes, one hour, or one day."

### CloudWatch Alarms
* You can set up custom alarms that will do things like send notifications when you hit billing thresholds.
* You can set actions that can be taken in the event of a threshold breach/alarm state.

*****

### EC2 Status Checks
* **System Status Checks:** Checks the underlying physical host (baremetal where your EC2 VM resides)
  * Loss of network connectivity
  * Loss of system power
  * Software issues on the physical host
  * Hardware issues on the physical host
  * **Typical path to resolution:** Explicit stop/start the EC2 instance. The VM will restart on a different physical host under the AWS covers. 
* **Instance Status Checks:** Checks the VM/EC2 instance itself
  * Failed system status checks will trigger implicit instance status check failures
  * Misconfigured networking or startup configuration
  * Exhasuted memory
  * Corrupted file system
  * Incompatible kernel
  * **Typical path to resolution:** Bounce the EC2 with a restart, or make modifications to the OS.
  
*****

### CloudWatch Roles
Role-based access & controls for resources across AWS services is standard practice. In order to send custom metrics like RAM usage from an EC2 to CloudWatch, a correspondent role should be created in IAM to permit the EC2 resources the requisite privileges for sending metrics to CloudWatch.

* Create IAM Role for CloudWatch
  * Role Name: CWEC2Role
  * Role Type: AWS Service Roles --> Amazon EC2
  * Attach Policy: CloudWatchFullAccess

**What does this do?** --> Creates a role that, when attached to an EC2, grants EC2's complete access to CloudWatch. 

********

### Monitoring EC2 With Custom Metrics (example)
1. Instantiate new EC2 (micro is fine)
2. "Configure Instance Details" --> IAM Role: CWEC2Role
3. SSH your new host & `sudo su - ` (for ease - not a best practice!) 
4. Update system & install prerequisite packages
  * ` apt-get update && apt-get upgrade -y && apt-get install -y unzip libwww-perl libdatetime-perl`
  *  `sudo yum install perl-Switch perl-DateTime perl-Sys-Syslog perl-LWP-Protocol-https perl-Digest-SHA -y && yum install zip unzip -y `
5. Script download & installation - personally, I like putting optional package installs in '/opt/'

		
		mkdir -p /opt/aws/cloudwatch
		curl http://aws-cloudwatch.s3.amazonaws.com/downloads/CloudWatchMonitoringScripts-1.2.1.zip -O
		unzip CloudWatchMonitoringScripts-1.2.1.zip
		rm CloudWatchMonitoringScripts-1.2.1.zip
		cd aws-scripts-mon
		
6. Verify that the magical scripts work. Yay PERL! PERL is still relevant!
		
		./mon-put-instance-data.pl --mem-util --verify --verbose
	*With IAM roles utilized, you don't need to add access keys & such in the configs. You cannot retroactively append IAM roles to EC2 Instances. You will have to destroy & recreate an instance to take advantage of that.*
	
7. To post memory metrics to CloudWatch, try this.

		./mon-put-instance-data.pl --mem-util --mem-used --mem-avail

8. To run this every minute, set up a CRON job. It's easy. CRON is still relevant! 
		
		crontab -l > new_cron
		echo "*/1 * * * * ~/aws-scripts-mon/mon-put-instance-data.pl --mem-util --disk-space-util --disk-path=/ --from-cron" >> new_cron
		crontab new_cron
		rm new_cron
		
9. Hop back over to CloudWatch in the AWS Console and seek out your new metrics for that instance. Pretty graphs will take a short while (recurrent polling intervals) to show up.

******

### Monitoring EBS Volumes
Types of storage:

  * Magnetic (yucky)
  * General Purpose SSD
  	* 3 IOPS/GB on your SSD
  	* Up to 10K IOPS - go to Provisioned IOPS past that
  	* When your volume does need more than the baseline IOPS, you use I/O credits in the credit balance, up to a mximum of 3K.
  	* Each volume recieves an initial I/O credit balance of 5,400,000 credits
  	* This sustains a maximum burst of 3K IOPS for 30 minutes
  	* When staying below your provisioned I/O level (bursting), the volume earns credits
  	* Bonus: How do you find out the number of credits you've earned? (Max is 5,400,000 credits)
  * Provisioned IOPS SSD
  
  For further reading & use cases, go here: <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html>
  
  * Back-end storage blocks for EBS volumes are allocated immediately to any EBS volume, however those blocks are "cold." Volumes that need to perform the moment they are introducted into production require pre-warming first. There is an implicit 5-50% performance decrease on cold volumes, but in most applications this hit is ammortized reasonably over the course of that volume's use cycle. 
    * New volumes: blocks need to be wiped clean
    
     ``sudo dd if=/dev/xvdf of=/dev/null bs=1M ``
    		
    * Volumes from Restored snapshots: You have to read all the blocks on the volume
      * To pre-warm only existing blocks of written data on a restored snapshot volume:
    		`` sudo dd if=/dev/xvdX of=/dev/null bs=1`` 

    	* To pre-warm existing blocks **and** empty blocks on a restored volume:
    		`` sudo dd if=/dev/xvdX of=/dev/xvdX conv=notrunc bs=1M ``
  * An important metric to remember is *VolumeQueueLength* - The number of read & write operations waiting to be completed in a specified period of time (e.g. - troubleshooting perf issues with a DB volume)
  * Volume Status Checks:
    * **ok** - everything is okay
    * **warning** - degraded or severely degraded
    * **impaired** - stalled or unaviable
    * **insufficient data** - no data present for the volume

******

### Monitoring RDS
Two types of monitoring for RDS
  
  1. CloudWatch: monitor by metrics
  2. RDS Console: metric graphs + events
    * Event Subscriptions
      * DB failover, config changes, low storage, maintenance, etc...
  
  * Important Metrics (for exam) -- skipping "credit" metrics
    * **DatabaseConnections** - The number of database connections in use
    * **DiskQueueDepth** (you want this to be zero) - The number of outstanding IOs (read/write requests) waiting to access the disk
    * **FreeStorageSpace** - The amount of available storage space
    * **ReplicaLag** - The amount of time a Read Replica DB instance lags behind the source DB instance. Applies to MySQL, MariaDB, and PostgreSQL Read Replicas (seconds)
    * **ReadIOPS** - The average number of disk I/O read operations per second
    * **WriteIOPS** - The average number of disk I/O write operations per second
    * **ReadLatency** - The average amount of time taken per disk I/O operation
    * **WriteLatency** - The average amount of time taken per disk I/O operation
    
  * Others
    * **BinLogDiskUsage** - The amount of disk space occupied by binary logs on the master
    * **CPUUtilization** - The percentage of CPU utilization
    * **FreeableMemory** - The amount of available random access memory
    * **SwapUsage** - The amount of swap space used on the DB instance
    * **ReadThroughput** - Average number of bytes read from disk per second
    * **WriteThroughput** - Average number of bytes written to disk per second
    * **NetworkReceiveThroughput** - Incoming traffic
    * **NetworkTransmitThroughput** - Outgoing traffic

    