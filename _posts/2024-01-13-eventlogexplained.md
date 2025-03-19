# A Comprehensive Guide to System Monitoring

In our [previous](https://ah11r.github.io/posts/weventviewer/) blog post, we explored the Windows Event Viewer as a powerful tool for monitoring and troubleshooting Windows systems. One of the most essential aspects of Event Viewer is Event IDs. These unique identifiers play a crucial role in understanding and diagnosing issues within your system. In this post, we'll delve deeper into the world of Event IDs, explaining what they are and examining some common and critical Event IDs across different logs.

### Understanding Event IDs:

Event IDs are numerical codes assigned to specific events in the Windows Event Logs. They serve as a quick reference to the type and significance of an event. Each Event ID corresponds to a particular event or error, making it easier to pinpoint issues and take appropriate action.

### Common Event IDs and Their Meanings:

Let's explore some common Event IDs across various Event Viewer logs and their relevance to a security operations center analyst:

**1. Event ID 6005 (System Log) - The event log service was started:**

* This Event ID indicates that the Event Log service has started. It can be useful for tracking system reboots and log service initiation.\
  In a SOC, this Event ID may be relevant for tracking system availability and potential disruptions.

**2. Event ID 4624 (Security Log) - An account was successfully logged on:**

* This Event ID signifies a successful user logon. It includes details like the account name, logon type, and source IP address, which is crucial for security monitoring.

**3. Event ID 7045 (System Log) - A service was installed in the system:**

* This Event ID indicates the installation of a service. It provides information about the service name, its binary path, and the account under which it runs.

**4. Event ID 41 (Kernel-Power Log) - The system has rebooted without cleanly shutting down first:**

* This Event ID signals an unexpected system restart, which can be caused by hardware failures, power issues, or system crashes.

**5. Event ID 1000 (Application Log) - Application Error:**

* Event ID 1000 appears when an application crashes or encounters an error. It includes details about the faulty application and module, aiding in troubleshooting.

### Event IDs for Specific Scenarios:

Aside from the common Event IDs mentioned above, there are specialized Event IDs that play a significant role in specific scenarios:

**Active Directory Events:**

**1. Event ID 4740 (Security Log) - A user account was locked out:**

* This Event ID indicates that a user's account has been locked due to multiple failed login attempts. It provides information about the account name and source IP address.

**2. Event ID 4720 (Security Log) - A user account was created:**

* This Event ID signifies the creation of a new user account in Active Directory. It includes details about the account name and the user who created it.

**3. Event ID 5136 (Security Log) - A directory service object was modified:**

* This Event ID logs changes made to directory service objects, such as user accounts or groups. It provides details about the modification, including the object's attributes.

**4. Event ID 4769 (Security Log) - A Kerberos service ticket was requested:**

* This Event ID is generated when a user requests a service ticket for a specific service. It includes information about the requesting user and the requested service.

**5. Event ID 1102 (Security Log) - The audit log was cleared:**

* This Event ID alerts you when the security audit log has been cleared. It's essential for maintaining audit trail integrity.

**Windows Security Events:**

**1. Event ID 4625 (Security Log) - An account failed to log on:**

* This Event ID signifies a failed login attempt. It provides details about the failed logon, including the account name and the reason for the failure.

**2. Event ID 5156 (Security Log) - Windows Filtering Platform has allowed a connection:**

* This Event ID logs successful network connections allowed by Windows Firewall. It includes information about the application and protocol used.

**3. Event ID 4719 (Security Log) - System audit policy was changed:**

* This Event ID indicates changes to the system's audit policy, helping monitor alterations to security settings.

**4. Event ID 4663 (Security Log) - An attempt was made to access an object:**

* This Event ID records attempts to access objects such as files or folders. It provides details about the object and the user or process attempting access.

**5. Event ID 4688 (Security Log) - A new process has been created:**

* This Event ID is generated when a new process is created on the system, providing information about the process and its parent process.

### Event IDs for Application and System Events:

**Application and System Events:**

**1. Event ID 1001 (Application Log) - Windows Error Reporting:**

* This Event ID provides information about errors reported by Windows Error Reporting, helping diagnose application crashes.

**2. Event ID 55 (System Log) - The file system structure on the disk is corrupt and unusable:**

* This Event ID warns of potential file system corruption on a disk drive. It's crucial for proactive maintenance and data integrity.

**3. Event ID 6008 (System Log) - The previous system shutdown at \[timestamp] on \[date] was unexpected:**

* This Event ID helps identify unexpected system shutdowns and their causes.

**4. Event ID 1001 (Application Log) - Fault bucket \[error code]:**

* This Event ID is related to application crashes and provides additional information about error codes and potential solutions.

**5. Event ID 45 (System Log) - The system could not create an error report:**

* This Event ID indicates issues with generating error reports, which can be useful for diagnosing errors related to Windows Error Reporting.

### Networking and DNS Events:

**1. Event ID 4227 (System Log) - TCP/IP failed to establish an outgoing connection:**

* This Event ID logs TCP/IP connection failures, which can be essential for diagnosing network connectivity issues.

**2. Event ID 1014 (System Log) - Name resolution for the name \[hostname] timed out:**

* This Event ID records DNS resolution timeouts, helping identify DNS-related problems.

**3. Event ID 2011 (System Log) - The server's configuration parameter "irpstacksize" is too small for the server to use a local device:**

* This Event ID is related to network-related errors and can provide insights into issues affecting network resource access.

**4. Event ID 5152 (Security Log) - The Windows Filtering Platform blocked a packet:**

* This Event ID logs blocked network traffic, which can be crucial for monitoring network security.

**5. Event ID 2504 (System Log) - The server could not bind to the transport:**

* This Event ID can help diagnose issues related to network service binding and startup.

### Summary:

Event IDs are indispensable tools in Windows Event Viewer for monitoring, diagnosing, and troubleshooting issues within your system. Familiarizing yourself with common Event IDs and their meanings can significantly enhance your ability to proactively manage and maintain your Windows-based systems. By regularly reviewing Event Viewer logs and paying.\
In the subsequent posts, we will cover a hands on lab where we will exammine some of these logs.
