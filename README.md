# Mattermost Raspberry Pi Recipe

## Problem

I want to install Mattermost on my Raspberry Pi for use on my network. 

## Solution

The Raspberry Pi architecture is not officially supported by Mattermost, but there is an excellent resource available with steps and helpful links [Mattermost on Raspberry Pi](https://kartoffelsalat.ddns.net/post/mattermost-raspi/), which further linked me to a really great repo with the latest install files provided by [SmartHoneyBee](https://github.com/SmartHoneybee/ubiquitous-memory/releases/). 

The builds are updated regularly, so the most recent version of Mattermost (5.21) is available. There are installers for quite a few flavors of Linux. The version I used was the **Linux-arm-tar.gz** as that’s the flavor I have running on my Pi. You can check this by running *uname --kernel-name --kernel-release" --machine*. If your Pi is out the box, the results should be something like **Linux 4.14.34-v7+ arm71**. If you've installed a different flavor of Linux, it'll be different so choose the appropriate install file.  

Before you install Mattermost, you need to set up MySQL. The full instructions are on the [Installing Mattermost on Debian Stretch](https://docs.mattermost.com/install/install-debian.html/) page and cover PostgreSQL and MySQL. I use MySQL, so these are the steps I followed: 

1. Log into the server that will host the database, and open a terminal window.
2. Download the MySQL repository package: ```wget https://dev.mysql.com/get/mysql-apt-config_0.8.6-1_all.deb```
3. Install the repository: ```sudo dpkg -i mysql-apt-config*```
4. Update your local package list: ```sudo apt-get update```
5. Add the MySQL repo MySQL: ```sudo apt-get install mysql-server```

**Note:** During the install, you’ll be prompted to create a password for the MySQL root user. Make a note of the password because you’ll need it in the next step.

6. Log in to MySQL as root: ```sudo mysql -u root -p```. When prompted, enter the root password that you created when installing MySQL.
7. Create the Mattermost user "mmuser": ```mysql> create user 'mmuser'@'%' identified by 'mmuser-password';``` where "mmuser-password" is the password you plan to use. 

The ‘%’ means that 'mmuser' can connect from any machine on the network. However, it’s more secure to use the IP address of the machine that hosts Mattermost. For example, if you install Mattermost on the machine with IP address 10.10.10.2, then use the following command: ```mysql> create user 'mmuser'@'10.10.10.2' identified by 'mmuser-password';```.

8. Create the Mattermost database: ```mysql> create database mattermost;```
9. Grant access privileges to the user "mmuser": ```mysql> grant all privileges on mattermost.* to 'mmuser'@'%';```
10. Log out of MySQL: ```mysql> exit```

During this process I encountered an issue where I wasn’t prompted to choose a MySQL root password so I couldn’t log in as root. While you may not encounter that, I found some handy steps at [How to set, change, and recover a MySQL root password] (https://www.techrepublic.com/article/how-to-set-change-and-recover-a-mysql-root-password/). Once you’ve set the password, make sure you run ```sudo mysql -root -p``` otherwise you’ll hit a permissions error for localhost.

When that’s done, you install Mattermost server as follows: 
1. Log in to the server that will host Mattermost Server and open a terminal window.
2. Extract the Mattermost Server files from the *.tar.gz* file downloaded at the start of this process: ```tar -xvzf mattermost*.gz```
4. Move the extracted file to the */opt* directory: ```sudo mv mattermost /opt```
5. Create the storage directory for files: ```sudo mkdir /opt/mattermost/data```

**Note:** The storage directory will contain all the files and images that your users post to Mattermost, so you need to make sure that the drive is large enough to hold the anticipated number of uploaded files and images.

Set up a system user and group called "mattermost" that will run this service, and set the ownership and permissions.
1. Create the Mattermost user and group: ```sudo useradd --system --user-group mattermost```
2. Set the user and group mattermost as the owner of the Mattermost files: ```sudo chown -R mattermost:mattermost /opt/mattermost```
3. Give write permissions to the mattermost group: ```sudo chmod -R g+w /opt/mattermost```
4. Set up the database driver in the file */opt/mattermost/config/config.json*. Open the file in a text editor and make the following changes:
    1. Set "DriverName" to "mysql"
    2. Set "DataSource" to the following value, replacing <mmuser-password> and <host-name-or-IP> with the appropriate values. Also make sure that the database name is "mattermost" instead of "mattermost_test":
```"mmuser:<mmuser-password>@tcp(<host-name-or-IP>:3306)/mattermost?charset=utf8mb4,utf8&readTimeout=30s&writeTimeout=30s"```

Test the Mattermost Server 
---------------------------
1. Change to the bin directory: ```cd /opt/mattermost```
2. Start the Mattermost server as the user mattermost: ```sudo -u mattermost ./bin/mattermost```

When the server starts, it shows some log information and the text server is listening on :8065. You can stop the server by pressing CTRL+C in the terminal window.

Set up Mattermost to Use systemd for Starting and Stopping.
---------------------------------------------------------
Create a systemd unit file:
sudo touch /lib/systemd/system/mattermost.service
Open the unit file as root in a text editor, and copy the following lines into the file:
[Unit]
Description=Mattermost
After=network.target
After=mysql.service
Requires=mysql.service

**Note:** If you are using PostgreSQL, replace *mysql.service* with *postgresql.service*. If you have installed MySQL or PostgreSQL on a dedicated server then you need to remove the After=postgresql.service and Requires=postgresql.service or After=mysql.service and Requires=mysql.service lines in the [Unit] section or the Mattermost service will not start.

[Service]
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
Note


Make systemd load the new unit: ```sudo systemctl daemon-reload```
Check to make sure that the unit was loaded: ```sudo systemctl status mattermost.service```

You should see an output similar to the following:

● mattermost.service - Mattermost
  Loaded: loaded (/lib/systemd/system/mattermost.service; disabled; vendor preset: enabled)
  Active: inactive (dead)
Start the service.
sudo systemctl start mattermost.service
Verify that Mattermost is running.
curl http://localhost:8065

You should see the HTML that’s returned by the Mattermost server.

Set Mattermost to start on machine start up.
sudo systemctl enable mattermost.service
Now that the Mattermost server is up and running, you can do some initial configuration and setup.

Now that you’re all logged in and set up, you should be able to log into the server using the IP address you listed in the config.json file.
  
## Discussion

The process I used was pretty manual and, if I wanted to deploy multiple Pis in my network, I’d need to repeat the process step by step. The solution to this is to use a Docker image. The reason I didn’t use a Docker image is that Docker is far more resource-intensive to use and on a Pi you want to run something as light as possible. A second reason is that creating a Docker image requires Docker knowledge and that learning curve is a bit steeper than simply following the steps provided. 
However, it is a better solution should you wish to deploy Mattermost across multiple Pis. 

  
