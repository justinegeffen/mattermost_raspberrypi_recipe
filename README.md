# Mattermost Raspberry Pi Recipe

## Problem

I want to install Mattermost on my Raspberry Pi for use on my network. 

## Solution

The Raspberry Pi architecture is not officially supported by Mattermost, but there is an excellent resource available with current downloadable builds. I found it on `Mattermost on a Rasperry Pi <https://kartoffelsalat.ddns.net/post/mattermost-raspi/>`__ and navigated over to https://github.com/SmartHoneybee/ubiquitous-memory/releases. The builds are updated regularly, so the most recent version is available. There are installers for quite a few flavors of Linux. The version I used was the Linux-arm-tar.gz. as that’s the flavor I have running on my Pi. You can check this by running uname -r or if you installed it yourself you’d probably already know. 

`Mattermost Manifesto <https://www.mattermost.org/manifesto/>`__

Before you install Mattermost, follow the instructions on this page to set up MySQL: https://docs.mattermost.com/install/install-debian.html. You can choose to use PostGres or MySQL. I used MySQL so I followed these steps: 
Log into the server that will host the database, and open a terminal window.
Download the MySQL repository package.
wget https://dev.mysql.com/get/mysql-apt-config_0.8.6-1_all.deb
Install the repository
sudo dpkg -i mysql-apt-config*
Update your local package list.
sudo apt-get update
Add the MySQL repo MySQL.
sudo apt-get install mysql-server

Note

During the install, you’ll be prompted to create a password for the MySQL root user. Make a note of the password because you’ll need it in the next step.

Log in to MySQL as root.
mysql -u root -p

When prompted, enter the root password that you created when installing MySQL.

Create the Mattermost user ‘mmuser’.
mysql> create user 'mmuser'@'%' identified by 'mmuser-password';

Note

Use a password that is more secure than ‘mmuser-password’.
The ‘%’ means that mmuser can connect from any machine on the network. However, it’s more secure to use the IP address of the machine that hosts Mattermost. For example, if you install Mattermost on the machine with IP address 10.10.10.2, then use the following command:
mysql> create user 'mmuser'@'10.10.10.2' identified by 'mmuser-password';

Create the Mattermost database.
mysql> create database mattermost;
Grant access privileges to the user ‘mmuser’.
mysql> grant all privileges on mattermost.* to 'mmuser'@'%';
Log out of MySQL.
mysql> exit
With the database installed and the initial setup complete, you can now install the Mattermost server. Log into the server that will host the database, and open a terminal window.
Download the MySQL repository package.
wget https://dev.mysql.com/get/mysql-apt-config_0.8.6-1_all.deb
Install the repository
sudo dpkg -i mysql-apt-config*
Update your local package list.
sudo apt-get update
Add the MySQL repo MySQL.
sudo apt-get install mysql-server

Note

During the install, you’ll be prompted to create a password for the MySQL root user. Make a note of the password because you’ll need it in the next step.

Log in to MySQL as root.
mysql -u root -p

When prompted, enter the root password that you created when installing MySQL.

Create the Mattermost user ‘mmuser’.
mysql> create user 'mmuser'@'%' identified by 'mmuser-password';

Note

Use a password that is more secure than ‘mmuser-password’.
The ‘%’ means that mmuser can connect from any machine on the network. However, it’s more secure to use the IP address of the machine that hosts Mattermost. For example, if you install Mattermost on the machine with IP address 10.10.10.2, then use the following command:
mysql> create user 'mmuser'@'10.10.10.2' identified by 'mmuser-password';

Create the Mattermost database.
mysql> create database mattermost;
Grant access privileges to the user ‘mmuser’.
mysql> grant all privileges on mattermost.* to 'mmuser'@'%';
Log out of MySQL.
mysql> exit
With the database installed and the initial setup complete, you can now install the Mattermost server. 


That said, I encountered an issue where I wasn’t prompted to choose a MySQL root password so I couldn’t log in as root. I found some handy steps here: https://www.techrepublic.com/article/how-to-set-change-and-recover-a-mysql-root-password/. Once you’ve set the password, make sure you run ‘sudo mysql -root -p’ otherwise you’ll hit a permissions error for localhost.

When that’s done, you install Mattermost server as follows: 
<steps>

Now that you’re all logged in and set up, you should be able to log into the server using the IP address you listed in the config.json file.
  
## Discussion

The process I used was pretty manual and, if I wanted to deploy multiple Pis in my network, I’d need to repeat the process step by step. The solution to this is to use a Docker image. The reason I didn’t use a Docker image is that Docker is far more resource-intensive to use and on a Pi you want to run something as light as possible. A second reason is that creating a Docker image requires Docker knowledge and that learning curve is a bit steeper than simply following the steps provided. 
However, it is a better solution should you wish to deploy Mattermost across multiple Pis. 

  
