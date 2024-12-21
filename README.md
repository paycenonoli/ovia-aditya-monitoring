![image alt](https://github.com/paycenonoli/ovia-aditya-monitoring/blob/main/ovia-monitoring-jpg.docx)

ULTIMATE DEVOPS MONITORING PROJECT
Tools needed:
-	Prometheus
-	Blackbox exporter
-	Node exporter
-	Alertmanager

Number of VMs/ EC2 Instances: 2
Type: t2.medium

VM1: Install Prometheus/ Blackbox exporter/ Alertmanager
VM2: Install Node exporter/ NGINX

LINKS FOR DOWNLOAD
Prometheus- prometheus.io (You get all the links here-Prometheus, Blackbox exporter, Node exporter)
[wget + copy-link-address]

N/B: Here, we are using direct packages.
We can also set these tools up in the same way using Docker containers or directly in Kubernetes pods.
•	Download, install, extract and rename the tools
•	Configure the tools accordingly

# Clone an application from your GitHub repository to VM 2 (instead of NGINX) and use it as a website for your monitoring (for Blackbox).

-	Run the node_exporter
cd node_exporter
./node_exporter & (use & so that it runs in the background even when you use CTRL+C on the terminal)
-	Access the node_exporter on port 9100

-	Run your application
(To run the application and access it on the browser, we need to have java and Maven to build it to get the jar file and execute the jar file. So, we install java and Maven)

cd ovia-jaiswal-boardgame
sudo apt install openjdk-17-jre-headless
sudo apt install maven -y

mvn package
java -jar database_service_project-0.0.7.jar

Access the application on port 8080.

Now, your node_exporter is running, your application is running.
Next configure/ set up the other tools on the other VM.
cd prometheus
./prometheus &
Access prometheus on port 9090
Set up alert rules in prometheus
Create alert_rules.yaml (configure rules for alerting)
sudo vim alert_rules.yaml
*Use yamllint to validate your yaml file
Define the information in alert_rules.yaml in our prometheus.yml
Add your alert_rules.yaml under rules_files in prometheus.yml

Start the alertmanager:
cd alertmanager
./alertmanager &
Access on port 9093

Now, in prometheus we still do not see any rules, so we restart our prometheus:
1st: kill the prometheus process
pgrep prometheus -> gives you the process ID, then kill it and start prometheus once more:

kill 3421

./prometheus &

Re-access and you see the rules populated.

Next: we set up the Blackbox exporter
(Remember, the two exporters must be configured in prometheus)

Sudo vim prometheus.yml
(Meanwhile) Under alerting, configure the url of the alert manager
Configure the scrape_configs (We should have three job_names here: prometheus, node_exporter, and Blackbox exporter):

- job_name: "node_exporter" # Job name for node exporter 
# metrics_path defaults to '/metrics' 
# scheme defaults to 'http'.
  static_configs:
    - targets: [“3.110.195.114:9100”]  * for node_exporter

Restart prometheus and if you go under ‘Targets’ you can now see node_exporter as one of the targets.

Next we add the Blackbox exporter to the job_name under scrape_configs:

Go to prometheus.io/ Blackbox exporter/go to the github repo next to it.
Scroll down and copy from ‘job_name’ to ‘replacement’.

---
- job_name: blackbox
  metrics_path: /probe
  params:
    module:
      - http_2xx
  static_configs:
    - targets:
        - http://prometheus.io
        - https://prometheus.io
        - http://example.com:8080
  relabel_configs:
    - source_labels:
        - __address__
      target_label: __param_target
    - source_labels:
        - __param_target
      target_label: instance
    - target_label: __address__
      replacement: 127.0.0.1:9115



Things to edit:
1.	The url of your application (example.com:8080)
2.	On which machine your Blackbox exporter is running(replacement)

Now, restart prometheus.
Re-access prometheus and Blackbox shows as one of the targets but with Error scarping target(because we have not started Blackbox)

Start Blackbox
cd blackbox
./blackbox_exporter &

Access on port 9115

Go back to prometheus under targets
Blackbox is now exposing metrics and prometheus can scrape.


##########################################################################



NOW THE MONITORING PART IS DONE.
NEXT IS TO CONFIGURE EMAIL NOTIFICATIONS.


cd alertmanager
	Configure the alertmanager.yml

sudo vim alertmanager.yml

Add the script (from documentation):

route:
  group_by: ['alertname'] # Group by alert name
  group_wait: 30s # Wait time before sending the first notification
  group_interval: 5m # Interval between notifications
  repeat_interval: 1h # Interval to resend notifications
  receiver: 'email-notifications' # Default receiver
receivers:
- name: 'email-notifications' # Receiver name
  email_configs:
  - to: jaiswaladi246@gmail.com # Email recipient
  from: test@gmail.com # Email sender
  smarthost: smtp.gmail.com:587 # SMTP server
  auth_username: your_email # SMTP auth username
  auth_identity: your_email # SMTP auth identity
  auth_password: "bdmq omqh vvkk zoqx" # SMTP auth password
  send_resolved: true # Send notifications for resolved alerts 
inhibit_rules:
  - source_match:
      severity: 'critical' # Source alert severity
    target_match:
      severity: 'warning' # Target alert severity
    equal: ['alertname', 'dev', 'instance'] # Fields to match


Make sure SMTP port is open (587)
For auth_username and auth_identity, you can use your Gmail ID
to: obiorah42@gmail.com
from: monitoring@example.com (for e.g.)
For auth_password, you need to generate that:
To generate that, use the URL: 
https://myaccount.google.com/apppasswords

 *In your Google account, under Security, make sure ‘two-step verification is turned on.
Then visit the url to generate the password.

Restart alertmanager
Restart prometheus

NEXT, WE TEST THE ALERT!

Go to prometheus/ Alerts/ Let’s use ‘WebsiteDown’
Now, our website has been running okay.
So, we stop our website and if it is stopped, an alert should be triggered and we get a notification since we configured an alert rule for ‘WebsiteDown’.
So…when the website is stopped , in prometheus, under Alerts/alert_rules/WebsiteDown,
The mode will move from Inactive to Pending. Then after the set duration of 1 minute, if the website stays down, the mode goes from Pending to Firing, and we get an email notification.


How does Prometheus know that the website is down?
So, in companies what is done is…each node that is part of the Kubernetes cluster will have the node exporter set up by default. That is, running 24 hours/7.

In the alert_rules.yaml, there is a section for ServiceUnavailable. There, you can write the service name you want to monitor [e.g. NGINX]

#####################################################################################

*We configure our email for notification in the alertmanager.yaml file.



