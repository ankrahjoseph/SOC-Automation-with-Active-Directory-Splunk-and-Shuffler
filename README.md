# SOC Automation with Active Directory Splunk and Shuffler
Set up Virtual Machines on the cloud, configured Active Directory and Splunk then setup a playbook on Shuffler Soar to Automate Incident Response whenever there is a suspicious RDP logon event.

>This project helps automate incidence response by allowing Security Analysts to stop attacks in a timely manner.

## Scenario
You have a company "MYPROJAD" that uses Active directory to manage its users and computers; RDP logon is allowed on its computers but you want to harden the system so that whenever a user logs on with an IP outside of the company's IP ranges, the SOC team is alerted and imediate action is taken. In this project, we will be using Splunk SIEM to centralized logs and set up alerts, Shuffler SOAR to automate actions taken on each incident.

I started by visualizing the logic and flow of the project using [Drawio](https://www.drawio.com).

![drawio]()

From the diagram, an Attacker succesfully authenticates into the test computer. The event is logged and sent to the Splunk server which triggers the playbook in Shuffler. An alert notification is sent to the SOC Team channel on slack and an email is sent to the SOC Team asking if the user account should be disabled. When the SOC Analyst clicks **NO** nothing happens but when **YES** is clicked, the playbook send a command to the Domain Controller to Disable the user. When succesfully disabled, the playbook sends confirmation to the Slack Channel.



