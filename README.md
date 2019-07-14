# Mattermost Raspberry Pi Recipe

## Problem

I want to install Mattermost on my Raspberry Pi for use on my network. 

## Solution

The Raspberry Pi architecture is not officially supported by Mattermost, but there is an excellent resource available with steps and helpful links [Mattermost on Raspberry Pi](https://kartoffelsalat.ddns.net/post/mattermost-raspi/), which further linked me to a really great repo with the latest install files provided by [SmartHoneyBee](https://github.com/SmartHoneybee/ubiquitous-memory/releases/). 

The builds are updated regularly, so the most recent version of Mattermost (5.21) is available. There are installers for quite a few flavors of Linux. The version I used was **Linux-arm-tar.gz** as I'm running Raspbian 9 (stretch) on my Raspberry Pi 3 Model B which has an ARM Cortex-A53 processor. You can check this by running *uname --kernel-name --kernel-release" --machine*. If your Pi is out the box, the results should be something like **Linux 4.14.34-v7+ arm71**. If you've installed a different flavor of Linux, it'll be different so choose the appropriate install file.  

Setting up MySQL
----------------
Before you install Mattermost, you need to set up MySQL. The full instructions are on the [Installing Mattermost on Debian Stretch](https://docs.mattermost.com/install/install-debian.html/) page and cover PostgreSQL and MySQL. I use MySQL, so these are the steps I followed: 

1. Log into the server that will host the database, and open a terminal window.

2. Download the MySQL repository package 
```bash
wget https://dev.mysql.com/get/mysql-apt-config_0.8.6-1_all.deb
```

3. Install the repository 
```bash
sudo dpkg -i mysql-apt-config*
```

4. Update your local package list
```bash
sudo apt-get update
```

5. Add the MySQL repo MySQL. During the install, you’ll be prompted to create a password for the MySQL root user. Make a note of the password because you’ll need it in the next step.
```bash
sudo apt-get install mysql-server
```

6. Log in to MySQL as root, using the root password that you created when installing MySQL.
```bash
sudo mysql -u root -p
``` 

7. Create the Mattermost user 'mmuser', replacing 'mmuser-password' in the command below with the password you plan to use. 
```bash
mysql> create user 'mmuser'@'%' identified by 'mmuser-password';
``` 

The ‘%’ means that 'mmuser' can connect from any machine on the network. However, it’s more secure to use the IP address of the machine that hosts Mattermost. For example, if you install Mattermost on the machine with IP address 10.10.10.2, then use ```bash 
mysql> create user 'mmuser'@'10.10.10.2' identified by 'mmuser-password';```

8. Create the Mattermost database
```bash
mysql> create database mattermost;
```

9. Grant access privileges to the user 'mmuser' 
```bash 
mysql> grant all privileges on mattermost.* to 'mmuser'@'%';
```

10. Log out of MySQL
```bash 
mysql> exit
```

During this process I encountered an issue where I wasn’t prompted to choose a MySQL root password so I couldn’t log in as root. While you may not encounter that, I found some handy steps at [How to set, change, and recover a MySQL root password] (https://www.techrepublic.com/article/how-to-set-change-and-recover-a-mysql-root-password/). 

Install Mattermost Server
--------------------------

1. Download the appropriate install file
```bash
wget https://github.com/SmartHoneybee/ubiquitous-memory/releases/download/v5.12.3/mattermost-v5.12.3-linux-arm.tar.gz
```

2. Extract the Mattermost Server files from the *.tar.gz* file
```bash
tar -xvzf mattermost*.gz
```

3. Move the extracted file to the */opt* directory 
```bash
sudo mv mattermost /opt

```
4. Create the storage directory for files 
```bash
sudo mkdir /opt/mattermost/data
```
The storage directory will contain all the files and images that your users post to Mattermost, so you need to make sure that the drive is large enough to hold the anticipated number of uploaded files and images.


5. Set up a system user and group called "mattermost" that will run this service, and set the ownership and permissions.
    1. Create the Mattermost user and group 
```bash 
sudo useradd --system --user-group mattermost
```
   2. Set the user and group mattermost as the owner of the Mattermost files
```bash 
sudo chown -R mattermost:mattermost /opt/mattermost
```
   3. Give write permissions to the mattermost group 
```bash 
sudo chmod -R g+w /opt/mattermost
```
   4. Set up the database driver in the file */opt/mattermost/config/config.json*. Open the file in a text editor and make the following changes:
    1. Set "DriverName" to "mysql"
    2. Set "DataSource" to the following value, replacing <mmuser-password> and <host-name-or-IP> with the appropriate values. 
    
 ```mmuser:<mmuser-password>@tcp(<host-name-or-IP>:3306)/mattermost?charset=utf8mb4,utf8&readTimeout=30s&writeTimeout=30s```
    
Start the Mattermost Server 
---------------------------
1. Change to the bin directory
```bash 
cd /opt/mattermost
```
2. Start the Mattermost server as the user mattermost 
```bash
sudo -u mattermost ./bin/mattermost
```
3. To test that you can access the server, navigate to ```<your-IP-address>:8065```. 

When the server starts the output log generates information, including the current version of Mattermost and the listening port (8065). You can stop the server by pressing CTRL+C on your keyboard. 

Set up Mattermost to Use systemd for Starting and Stopping.
---------------------------------------------------------
If you want your Mattermost server to start up automatically on boot, you can create a systemd file to install it as a service. 

1. Create a systemd unit file
```bash
sudo touch /lib/systemd/system/mattermost.service
```
2. Open the unit file as root in a text editor, and copy the following lines into the file:
```[Unit]
Description=Mattermost
After=network.target
After=mysql.service
Requires=mysql.service
```

**Note:** If you are using PostgreSQL, replace *mysql.service* with *postgresql.service*. If you have installed MySQL or PostgreSQL on a dedicated server then you need to remove the After=postgresql.service and Requires=postgresql.service or After=mysql.service and Requires=mysql.service lines in the [Unit] section or the Mattermost service will not start.

```[Service]
Type=notify
ExecStart=/opt/mattermost/bin/mattermost
TimeoutStartSec=3600
Restart=always
RestartSec=10
WorkingDirectory=/opt/mattermost
User=mattermost
Group=mattermost
LimitNOFILE=49152

[Install]
WantedBy=multi-user.target
```

3. Make systemd load the new unit
```bash 
sudo systemctl daemon-reload
```
4. Check to make sure that the unit was loaded
```bash
sudo systemctl status mattermost.service
```

You should see an output similar to the following:

```bash
mattermost.service - Mattermost
Loaded: loaded (/lib/systemd/system/mattermost.service; disabled; vendor preset: enabled)
Active: inactive (dead)
 ```

5. Start the service
```bash
sudo systemctl start mattermost.service
```
6. Verify that Mattermost is running 
```bash
curl http://localhost:8065
```
You should see the HTML that’s returned by the Mattermost server. In my case it was a message referring to a connection error, which was a bit confusing. So I re-checked the service status by running 
```bash
sudo systemctl status mattermost.service
``` 
and confirmed that the service was active. 

7. Finally, set Mattermost to start on boot
```bash
sudo systemctl enable mattermost.service
```
The output should indicate that a symlink has been successfully created. 


Inviting Team Members
---------------------
The last step in the process is to make the Mattermost server available to team members so they can join the server. To do this, log into the server using ```http://<your-IP-address>:<port>``` and follow the configuration steps. Once complete, open the main menu, select **Get Team Invite Link**, copy the link, and send it out.


## Discussion

If I wanted to deploy multiple Mattermost Pi appliances, I’d need to repeat this process step-by-step multiple times. One solution to this is to use a Docker image. For this recipe I didn’t use a Docker image as Docker is far more resource-intensive to use and on a Pi you generally want to run something as light as possible. A second reason is that creating a Docker image requires Docker knowledge and that learning curve is a bit steeper than simply following the steps provided. 
However, it is a better solution should you wish to deploy Mattermost across multiple Pis. 

This setup was done on my local network, which meant that my users couldn't join the server when they were off my network. I worked around this by by forwarding port 8065 to my Pi in my router settings and providing them with my DNS host name instead of the IP address. 
