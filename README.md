# SOC Automation with Active Directory Splunk and Shuffler
Set up Virtual Machines on the cloud, configured Active Directory and Splunk then setup a playbook on Shuffler Soar to Automate Incident Response whenever there is a suspicious RDP logon event.

>This project helps automate incidence response by allowing Security Analysts to stop attacks in a timely manner.

## Scenario
You work for a company "MYPROJAD" that uses Active directory to manage its users and computers; RDP logon is allowed on its computers but you want to harden the system so that whenever a user logs on with an IP outside of the company's IP ranges, the SOC team is alerted and imediate action is taken. In this project, we will be using Splunk SIEM to centralized logs and set up alerts, Shuffler SOAR to automate actions taken on each incident.

I started by visualizing the logic and flow of the project using [Drawio](https://www.drawio.com).

![drawio]()

From the diagram, an Attacker succesfully authenticates into the test computer. The event is logged and sent to the Splunk server which triggers the playbook in Shuffler. An alert notification is sent to the SOC Team channel on slack and an email is sent to the SOC Team asking if the user account should be disabled. When the SOC Analyst clicks **NO** nothing happens but when **YES** is clicked, the playbook send a command to the Domain Controller to Disable the user. When succesfully disabled, the playbook sends confirmation to the Slack channel.

## Set UP

In this Project, I set up the virtual machines on the cloud with [Vultr](https://my.vultr.com). I signed up for a FREE account using a referral link [https://www.vultr.com/?ref=9784406-9J] which gave me $300 credit to test the platform. After signing up, I deployed three Machines:

- **Project-ADDC01** (Domain COntroller): 2vCPUs, 4GB RAM, 80GB SSD and Windows Server 2022 Operating System with a public IP assigned.
- **Col-Vult02** (Test Computer): 1vCPU, 2GB RAM, 55GB SSD and Windows Server 2022 Operating System with a public IP assigned.
- **Project-Splunk** (Splunk Server): 4vCPUs, 8GB RAM, 160GB SSD and Ubuntu 22.04 Operating System with a public IP assigned.

![Dashboard]()

I created a firewall Group to only allow SSH and RDP from my laptop to secure the network. I later on enabled RDP for any IP to allow suspicious login to the test Machine.

![Firewall Rules]()

The Servers had to be able to communicate with each other so I enabled VPC Network on all the VMs. For the VMs runnings windows, I had to manually configure the network adapter for the private network.
>Splunk VPC

