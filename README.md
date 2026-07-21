# unraid-apps
A collection of UNRAID app templates I maintain based on upstream Docker images. All credit goes to original authors. 

## [`jenkins-inbound-agent`](https://jenkins.io)

This template is for the [Jenkins Inbound Agent](https://hub.docker.com/r/jenkins/inbound-agent) image. The container serves as a build agent for a pre-existing Jenkins instance. If you don't have one setup, you can try ich777/Jenkins in the unRAID Community Apps as a starting place. 

Prior to installing this application, you'll need to do a few things: 

1. You'll need to create the app directory and the working directory manually in Terminal: 

`mkdir -p /mnt/user/appdata/jenkins-inbound-agent/agent`

2. You'll need to add a new node in your Jenkins instance. 
    a. Navigate to Manage Jenkins > Nodes > "New Node". 
    b. Name your node, pick an all lowercase name, no spaces, e.g. `unraid-jenkins-agent`.
    c. Select `Permanent Agent` as the agent type.

3. Configure the node in Jenkins. After you select `Permanent Agent` and hit next, you should see a configuration page. Change the following items, leave everything else to the defaults:
    a. Number of executors `1` --> `2` or above, depending on how beefy your server is. 
    b. Remote root directory ` ` --> `/home/jenkins/agent`
    You can go ahead and save after this point, any other changes you make I assume you know what you're doing here. 

4. After configuring the node, you'll need to grab the following info to update the unRAID Template:
    a. JENKINS_URL: This should be the URL to access your jenkins instance, e.g. https://jenkins.acme.com/
    b. JENKINS_AGENT_NAME: This will be the name you entered earlier, e.g. `unraid-jenkins-agent`. 
    c. Navigate to Nodes and select your new node by clicking its name. On the Status page you see a bunch of commands, inside of them is your JENKINS_SECRET. You should see it as part of a string like this: `... -url https://jenkins.acme.com/ -secret sdaf65bq3ab563q5v65vb6va5a..` -name. You want to copy the entire string that's between `-secret` and `-name`. 

After doing all of that, you can update the Jenkins Agent Name, Jenkins Instance Secret, and Jenkins URL fields in the template with the info gathered above, and then hit apply. After the container starts up, you can watch the logs and you should see something like this: 

```
Jul 20, 2026 11:15:10 AM hudson.remoting.Launcher createEngine
INFO: Setting up agent: jenkins-agent-02
Jul 20, 2026 11:15:10 AM hudson.remoting.Engine startEngine
INFO: Using Remoting version: 3383.vc8881d4b_0e76
Jul 20, 2026 11:15:10 AM hudson.remoting.Engine startEngine
WARNING: No Working Directory. Using the legacy JAR Cache location: /home/jenkins/.jenkins/cache/jars
Jul 20, 2026 11:15:11 AM hudson.remoting.Launcher$CuiListener status
INFO: WebSocket connection open
Jul 20, 2026 11:15:11 AM hudson.remoting.Launcher$CuiListener status
INFO: Connected
```

You'll also see the status of the node change in Jenkins, and the number of build executors should match the number you selected when configuring the node. 

And that's it, your node is configured and you can now send builds from your pipelines to it! All credit goes to the amazing team over at [Jenkins](https://www.jenkins.io/). 

## [`uvdesk-unraid`](https://www.uvdesk.com/en/)

### What is UVdesk?

[**UVdesk**](https://www.uvdesk.com/en/) is a open source helpdesk software solution. It has support for a multitude of features including the following: 

* A multi-customer & multi-agent ticket system.
* A full mailer system to send mail to agents and customers.
* A fully featured knowledge base for self-service.
* CMS Platform Support
* Plugin Support
* Custom branding


#### Requirements

* You will need to have a **local** mySQL or MariaDB instance running on the same docker network as uvdesk. 
* You will need to create a `uvdesk` database and `uvdesk` user with full privileges on that database *prior* to downloading uvdesk from Community Applications. Please refer to [Setting up the Database](#setting-up-the-database) for instructions.
* You will need a reverse proxy setup. 

#### Reverse-Proxy

Setting up a reverse proxy is a task all unto itself, and best left to better authors than I: 

* IBRACORP NGINX Proxy Manager Guide: <https://youtu.be/h1a4u72o-64>
* IBRACORP NGINX Proxy Manager **with Cloudflare** Guide: <https://www.youtube.com/watch?v=c6Y6M8CdcQ0>
* Spaceinvaderone's Reverse Proxy Guide (SWAG): <https://youtu.be/I0lhZc25Sro>

There are other guides, but these are all unRAID specific and should get you what you need to setup a reverse proxy. Note, the reverse proxy **must be setup prior to installing uvdesk**.



#### Setting up the Database

If you don't know how to setup a database in mySQL, it sounds scarier than it is. First, download a mariaDB or mySQL image from Community Applications. In the template configuration, for the database user, database name, and database password, use whatever you want here and save it somewhere. Please note this is *different* from the **root** password, which some of these database containers will randomly generate on first startup. For those, you'll want to open the console after first creating it and save the root password from the logs in case you need it later or if you plan on running multiple databases in the container. 

#### Installing uvdesk from Community Applications

To install uvdesk from Community Applications, simply navigate to Community Apps and head to either **Productivity** or **Tools**, or just search `uvdesk`. 

#### First-Time Setup & Accessing UVdesk

##### Configuring the Container
When you get to the container configuration page in unRAID, make sure to fill in the following fields: 

* Timezone: Helpful for the timing on everything to be localized to you, especially with tickets. should be set to something like `America/New York`, based on your timezone. 
* External DB Host: If you're running your DB on Unraid, this will be the IP of your Unraid server. If not, supply the appropriate address. 
* External DB Port: This should be `3306` but if you're running off of a non-standard port, update accordingly. 
* App Environment: This should stay `prod`, the `dev` instance is ephemeral and won't save data at all. 
* Site URL: This should be the full FQDN (Fully Qualified Domain Name) of your site, the one you have setup in your reverse proxy or tunnel, e.g. `helpdesk.acme.com`. You **should not** put `https://` in front. 

##### Web Installer Wizard
Once you've filled in the fields, you can hit apply. Afterwards, hitting the WebUI button should take you to the installer wizard, alternatively you can navigate to `http:UNRAID-IP:6744`. 

**It is recommended that if you use password-management browser extensions that you run this in an incognito/private window. There's a breaking interaction with password managers and the wizard.**

You can run through the wizard, filling out the appropriate fields. The server version on the first page should be set to whatever version of mySQL you're running. The oldest supported version is `5.7.44` but ideally you should run something newer. 

##### Accessing the webUI
After the install, you should be able to navigate to your site's admin panel at https://mysite.com/en/member/login. Login with the account you made during setup and you will arrive at the dashboard!