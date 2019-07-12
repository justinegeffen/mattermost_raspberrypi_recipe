# Mattermost Raspberry Pi Recipe

## Problem

I want to install Mattermost on my Raspberry Pi for use on my network. 

## Solution

The Raspberry Pi architecture is not officially supported by Mattermost, but there is an excellent resource available with steps and helpful links [Mattermost on Raspberry Pi](https://kartoffelsalat.ddns.net/post/mattermost-raspi/), which further linked me to a really great repo with the latest install files provided by [SmartHoneyBee](https://github.com/SmartHoneybee/ubiquitous-memory/releases/). 

The builds are updated regularly, so the most recent version of Mattermost (5.21) is available. There are installers for quite a few flavors of Linux. The version I used was the **Linux-arm-tar.gz** as that’s the flavor I have running on my Pi. You can check this by running *uname --kernel-name --kernel-release" --machine*. If your Pi is out the box, the results should be something like **Linux 4.14.34-v7+ arm71**. If you've installed a different flavor of Linux, it'll be different so choose the appropriate install file.  

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
5. Add the MySQL repo MySQL
```bash
sudo apt-get install mysql-server
```

**Note:** During the install, you’ll be prompted to create a password for the MySQL root user. Make a note of the password because you’ll need it in the next step.

6. Log in to MySQL as root
```bash
sudo mysql -u root -p
``` 
When prompted, enter the root password that you created when installing MySQL.
7. Create the Mattermost user "mmuser"
```bash
mysql> create user 'mmuser'@'%' identified by 'mmuser-password';
``` 
where "mmuser-password" is the password you plan to use. 

The ‘%’ means that "mmuser" can connect from any machine on the network. However, it’s more secure to use the IP address of the machine that hosts Mattermost. For example, if you install Mattermost on the machine with IP address 10.10.10.2, then use ```bash 
mysql> create user 'mmuser'@'10.10.10.2' identified by 'mmuser-password';```

8. Create the Mattermost database
```bash
mysql> create database mattermost;
```
9. Grant access privileges to the user "mmuser" 
```bash 
mysql> grant all privileges on mattermost.* to 'mmuser'@'%';
```
10. Log out of MySQL
```bash 
mysql> exit
```

During this process I encountered an issue where I wasn’t prompted to choose a MySQL root password so I couldn’t log in as root. While you may not encounter that, I found some handy steps at [How to set, change, and recover a MySQL root password] (https://www.techrepublic.com/article/how-to-set-change-and-recover-a-mysql-root-password/). Once you’ve set the password, make sure you run ```sudo mysql -root -p``` otherwise you’ll hit a permissions error for localhost.

Install Mattermost Server
--------------------------
1. Log in to the server that will host Mattermost Server and open a terminal window.
2. Extract the Mattermost Server files from the *.tar.gz* file downloaded at the start of this process
```bash
tar -xvzf mattermost*.gz
```
4. Move the extracted file to the */opt* directory 
```bash
sudo mv mattermost /opt
```
5. Create the storage directory for files 
```bash
sudo mkdir /opt/mattermost/data
```

**Note:** The storage directory will contain all the files and images that your users post to Mattermost, so you need to make sure that the drive is large enough to hold the anticipated number of uploaded files and images.

Set up a system user and group called "mattermost" that will run this service, and set the ownership and permissions.
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
    
  Also make sure that the database name is "mattermost" instead of "mattermost_test"
 ```"mmuser:<mmuser-password>@tcp(<host-name-or-IP>:3306)/mattermost?charset=utf8mb4,utf8&readTimeout=30s&writeTimeout=30s"```
    
Test the Mattermost Server 
---------------------------
1. Change to the bin directory
```bash 
cd /opt/mattermost
```
2. Start the Mattermost server as the user mattermost 
```bash
sudo -u mattermost ./bin/mattermost
```

When the server starts, it shows some log information and the text server is listening on :8065. You can stop the server by pressing CTRL+C in the terminal window.

Set up Mattermost to Use systemd for Starting and Stopping.
---------------------------------------------------------
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
7. Set Mattermost to start on machine start up
```bash
sudo systemctl enable mattermost.service
```

You should see the HTML that’s returned by the Mattermost server. 

Inviting Team Members
---------------------
The last step in the process was to make my Mattermost server available on the network. Once everything is running, you can log into the Mattermost server using the IP address you listed in the config.json file. 

<create server, invite people> 

Configuration and Settings
--------------------------
Once everything is up and running, take a look at the Mattermost documentation for advanced configuration and settings. 

## Discussion

The process I used was pretty manual and, if I wanted to deploy multiple Pis in my network, I’d need to repeat the process step by step. The solution to this is to use a Docker image. The reason I didn’t use a Docker image is that Docker is far more resource-intensive to use and on a Pi you want to run something as light as possible. A second reason is that creating a Docker image requires Docker knowledge and that learning curve is a bit steeper than simply following the steps provided. 
However, it is a better solution should you wish to deploy Mattermost across multiple Pis. 

This setup was done on my local network, which meant that my users couldn't join the server when they were out of range. I worked around this by providing them with my DNS server name, which they used instead of the IP address. This allows it to resolve and join the server regardless of whether they're on my home network or not. 

  
