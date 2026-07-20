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

