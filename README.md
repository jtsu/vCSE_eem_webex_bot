# IOS-XE Post ConfigDiff 
This app uses the features and resources on the Cisco IOS-XE platforms.  The app will track config changes and post the diff to a collaboration platform. The app will post messages to Webex Teams, Microsoft Teams, and Slack.  Some network environments require the use  of a HTTP proxy server, so a proxy option has been added to our app.  The app will also run directly on a Cisco IOS-XE switch or router, so no additional server resources are needed.

## Use Case Description

Information regarding our use case can be found in our [use case statement](./USECASE.md).


## Installation
This app uses the following:
* Webex Teams, a Cisco Communications and Messaging Application.
  * Microsoft Teams or Slack can also be used.
* Cisco IOS XE Platform
  * This can be  a Cisco Catalyst 9000 Switch or Cisco Router platform that can supports IOS XE v16.5.1 or later.  
  * For our project, we used a Cisco CSR 1000v with IOS XE version 17.03.02.
* EEM in IOS-XE
  * The Embedded Event manager (EEM) is a software component of cisco IOS-XE that can track and classify events that take place and can help you to automate tasks.
* GuestShell in IOS-XE
  * The ability to execute Python code directly on a Cisco Switch is a part of the Application Hosting capabilities provided by GuestShell.  GuestShell is a containerized Linux runtime environment in which you can install and run applications, such as Python scripts.  From within GuestShell, you have access to the networks of the host platform, bootflash and IOS CLI.  GuestShell is isolated from the underlying host software to prevent interference of the core network functions of the device.
* Python Script
  * This is the part of the app that will process and post the config diff to Webex Teams.

## Collaboration Clients Configuration  
* This app supports posting messages to the following platforms:
  * Webex Teams
  * Microsoft Teams
  * Slack
* We're going to use *Incoming Webhooks* to post the config diffs to each client.  Incoming webhooks let you post messages when an event occurs in another service.
* The steps to configure Incoming Webhooks are very similar for each of our three clients.
  * Create a room/team/channel in each client where our app will post the config diffs
  * Connect the *Incoming Webhook(s)* App in each client.
  * Name and select the room/team/channel you created.
  * Copy each webhook URL to the python module, 'mytokens.py'.

## EEM Configuration
This is the IOS-XE Configuration for EEM Applet.
  ```
  csr1000v# conf t
  csr1000v(config)# event manager applet test
  csr1000v(config-applet)# event syslog pattern "%SYS-5-CONFIG_I: Configured from" maxrun 200
  csr1000v(config-applet)# action 0.0 cli command "en"
  csr1000v(config-applet)# action 1.0 cli command "guestshell run python3 configDiff.py"
  csr1000v(config-applet)# end
  ```

## GuestShell Configuration
* IOX needs to be enable on the IOX-XE platform for GuestShell.
  ```
  csr1000v# conf t
  Enter configuration commands, one per line.  End with CNTL/Z.
  csr1000v(config)# iox
  csr1000v(config)# end
  ```

* A VirtualPortGroup is used to enable the communication between IOS XE and the GuestShell container.
  ```
  csr1000v# conf t
  csr1000v(config)# interface VirtualPortGroup 0
  csr1000v(config-if)# ip address 192.168.1.1 255.255.255.0
  csr1000v(config-if)# end
  ```

* Configure the network settings that will get passed to GuestShell when it's enabled.  
  ```
  csr1000v# conf t
  csr1000v(config)# app-hosting appid guestshell
  csr1000v(config-app-hosting)# vnic gateway1 virtualportgroup 0 guest-interface 0 guest-ipaddress 192.168.1.2 netmask 255.255.255.0 gateway 192.168.1.1 name-server 208.67.222.222
  csr1000v(config-app-hosting)# end
  ```

* Configure NAT if an access from the container to the outside world is needed.
  ```
  csr1000v# conf t
  csr1000v(config)# interface VirtualPortGroup0
  csr1000v(config-if)#  ip nat inside
  !
  csr1000v(config-if)# interface GigabitEthernet1
  csr1000v(config-if)#  ip nat outside
  csr1000v(config-if)# exit
  !
  csr1000v(config)# ip access-list extended NAT-ACL
  csr1000v(config-ext-nacl)# permit ip 192.168.1.0 0.0.0.255 any
  !
  csr1000v(config-if)# exit
  csr1000v(config)# ip nat inside source list NAT-ACL interface GigabitEthernet1 overload
  csr1000v(config)# end
  ```

* Enable GuestShell on the IOX-XE platform.
  ```
  csr1000v# guestshell enable
  ```

- Enter GuestShell to install python and some needed modules.
  ```
  csr1000v# guestshell
  ```

* Optional: We have already defined a DNS Name Server in the app-hosting config for GuestShell, so this step isn't needed. But if you didn't want to configure DNS from IOX-XE, you could configure it directly in the GuestShell environment.  
  ```
  [guestshell@guestshell ~]$ echo "nameserver 208.67.222.222" | sudo tee --append /etc/resolv.conf
  ```

* Depending on the IOS-XE platform and version, you may need to install python and some additional utilities.
  ```
  [guestshell@guestshell ~]$ sudo yum update -y
  [guestshell@guestshell ~]$ sudo yum install -y nano python3 epel-release
  [guestshell@guestshell ~]$ sudo yum search pip | grep python3
  [guestshell@guestshell ~]$ sudo yum install -y python3-pip
  ```

* Install the python requests module, and use the optional proxy, if needed. Be sure to use the proxy url and port for your environment.
  ```
  [guestshell@guestshell ~]$ sudo pip3 install --proxy your.proxy-server.com:8080 requests
  ```

* Copy the python script to the EEM user policy directory.  
  - You can copy the script to a directory in GuestShell or you can create a directory on the flash from the IOS-XE CLI.
  - In the EEM config above, the script is located in the home path on GuestShell.
  - If you would like to copy the script to the bootflash, use the absolute path in the EEM config.

* Exit GuestShell and return to IOS-XE
  ```
  [guestshell@guestshell ~]$ exit
  ```

**NOTE:** The guestshell environment will persist across reboots.  To return to a default state, destory the guestshell and enable guestshell again.


## Optional Configuration - System Proxy Settings for GuestShell
If a proxy server is needed in your enviroment, you'll need to configure the following proxy settings in GuestShell.

* Create a proxy.sh shell script to add the proxy settings to the system profile.
  ```
  [guestshell@guestshell ~]$ sudo nano /etc/profile.d/proxy.sh
  ```

* Add the following parameters in to proxy.sh shell script.  Be sure to use the proxy url and port for your environment.
  ```  
  PROXY_URL="http://your.proxy-server.com:8080/"
  export http_proxy="$PROXY_URL"
  export https_proxy="$PROXY_URL"
  export ftp_proxy="$PROXY_URL"
  export no_proxy="127.0.0.1,localhost"
  export HTTP_PROXY="$PROXY_URL"
  export HTTPS_PROXY="$PROXY_URL"
  export FTP_PROXY="$PROXY_URL"
  export NO_PROXY="127.0.0.1,localhost"
  ```

* Source the profile to activate the proxy settings.
  ```
  [guestshell@guestshell ~]$ source /etc/profile
  [guestshell@guestshell ~]$ env | grep -i proxy
  ```

* Configure the proxy server for the Yum package manager.  Be sure to use the proxy url and port for your environment.
  ```
  [guestshell@guestshell ~]$ echo "proxy=your.proxy-server.com:8080" | sudo tee --append /etc/yum.conf

  ```

## Usage
To see the app in action, simply make a configuration change on your Cisco switch or router.  For example, you can change the description of an interface.
  ```
  csr1000v# conf t
  csr1000v(config)# interface GigabitEthernet3 
  csr1000v(config-if)# description Test Interface
  csr1000v(config-if)# end
  ```
**NOTE:** Be sure to exit configuration mode since EEM is looking for a specific syslog pattern.

After a config change has been made, login to WebEx Teams, or your preferred collaboration client, and check your room for the diff message.  

![Example of diff posted in Webex Teams](./diff_post_webex.png)


## References 
If you would like to get  more information on the topics or features used in our project, we compiled a list of links to white papers, documentation, helpful blogs and Cisco DevNet Learning Labs that may be of interest.  [Here's a link to our references](./USECASE.md#white-papers-and-helpful-blog-posts).


## Cisco DevNet Sandbox
For our project, we used the Cisco CSR 1000v. The Cisco CSR 1000v is a virtual router that can be easily deployed on a virtual machine.  However, if you don't have access to a Cisco IOS-XE platform, we recommend you try the Cisco IOS XE Sandboxes available at Cisco DevNet.  We included a sandbox reference in our USECASE documentation. [Here's a link to that reference](./USECASE.md#related-sandbox).


## Author(s)
This project was written and is maintained by the following individuals:
* Jason Su <jtsu@cisco.com>
* Dennis Tran <dentran@cisco.com>
* Matt Jennings <matjenni@cisco.com>
* Steven Tanti <stanti@cisco.com>




