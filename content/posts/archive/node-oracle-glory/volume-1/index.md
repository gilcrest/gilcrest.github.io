---
title: "My Path to node.js and Oracle Glory - Volume 1"
date: 2015-03-09T20:33:00-04:00
description: Oracle XE Setup on Windows machine
menu:
  sidebar:
    name: Volume 1
    identifier: volume-1
    parent: node-oracle-glory
    weight: 1200
---

I am starting a new personal project with the following goals in mind:

1. Learn JavaScript (I'm a beginner, but been reading/implementing for a while)
1. Learn node.js
1. Create a RESTful API using node.js calling a remote database server using Oracle's node driver (node-oracledb)

* API should be simple to understand
* API in the backend should use the HR sample schema
* When using node.js, goal is to form a service request object in javascript and pass that object to a stored procedure in Oracle as an IN parameter, the same stored procedure should return a response object that will be parsed in javascript and the results displayed

> DISCLAIMER - When I actually have free time to develop - I am primarily a PL/SQL developer -- I love PL/SQL - there I said it!  I work with java and perl, but at the end of the day, I love the insulated, hassle-free world of developing within a database.  I know enough of command line to get by, but my 4th major goal is to become a command line "expert" (or at least know where all the relevant references I need are) for sqlplus and more importantly Linux, so bear with me as I toil through the basics along the way!

So, on to the actual work - I am going to setup two machines on my home network -- one machine (an old Windows 7 laptop) will act as a database server, the other machine (my new Chromebook) will act as the calling application (in this case node.js using Oracle's **node-Oracledb** driver).

Step 1.  Install Oracle XE on my windows machine using this [URL][xe-dl] - I won't go into depth on how to install XE as it's well documented.

~~Step 2.~~ DO NOT DO THIS!  Live and learn - I had no idea that the HR and other sample schemas are already installed with XE and went through all the below steps to clone and install the schema, only to figure out that I had accidentally installed the HR objects in the SYSTEM schema (I've now corrected the steps)....  Anyway, if for some reason, you need to install Oracle's sample schemas in your XE instance using Christopher Jones' db-sample-schemas on Github at this URL, then follow these steps!

* If you're not familiar with Github, it's definitely worth your while to learn about it, though there is a learning curve (I still suck at it, actually, but getting there...)

* Install Github for Windows using this [URL][gh-desktop]

* Once installed, you can just go to [https://github.com/oracle/db-sample-schemas.git][hr-gh] and choose "Clone in Desktop" and choose the folder location you wish to store the db-sample-schemas repository.  If you're not comfortable with Github yet - you can always get the same files by choosing "Download ZIP"

* After you've cloned or downloaded the files, navigate to the human_resources directory using the Windows command line interface (cmd.exe).

* When you're in the directory, type sqlplus - you'll then be asked to login - **be sure to login as the HR schema**.  I am assuming you've already created the HR schema.  If you have not, you should - you'll need to figure that out on your own though.  You can look at the hr_main.sql in the human_resources directory that you just downloaded for an example of creating HR.

* After you're logged in as HR in sqlplus, just type @hr_cre.sql - sqlplus will then execute the script and create all the necessary objects.  You can then type @hr_popul.sql to populate the tables you just created with data!

* Anyway, if you screw things up like I did, Christopher has nicely provided a script hr_drop.sql that you can run whenever necessary.

Step 3.  Open up Windows firewall for Port 1521:  

* To enable remote calls from my Chromebook, I need to enable Firewall rules on my Windows machine so that data can flow in and out of the machine through port 1521 (the standard port that Oracle is configured for)

* First, you want to be sure your listener is up and running for the Oracle XE instance you installed (it should be by default), but you can check in the Windows command line interface (cmd.exe) by typing **lsnrctl status** (you don't have to browse anywhere in particular for this command)

  * You can find out some critical items from this status, ie your HOST and PORT (will be 1521 by default) which you'll need when connecting later

  * Click here for more information on [Managing Network Connections][mnc] - doc shows you how to stop and start your db listener if you need me to...

* Assuming you are using the default port, next step is to open up connections (inbound and outbound) on it

  * Goto Start -> Control Panel -> System and Security -> Windows Firewall -> Advanced settings
    * Once in Advanced Settings, you'll see a window titled "Windows Firewall with Advanced Security"
    * click on **Inbound Rules** and choose the "New Rule..." action
    * Choose "Port" when asked What type of rule would you like to create?
    * Choose TCP when asked Does this apply to TCP or UDP?
    * Add 1521 next to Specified Local Ports
    * Choose Allow the connection
    * Rule applies to whichever domain you choose...
    * Give the rule a name - I chose **OracleInboundPort**, click Finish
    * Rinse and repeat the above for the **Outbound Rules**, just name the rule appropriately, i.e. **OracleOutboundPort**

## That's it for today - next up Volume 2, Configuring Linux on my Chromebook

[xe-dl]:   http://www.oracle.com/technetwork/database/database-technologies/express-edition/downloads/index.html
[gh-desktop]: https://help.github.com/articles/getting-started-with-github-for-windows/
[hr-gh]:   https://github.com/oracle/db-sample-schemas.git
[mnc]:     http://docs.oracle.com/cd/E17781_01/server.112/e18804/network.htm#ADMQS162
