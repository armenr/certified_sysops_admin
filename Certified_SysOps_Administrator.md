# AWS Certified SysOps Administrator - Associate

Welp, here go my notes.

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