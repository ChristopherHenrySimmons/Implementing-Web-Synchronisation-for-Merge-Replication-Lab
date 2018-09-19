# Implementing-Web-Synchronisation-for-Merge-Replication-Lab

The Web synchronisation option for SQL Server Merge Replication enables data replication using the HTTPS protocol over the Internet. These notes describe how this form of Merge Replication is implemented. The lab notes called Database Replication in the Lab section on Moodle are essential reading if the instructions below are to make sense.  To use Web synchronisation for Merge Replication, the following major steps must be performed:
1.	Create the databases for the Publisher and Subscriber(s).
2.	Configure a new publication to allow Web synchronisation.
3.	Configure the computer that is running Microsoft Internet Information Services (IIS) to synchronise subscriptions
4.	Configure one or more new subscriptions to use Web synchronisation.
Step 1: Create Databases for Publisher and Subscribers
a.	The first step is to change the name of the server on which the publication and subscription database will be stored so that it is the same as the machine name of your computer. This is required to perform replication, since we must provide an actual server name rather than an alias during the configuration process. To change the server name, connect to SQL Server. Now, open a New Query Window and enter the following and then execute the following query: PE203-30\MSSQLSVR
select @@SERVERNAME
go
Copy the server name returned from this query and use it to replace ‘old-server-name’ in the command below. Use the server name on your machine (as seen when you connect to SQL Server - e.g. ‘PE203-20\MSSQLSVR’) to replace ‘new-server-name’. In both cases, include the quote marks to enclose the server names you provide in the commands below.
sp_dropserver 'old-server-name'
go
sp_addserver 'new-server-name', LOCAL
go
b.	Now Close SQL Server, re-start the computer (important) and then connect to SQL Server again, but this time, use the new server name that you specified above. 

c.	We will be replicating a database used for recording property details by a real estate company. In the Lab section on Moodle, you will find links called RealEstatePub and RealEstateSub. The RealEstatePub link will open a text file that you should copy and paste into a New Query Window within SQL Server. This will create the database used by the Publisher. 
d.	Perform the same step for the link RealEstateSub, which will create the database used by a Subscriber. 

This step would be performed for each Subscriber i.e. each of the laptops used by the estate agents. 

It should be noted that initially, both databases are exactly the same and differ only in the name that each are called. The purpose of the steps that follow are to implement a mechanism that will reconcile changes made in either the Publisher database or the Subscriber database when an estate agent chooses to synchronise the two over the Internet.

Step 2: Configuring the Publication
To use Web synchronisation, first we must create a publication. In this scenario, the publication is the central database located at head office. In addition, since we are using a Publisher for the first time, we must also configure a Distributor and a snapshot share (i.e. the mechanism used for storing a snapshot of the distribution database). The Merge Agent (the program that reconciles the databases on the Subscriber and Publisher) at each Subscriber must have read permissions on the folder used to implement snapshot share. We will begin by configuring your own Windows account so that it can be used by the Merge Agent to perform its role. To do this, use Windows Explorer to navigate to the following folder:
C:\Program Files\Microsoft SQL Server\MSSQL14.MSSQLSERVER\MSSQL\repldata
After navigating to this folder, you will be prompted to give yourself access rights to it (which will include the required Read permissions for the Merge Agent that will use your account); 
Click Continue.
Start SQL Server Agent and Create a Publication and Define Articles 
1.	In Microsoft SQL Server Management Studio, connect to the Publisher (using the new server name you defined in Step 1 above), and locate the SQL Server Agent (the object at the very bottom of the Object Explorer pane. Right-click on the SQL Server Agent object and select Start from the short-cut menu. The replication agents we use run as jobs scheduled by SQL Server Agent. SQL Server Agent must be running for the jobs to run
2.	Expand the Replication folder, and then right-click the Local Publications folder.
3.	Click New Publication.
4.	Follow the pages in the New Publication Wizard to:
•	Specify a Distributor.
Specify on the Distributor page that the Publisher server will act as its own Distributor (a local Distributor), and since the server is not configured as a Distributor, the New Publication Wizard will configure the server. Specify the snapshot folder (the folder named above – which should be the default) for the Distributor on the Snapshot Folder page. The snapshot folder is simply a directory that you have designated as a physical location for agents to share data; agents that read from and write to this folder must have sufficient permissions to access it. If we were to specify that another server should act as the Distributor, we simply enter a password on the Administrative Password page for connections made from the Publisher to the Distributor. 
•	Choose a publication database (RealEstatePub).
•	Select a publication type (select Merge Replication)
•	Specify data and database objects to publish (select all the table objects listed, but not the stored procedures)
•	Set the Snapshot Agent schedule (accept the defaults)
•	Specify the credentials under which the Snapshot replication agent will run and make connections e.g. ‘PCSUPPORT\haradr’ i.e. the full user account name listed on your machine when connecting to SQL Server and your network login password. These credentials will also be specified for the Merge Agent in a later step.
•	Specify a name for the publication (RealEstatePub).

Step 3: Configure IIS 7 For Web Synchronisation
The procedures in this step will guide you through the process of manually configuring Microsoft Internet Information Services (IIS) version 8 for use with Web synchronisation for merge replication. Manual configuration is necessary because the Configure Web Synchronisation Wizard is not supported on IIS 8.0. (Even though it was in IIS 6.0 and even 5.0!), which would have largely automated this step.
To use Web synchronisation, you must configure IIS 8 by completing the following steps. 
•	Install and configure the Microsoft SQL Server Replication Listener on the computer that is running IIS.
•	Configure Secure Sockets Layer (SSL). SSL is required for communication between IIS and all subscribers.
•	Configure IIS authentication.
•	Configure an account and set permissions for the SQL Server Replication Listener.

Installing and Configuring the SQL Server Replication Listener
1.	Create a new file directory for replisapi.dll on the computer that is running IIS. Create the directory C:\Inetpub\SQLReplication\
2.	Copy replisapi.dll from the directory C:\Program Files\Microsoft SQL Server\140\com\ to the file directory that you created above.
3.	Register replisapi.dll:
a.	Start the Command Prompt and navigate to C:\Inetpub\SQLReplication\
b.	In this directory execute the following command:
regsvr32 replisapi.dll
4.	We could use the Default Web Site for replication but as we need to reconfigure many of the web site settings, we will create a new web site so that your existing applications will still run. This Web site will be accessed by replication components during synchronisation.
5.	Create a new folder in C:\inetpub called RealEstate and copy the files iisstart.html and iis-start.PNG from C:\inetpub\wwwroot to the new folder. These files will display the default web page that will confirm the correctness of the configuration steps we will perform below.
6.	In Internet Information Services (IIS) Manager, in the Connections pane, select the Default Web Site and click on Stop in the Actions Pane. Right-click on the Sites node and select Add Web Site. Enter RealEstate as the site name in the dialog box and browse to the new folder you created to specify the new web sites physical location. Then click on OK.
7.	Create a virtual directory in IIS. The virtual directory should be created under the Web site that you created in step 6 and map it to the directory created in step 1 (i.e. C:\Inetpub\SQLReplication). 
a.	In Internet Information Services (IIS) Manager, in the Connections pane, right-click on the RealEstate web site, and then select Add Virtual Directory.
b.	For Alias, enter SQLReplication.
c.	For Physical Path, enter C:\Inetpub\SQLReplication\, and then click OK.
8.	Configure IIS to enable replisapi.dll to execute. 
a.	In Internet Information Services (IIS) Manager, click the RealEstate Web Site.
b.	In the centre pane, double-click Handler Mappings.
c.	In the Actions pane, click Add Module Mapping.
d.	For Request Path, enter replisapi.dll.
e.	From the Module drop-down list, select IsapiModule.
f.	For Executable, enter C:\Inetpub\SQLReplication\replisapi.dll.
g.	For Name, enter Replisapi.
h.	Click the Request Restrictions button, click the Access tab, and then click Execute.
i.	Click OK to close the Request Restrictions dialog box, and then click OK again to close the Add Module Mapping dialog box. When you are prompted to allow the ISAPI extension, click Yes to add the extension.
j.	Verify that Replisapi.dll is listed under the Enabled handler mappings. Right-click the Replisapi entry and then click Edit Feature Permissions. Check the Execute box, and then click OK.

Configuring IIS Authentication
When subscriber computers connect to IIS, IIS must authenticate the subscribers before they can access resources and processes. Authentication can be applied to the whole Web site or to the virtual directory that you created.
To Configure IIS Authentication
1.	In Control Panel, click Programs and Features, and then click Turn Windows Features on or off. 
2.	Expand Internet Information Services, expand World Wide Web Services, expand Security, select Basic Authentication, and then click OK. 

 

3.	Close and re-open IIS Manager, if already open, to refresh with Basic Authentication option we have just turned-on.
4.	In Internet Information Services (IIS) Manager, click on the RealEstate Web Site.
5.	In the middle pane, double-click Authentication.
6.	Right-click Anonymous Authentication, and then choose Disable.
7.	Right-click Basic Authentication, and then choose Enable.



Configuring Secure Sockets Layer (SSL)
To configure SSL, specify a certificate to be used by the computer running IIS. To configure IIS for deployment, you must first obtain a certificate from a certification authority (CA). However, we will generate a self-signed certificate for testing. The only difference between deploying for production and the procedures given here is that in production and pre-production testing, you would use a certificate issued by a CA instead of a self-signed certificate.
To configure SSL, you will perform the following steps:
1.	Create a self-signed certificate.
2.	Bind the certificate to the replication Web site.
3.	Configure the Web site to require SSL and ignore client certificates.
To create a self-signed certificate for testing
1.	In Internet Information Services (IIS) Manager, click the local server node (the topmost node in the Connections pane – named after your machine e.g. PE203-20), and then in the centre pane, double-click on Server Certificates.
2.	In the Actions pane, click Create Self-Signed Certificate.
3.	In the Create Self-Signed Certificate dialog box, enter a name for the certificate (RealEstate), and then click OK.
To bind a certificate to a Web site
1.	In the Connections pane, click on the RealEstate Web Site. 
2.	In the Actions pane, click Bindings, and then click Add. The Add Site Binding dialog box will appear.
3.	From the Type drop-down list, select https. Leave the default settings for IP address and Port.
4.	From the SSL certificate drop-down list, select the certificate created above (RealEstate) click OK, and then click Close.
To require SSL security for a Web site
1.	In the Connections pane, click the RealEstate Web Site.
2.	In the middle pane, double-click SSL Settings.
3.	Check the Require SSL option. Under Client certificates, verify that the Ignore button is selected.
4.	In the Actions pane, click on Apply.

To test the certificate
1.	In the Connections pane, click the RealEstate Web Site.
2.	From the Actions pane, click Browse *:443(https).
3.	Internet Explorer will open and display a message that "This site is not secure…”. This warning tells you that the associated certificate was not issued by a recognised CA and might not be trustworthy. This is an expected warning, so click “Details” and then,  “Go on to this website (not recommended)”.
4.	If you are prompted to Connect to localhost, enter a user name and password to proceed (e.g. ‘PCSUPPORT\haradr’ + your network login password you use when you first start a lab machine – NOT your Moodle password). You should see the IIS default page for the Web site.
Setting Permissions for the SQL Server Replication Listener
When a subscriber computer connects to the computer running IIS, the subscriber is authenticated by using the type of authentication specified when you configured IIS. After IIS authenticates the subscriber, IIS checks whether the subscriber is authorized to invoke SQL Server replication. You control the users that can invoke SQL Server replication by setting permissions for replisapi.dll. Properly configuring permissions is necessary to prevent unauthorized access to SQL Server replication. 
To configure the minimum permissions for the account under which the SQL Server Replication Listener runs, complete the following procedures. 
Important The account created in this section is the account that will connect to the Publisher and Distributor during synchronisation. This account must be added as a SQL Login account on the distribution and publication server.
The account used for the SQL Server Replication Listener must have the following permissions: 
•	Be a member of the Publication Access List (PAL – performed in next section).
•	Be mapped to a login associated with a user in the publication database.
•	Be mapped to a login associated with a user in the distribution database.
•	Have Read permissions on the snapshot share.
In addition to the above requirements, the required logins should also be in the publication access list (PAL). 
The steps required to meet these requirements are detailed below:
To configure the Replication Listener account and set its permissions
1.	Create a local account on the computer running IIS:
a.	Open Computer Management – Right click on the Windows Start button and select Computer Management from the short-cut menu.
b.	Expand System Tools, and then expand Local Users and Groups.
c.	Right-click Users, and then click New User.
d.	Enter a user name and password (RepListen/RepListen). 
e.	Clear User must change password at next logon. (Important)
f.	Click Create, and then click Close.
2.	Add the account to the IIS_IUSRS group:
a.	In Computer Management, expand System Tools, expand Local Users and Groups, and then double-click Groups.
b.	Right-click IIS_IUSRS, and then click Add to Group.
c.	In the IIS_IUSRS Properties dialog box, click Add.
d.	In the Select Users, Computers, or Groups dialog box, add the account created in step 1 (i.e. RepListen).
e.	Verify that From this location displays the name of the local computer e.g. PE203-20 (not a domain PCSUPPORT). If this field does not display the local computer name, click Locations. In the Locations dialog box, select the local computer, and then click OK. Click on Check Names button to ensure the RepListen user is recognised.
f.	In the Select Users dialog box and the IIS_IUSRS Properties dialog box, click OK.
3.	Map the account to a SQL Server Login 
The Replication Listener account must be mapped to an SQL Server login on the distribution and publication server.
a.	Connect to the server name we created in SQL Server (e.g. PE204-20) and expand the Security node (located beneath the Database node in the Object Explorer pane). Right-click on Logins and Select New Login. 

b.	Click on the Search button to the right of the Login Name textbox. Click on the Advanced button in the Select User or Group dialog. Click on the Find Now button and then select the RepListen Windows user you created in step 1. 

c.	Click on the OK button and OK again to return to the Login – New window.

d.	Select the Server Roles page and ensure that the account has Public checked.

e.	Select the User Mappings page and check the Map check box against the RealEstatePub database and check the db_owner role.

f.	Click on the OK button to complete the mapping of RepListen account.

4.	Add the new login (i.e. RepListen) to the PAL list 

a.	In the Object Explorer pane of SQL Server, expand the Replication node, then the Local Publication node and then right click on the publication and select Properties from the short cut menu.

b.	In Publication Access List page of the Publication Properties - <Publication> dialog box, first check that the user group used when connecting to SQL Server (i.e. PCSUPPORT\Domain Users) is listed. This account group will be used by the Merge and Snapshot Agents, who also need to be on the PAL list. Do not remove distributor_admin from the PAL. This account is also used by replication. 

c.	Now click on the Add button and select the RepListen account you previously created to add this account to the list. Finally, Click on OK to complete the new login.
5.	Grant minimum account permissions on the folder that contains replisapi.dll:
a.	In Windows Explorer, right-click the folder that you created for replisapi.dll (C:\Inetpub\SQLReplication\), and then click Properties. (Ensure the folder, not the file is right-clicked)
b.	On the Security tab, click Edit.
c.	In the Permissions for <foldername> dialog box, click Add to add the account that you created in step 1 (RepListen).
d.	Verify that From this location displays the name of the local computer (not a domain). If this field does not display the local computer name, click Locations. In the Locations dialog box, select the local computer, and then click OK. Click on Check Names button to ensure the RepListen user is recognised.
e.	Verify that the account is granted only Read, Read & Execute and List folder contents permissions. Then click on OK.
6.	View/Create an application pool in Internet Information Services (IIS) Manager:
a.	In Internet Information Services (IIS) Manager, click Application Pools, and verify the application pool (RealEstate) exists and that it uses the .Net Framework Version 4. Leave the default values for the remaining fields.

The SQL Server Replication Listener supports two concurrent synchronisation operations per thread. Exceeding this limit may cause the Replication Listener to stop responding. The number of threads allocated to replisapi.dll is determined by the application pool Maximum Worker Processes property. By default, this property is set at 1. If we anticipate having more than two concurrent synchronisation clients, then we will need to create a web garden.
You can support a greater number of concurrent synchronisation operations per CPU by increasing the Maximum Worker Process property value. Scaling out by increasing the number of worker processes per CPU is known as "Web gardening."
Web gardening will allow more than two Subscribers to synchronise at the same time. It will also increase CPU utilisation by replisapi.dll, which can negatively impact overall server performance. It is important to balance these considerations when you choose a value for Maximum Worker Processes.
7.	To increase Maximum Worker Processes in IIS 8
a.	In Internet Information Services (IIS) Manager, expand the local server node, and then click on the Application Pool node.
b.	Select the application pool associated with the Web synchronisation site (RealEstate), and then click Advanced Settings on the Actions pane.
c.	On the Advanced Settings dialog, under the Process Model heading, click the row labelled Maximum Worker Processes. Change the property value to 10.
8.	Associate the account with the application pool:
a.	Also within the Process Model section, click the Identity field. 
b.	Click the ellipsis button on the right side of the Identity row. 
c.	Click the Custom Account radio button, and then click Set. 
d.	In the User name and Password fields, enter the account and password that were created in step 1 (RepListen/RepListen), and then click OK.
9.	Confirm that the application pool is associated with the replication Web site: 
a.	In Internet Information Services (IIS) Manager, expand the local server node, and then click on the RealEstate web site.
b.	In the Actions pane, under Browse Web Site, click Advanced Settings.
c.	Confirm that the entry under Application Pool is the same name as the application pool you created in Step 4 (i.e. RealEstate). If it is not, click on the ellipsis button to the right of Application Pool and then select the application pool you previously created from the drop-down list, and then click OK.
d.	Click OK again to close Advanced Settings.


 Testing the Connection to replisapi.dll
Run Web synchronisation in diagnostic mode to test the connection to the computer running IIS and to make sure that the Secure Sockets Layer (SSL) certificate is properly installed. To run Web synchronisation in diagnostic mode, you must be an administrator on the computer running IIS.
To test the connection to replisapi.dll
1.	Make sure that local area network (LAN) settings at the Subscriber are correct:
a.	From the Windows Start Icon, type Internet Options.  
b.	In the Internet Properties dialog box that now appear, click on the Connections tab and click on LAN Settings.
c.	Clear the Automatically Detect Settings and the Use a proxy server for your LAN (if selected).
2.	At the Subscriber, in Internet Explorer, connect to the server in diagnostic mode by appending ?diag to the address for the replisapi.dll. For example:

 https://PE204-20.pcsupport.ac.nz/SQLReplication/replisapi.dll?diag 

The PE204-20.pcsupport.ac.nz component of the above address must be exactly the same as the Issued to: name specified in the Server Certificate. Check this name in IIS by clicking on the top-most node in the Connections pane and double-clicking on Server Certificates in the middle pane.
3.	Since the certificate that you specified for IIS is not recognised by the Windows operating system, you will be prompted to log in to the server. Use the username and password for the Merge Agent that will use to connect to IIS (i.e. your network account, which is also used by the Snapshot Agent in our configuration). These credentials will also be specified in the New Subscription Wizard.

e.g. Username: PCSUPPORT\haradr  Password: your network p/w (Not your Moodle p/w)
4.	In the Internet Explorer window called SQL Websync diagnostic information, verify that the value in each Status column on the page is SUCCESS.
To configure a merge publication to allow Web synchronisation
1.	After you complete the New Publication Wizard, enable the Web synchronisation option for the publication:
a.	In SQL Server, right click on the Publication you created in Step 2, p2. and select properties from the short-cut menu. Now select the FTP Snapshot and Internet page of the Publication Properties - <Publication> dialog box and select the option to Allow Subscribers to synchronise by connecting to a Web server. 
b.	Enter an address for IIS in the Address of Web server to which Subscribers should connect text box. The Web server address, 
(e.g.   https://PE204-20.pcsupport.ac.nz/SQLReplication/replisapi.dll), specifies the location of replisapi.dll. 
c.	The Web address that you specify on this page is used as the default address for all subscriptions to this publication. You can override the address on the Web Server Information page of the New Subscription Wizard. For example, you can have one set of Subscribers use one instance of IIS, and a second set of Subscribers use a second instance of IIS.
2.	Click OK.

Step 4: Configuring the Subscription
After you enable a publication and configure IIS, create a pull subscription and specify that the pull subscription should synchronise by using IIS. (Web synchronisation is supported only for pull subscriptions.)
To create a pull subscription from the Subscriber
1.	Connect to the Subscriber (in our case, this is the same as the Publisher e.g. connect using the new server name we created earlier) in SQL Server Management Studio, and then expand the Replication folder.
2.	Right-click the Local Subscriptions folder, and then click New Subscriptions.
3.	On the Publication page of the New Subscription Wizard, select the RealEstate publication.
4.	Select where replication agents will run. For a pull subscription (which is the type we will use), select Run each agent at its Subscriber (pull subscriptions) on the Merge Agent Location page.
5.	On the Subscribers page, click in the check box and select RealEstateSub as the subscription database.
6.	On the Merge Agent Security page, provide your Windows login as credentials (e.g. PCSUPPORT\haradr). This is the account that will used by the Merge Agent in our configuration.

To configure the pull subscription that you have just created so that it uses Web synchronisation, use the New Subscription Wizard, as described in the following procedure.

To configure a subscription to use Web synchronisation
1.	On the Web Synchronisation page, select Use Web Synchronisation for each Subscriber that should use this synchronisation method.
2.	On the Web Server Information page:
a.	Accept the Web address that is specified when you configured the publication for Web synchronisation.
b.	Select a type of authentication to use when connecting to IIS. You can only use a type of authentication that has been enabled for IIS through the Configure Web Synchronisation Wizard or IIS Manager.
MS recommends that we use Basic Authentication with Secure Sockets Layer (SSL) as the method by which credentials are passed to the IIS. SSL is required, regardless of the type of authentication that is used. Since we will specify Basic Authentication, MS recommends that we specify the same account for the connection to IIS as the account that we specified for the Merge Agent in this wizard (e.g. PCSUPPORT\haradr).
3.	Select Basic Authentication option and supply the Merge agent account details (e.g. PCSUPPORT\haradr) to complete the wizard.
Testing the completed configuration
Finally…. we are now ready to test the Merge Replication and Web Synchronisation mechanism we have created. To perform the test, simply modify several records in both the subscription (RealEstateSub) and publication database (RealEstatePub). Then start the synchronisation steps, detailed below.
Synchronise the Pull Subscription
Subscriptions are synchronised by the Distribution Agent (for snapshot and transactional replication) or the Merge Agent (for merge replication). Agents can:
•	Run continuously
•	Run on demand
•	Run on a schedule
We will specify the ‘Run on demand’ option, which is the typical synchronisation method for Merge Replication. We synchronise a subscription on demand from the Local Subscriptions folder in Microsoft SQL Server Management Studio.
To synchronise a pull subscription on demand in Management Studio
1.	First, initialise the Snapshot Agent by expanding the Replication folder in SQL Server and then expand the Local Publications folder.
2.	Right-click the on the RealEstate publication and select View Snapshot Agent Status from the short-cut menu. 
3.	In the View Snapshot Agent Status dialog box, click on the Start button to initiate the Snapshot Agent. Click on the Close button once the process has completed.

These first three steps only need to be performed once. The next four steps should be performed whenever a subscriber (i.e. a Real Estate Agent in our example) wishes to synchronise with the publisher.
4.	Expand the Replication folder, and then expand the Local Subscriptions folder.
5.	Right-click the subscription you want to synchronise (RealEstateSub), and then click View Synchronisation Status.
6.	In the View Synchronisation Status dialog box, click Start. When synchronisation is complete, the message Synchronisation completed is displayed.
7.	Click Close.
Finally, check that publication and subscription databases have been synchronised.
