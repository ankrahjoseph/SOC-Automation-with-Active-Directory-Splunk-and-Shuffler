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

**Splunk VPC**

![Splunk VPC]()

**Project-ADDC01 VPC and Adapter Settings**

![Proj-ADDC VPC]()  
![Proj-ADDC Adapter]()

**Col-Vult02 VPC and Adapter Settings**

![Col-Vult VPC]()  
![Col-Vult Adapter]()  
>I used the domain controller's private IP address as the preffered DNS server so I can join the domain.

After configuring the Adapters, I pinged the servers to confirm they could talk to each other.

On the Project-ADDC01 server, I installed Active Directory Services and set **myprojad.local** as the domain when I promoted it to a domain controller. I created three user accounts and made one a Domain Admin so I could easily change settings on the servers. On the Col-Vult02 I joined the domain and authenticated with the admin account I created.

![Col-Vult02]()  

I also created a security group and added all the users to it then on the Col-Vult02 VM, I allowed the security group to connect to it via RDP. I used the security group to make it easier to manage and not have to allow each user individually. Hence, any new account created just needs to be added to the security group to have access to connect to Col-Vult02 via RDP.

I connected to Project-Splunk VM via SSH and run **$ apt-get update && apt-get upgrade** to update the server. After the update, I visited splunk enterprise [download](https://www.splunk.com/en_us/download/splunk-enterprise.html) page and copied the wget command for linux .deb package.

![Splunk download]()

After running the command on the Project-Splunk VM, I run **$ dpkg -i splunk###.deb** to install Splunk on the server.

![Splunk install]()

After installation, I navigated to /opt/splunk/bin and run **$ ./splink start** to start the service and configure the admin credentials. I enabled TCP port 8000 from my computer's IP on the firewall group on Vultr and run **$ ufw allow 8000** on the splunk server to allow incoming connections to the splunk portal.

On my computer I accessed the splunk portal via web browser with public IP of Project-Splunk VM and port 8000 (140.82.1.26:8000) then logged on as admin with the credentials created via the installation.

![Logon Splunk]()

I changed the timezone settings in Splunk to EST to personalize log timestamps, installed splunk windows add-on App, created a new index (myprojad) and configured splunk to receive data through port 9997. 

![Splunk Rec Config]()

I downloaded the Splunk universal forwarder for Windows Server 2022 and isntalled it on both of Windows Server VM,configired the forwarder to send to receiver at Private IP of the Splunk server on port 9997 (10.1.96.4:9997) then I navigated to **C:\Program Files\SplunkUniversalForwarder\etc\system\default** on both VMs, copied the **inputs.conf** file to **C:\Program Files\SplunkUniversalForwarder\etc\system\local** and edited the **inputs.conf** file in the **local** directory by appending the file with:

**[WinEventLog://Security]**  
**index = myprojad**  
**disabled = false**  

I restarted the SplunkForwarder Service on both VMs and made sure they log on as localSystem. I went back to the splunk portal and searched for index=myprojad got telemetry from both hosts.

![Telemetry]()

I tested put a few searches and came up with a search input to detect suspicious login even via RDP. 

>index="myprojad" EventCode=4624 (Logon_Type=7 OR Logon_Type=10) Source_Network_Address=* Source_Network_Address!="-" Source_Network_Address!=104.*

The search looks for Events with EventCode 4624 (successful logon), logon type 7 or 10 (RDP), having a Source IP, Source IP not being blank and Source IP not starting from 104.###.###.###.

In this scenario, our company's IPs start with 104.###.###.### so any IP outside that range is suspicious. I loggen on to the test computer as Psmith with a different IP and got a hit.

![First Sus Event]()

I piped the command to **stats count by _time,ComputerName,Source_Network_Address,user,Logon_Type** and saved it as an alert; the alert is set to trigger in realtime and triger per result. When the alert is triggered, it adds to Triggered Alerts and sends to a Shuffler SOAR webhook.

![Alert Settings]()

![triggered alert]()  
>Triggered Alert

## Shuffler SOAR
![ShufflerPlay]()  
Logic flow of Playbook

When the webhook receives the alert from Splunk, it forwards it to the Slack Node.  
I authenticated the slack node to use the Slack SOC Workplace I created, configured the text to send and specified the channel to post into with the channel unique ID.  
>Link to the Slack Alert Notification Node [screenshot]().  

After the Slack Alert is sent, it sends a Query alert to SOC email asking if user should be disabled.  
>Link to Query Alert Node [screenshot]().

When SOC Analyst clicks **No**, the workflow aborts but when the SOC Analyst clicks **Yes**, the workflow continues.

If the SOC Analyst clicks **Yes**, the Disable_User_action Node (Active Directory) run. I authenticated the login details, IP, Port and domain for Shuffler to be able disable the user.  
>Link to Disable_User_action Node [screenshot]().

After user is disabled, Get User Attributes node is run to check if user is disabled then continue the workflow to send a Slack confimation that the user was disabled. If after checking attributes and user is not disabled, no confirmation would be sent to the SOC channel in Slack then the engineer can toubleshoot the playbook for issues. This node uses the same AD authentication set for the previous node.
>Link to Get User Attributes Node [screenshot]().

The Check AD user Node (repeat back to me) calls for the **AccountControl** attribute and compares to find if it contains **"ACCOUNTDISABLED"** before continuing the flow if it returns true.
>Link to Check AD User Node [screenshot]().

If the logic returns true, the Update Notification Node sends a confirmation to the Slack channel.
