# AWS Certified SysOps Administrator - Associate

Welp, here go my notes. Most of this is knowledge I already possess. These are notes to memorialize my formal study as it pertains to the certification. 

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

		