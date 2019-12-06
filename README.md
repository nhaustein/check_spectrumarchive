# Introduction to check_spectrumarchive.sh
This utility performs status checks of IBM Spectrum Archive Enterprise Edition components. The components that can be checked are:
- status of software
- state of Spectrum Archive EE nodes
- state of tape drives
- state of tapes
- state of pools and pool utilization
- state of running and completed task
- all components together.

The utility uses the Spectrum Archive EE admin command line interface (eeadm) to obtain the status of the components in json format. The output of eeadm command is parsed and a decision is derived whether the status of the component is in GOOD, WARNING or ERROR state. This utility writes the results to standard out in one line including the status of the components and the return code is in accordance with the detected status. For components that are not in GOOD state further details are displayed. 

The utility is based on Spectrum Archive EE version 1.3 and above. It requires the jq tool to be installed on all Spectrum Archive EE nodes where this tool runs. 

The utility can be invoked to display the status of a single component or of all components. If all components are checked (option -e) then the utility can raise event notifications to the IBM Spectrum Scale event monitor. This requires the custom events to be configured. 

## Syntax
This utility can be invoked with one parameter at a time and performs the appropriate checks. 

	usage: ./check_spectrumarchive.sh [ -s | -n | -t | -d | -p<util> | -a<r|c> -h ]
	
	Options:
        -s			--> Verify IBM Spectrum Archive status
        -n			--> Verify node status
        -t			--> Verify tape states
        -d			--> Verify drive states
        -p<util>	--> Check if tape pool utilization is above %util
        -a<r|c>		--> Check if running or completed tasks have failed
		-e			--> Check all components and optionally send event notifications. 
        -h			--> Print This Help Screen

The utility returns `OK`, `WARNING` or `ERROR` including the component and the appropriate return 0, 1 or 2 respectively.

Only one option can be specified at a time. The combination of multiple options in one call of the utility does not work. However the `option -e` will check all components and send event notifications if the custom events have been configured. 

This utility can be used standalone or it can be integrated with a external Icinga or nagios monitoring server. 


## Installation
Clone the git and transfer the `check_spectrumarchive.sh` utility to each Spectrum Archive EE node that needs to be monitored. Make the utility executable. 

Install the dependencies outline below. 

The utility can now be used from the command line or by an automated process. 

Optionally integrate the utility with an external monitoring tool such as Icinga. Or integrate the utility with the IBM Spectrum Scale event notifications.  


### Dependencies
This utility is based on Spectrum Archive EE version 1.3 and relies on the eeadm command. The EE admin command is specified in parameter $EE_ADM_CMD within the script. It has not been tested with the older ltfsee command. 

The tool [jq](https://stedolan.github.io/jq/) is used to parse the json output generated by the eeadm command. The jq tool needs to be install on all nodes where this tool runs on (all Spectrum Archive nodes being monitored). The default location where jq is expected is /usr/local/bin. This path can be changed within the script (paramater: $JQ_TOOL).
More information about jq: https://stedolan.github.io/jq/


## Integration option
There are two option to integrate this utility:

1. Integration with the IBM Spectrum Scale event monitoring framework
2. Integration with Icing

These two options are further discussed below.


# Integration with IBM Spectrum Scale event monitoring
This utility can be invoked to check all components (option -e) and send events to the IBM Spectrum Scale monitoring framework in case WARNING or ERROR has been detected. Events will be send if the custom event notifications for this utility have been configured. 

IBM Spectrum Scale includes a monitoring framework that allows to send custom events. Custom events can be surfaced by the IBM Spectrum Scale GUI and send as event notifications through SNMP or email. 

To integrate this utility with the IBM Spectrum Scale event monitoring framework custom events need to be defined and configured. The file [custom.json](custom.json) included predefined custom events. The following events are pre-defined:

| Event name | Event Code | Event description |
|------------|------------|-------------------|
| checkEE_info | 888341 | is send if all checks determined a good state |
| checkEE_warning | 888342 | is send if one or more component checks returned a WARNING state. Includes the component that has been checked and the WARNING message |
| checkEE_error | 888343 | is send if one or more component checks returned a ERROR state. Includes the component that has been checked and the ERROR message |


## Configure custom events
Before configuring the custom events check if a custom event file exists in /usr/lpp/mmfs/lib/mmsysmon/. If this is the case the [custom.json](custom.json) has to be added to the existing custom event file. Consider the json syntax.

It is recommended to place the custom.json file in /var/mmfs/mmsysmon/ and create a symlink to this file under /usr/lpp/mmfs/lib/mmsysmon/ in order to prevent the custom.json file to be discarded after code upgrades. 

After the custom.json file has been installed, restart the monitoring framework:

	# systemctl restart mmsysmon.service 

After the [custom.json](custom.json) has been configured, test if the custom events have been picked up, using the command:
 
	# mmhealth event show 888341

If this results in an error message, then investigate the problem. The system monitor provides log files in /var/adm/ras/mmmsysmon*. 

If the events are working then restart the Spectrum Scale GUI

	# systemctl restart gpfsgui


## Check all components,
This utility can check all IBM Spectrum Archive EE components using the `option -e`. If custom events have been configured then the utility will raise events for each WARNING and ERROR state. If no WARNING or ERROR state has been detected then this utility will send a single INFO event informing that the check has been performed and no issue has been found. 

To invoke the check of all components use the command: 

	# ./check_spectrumarchive.sh -e

The output if written to standard out. The return code of the utility correlates to the highest return code for each check. If event notificatios are enabled by installing the [custom.json](custom.json) then custom event are raised for checks resulting in WARNING and ERROR states.

Checking all componentes can be automated with the [IBM Spectrum Scale storage services automtation framework](https://github.com/nhaustein/SpectrumScaleAutomation). This framework can be configured to run the checks for all Spectrum Archive EE components periodically and raise events when appropriate. 


### Examples of events
Find below an example for the events raised to the Spectrum Scale system monitor:

No trouble found: 

	2019-12-02 11:42:49.706032 CET        checkee_info              INFO       Component all EE components ended successful, message: No problem found


Warning message:

	2019-12-02 06:27:36.207856 EST        checkee_warning           WARNING    Check check_pools determined Warnings, message: WARNING: Pool emtpypool has no tapes assigned (capacity=0);.


Error messages:

	2019-12-02 06:23:42.941351 EST        checkee_error             ERROR      Component check_status determined Errors, message: ERROR: EE is not running and node status not detected..

If the Spectrum Scale GUI is installed and active the events raised by this utility when using option -e are propagated to the GUI event log. Event notifications can be configured to send events via email or SNMP. The component scope is NODE. 


# Integration with Icinga
Icinga allows to monitor infrastructure and services. The Icinga architecture is client and server based. 


The server is the Icinga server providing the graphical user interface and the option to configure monitored objects such as hostgroups, hosts and services. The hosts to be monitored are the Spectrum Archive nodes. The services are checked with the check_spectrumarchive.sh script. The Icinga server essentially calls the script on the remote Spectrum Archive nodes using NRPE. More information about Icinga: https://Icinga.com/products/


The client is the IBM Spectrum Archive nodes being monitored. The communication between the server and the client can be based on Nagios Remote Plugin Executor (NRPE). This requires to install and configure NRPE on the Spectrum Archive nodes. 
[More information about NRPE](https://exchange.nagios.org/directory/Addons/Monitoring-Agents/NRPE--2D-Nagios-Remote-Plugin-Executor/details)



## Prepare the client (EE nodes)
In order to monitor the Spectrum Archive nodes using NRPE the NRPE packages and optionally the nagios-plugins have to be installed and configured. These packages need to be installed on all Spectrum Archive to be monitored. 


There are different ways to install NRPE and nagios plugins. Red Hat does not include these packages in the standard installation repository, but they can be downloaded from other sources (e.g. rpmfind). The following packages should be installed: 

	nrpe, nagios-common, nagios-plugin

An alternative way for installing NRPE and nagios-plugins can be found here: https://support.nagios.com/kb/article.php?id=8


After the installation of NRPE has finished, notice some important path and configuration files:
- NRPE configution file (NRPE.cfg), default location is /etc/nagios/NRPE.cfg
- nagios plugins (check_*), default location is /usr/lib64/nagios/plugins


Edit the NRPE configuration file (e.g /etc/nagios/NRPE.cfg) and set the include directory:

	include_dir=/etc/NRPE.d/


The check_spectrumarchive.sh script must be run with root privileges. NRPE however does not run as root but as a user that is defined in the NRPE.cfg file (NRPE_user, NRPE_group). The default user and group name is NRPE. Consequently sudo must be configured on the server to allow the NRPE-user to run the check_spectrumarchive.sh tool. To configure sudo, perform these steps:
1. In the NRPE-configuration file (/etc/nagios/NRPE.cfg) set command prefix to sudo:

		command_prefix=/usr/bin/sudo

2. Add the NRPE-user to the sudoer configuration:

		%NRPE          ALL=(ALL) NOPASSWD: /usr/local/bin/check_spectrumarchive.sh*,/usr/lib64/nagios/plugins/*


Now copy the executable script check_spectrumarchive.sh to /usr/local/bin


Switch to the NRPE-user and test if the script works under the sudo context:

	/usr/bin/sudo check_spectrumarchive.sh -s

Note, if you are not able to switch to the NRPE-user you may have to specify a login shell for the user (temporarily). 


Create the NRPE-configuration for the Spectrum Archive specific checks using this script. Note, the allowed_hosts must include the IP address of your Icinga server. Each check has a name given in [] which executes a particular command, such as /usr/local/bin/check_spectrumarchive.sh -s. Find an example below: 

	allowed_hosts=127.0.0.1,9.155.114.101
	command[check_users]=/usr/local/nagios/libexec/check_users -w 2 -c 5
	command[check_ee_state]=/usr/local/bin/check_spectrumarchive.sh -s
	command[check_ee_nodes]=/usr/local/bin/check_spectrumarchive.sh -n
	command[check_ee_tapes]=/usr/local/bin/check_spectrumarchive.sh -t
	command[check_ee_drives]=/usr/local/bin/check_spectrumarchive.sh -d
	command[check_ee_pools]=/usr/local/bin/check_spectrumarchive.sh -p 80
	command[check_ee_rtasks]=/usr/local/bin/check_spectrumarchive.sh -a r
	command[check_ee_ctasks]=/usr/local/bin/check_spectrumarchive.sh -a c

Find an example of the NRPE configuration [config_files/nrpe_eenode_local.cfg](config_files/nrpe_eenode_local.cfg). 


Now start and enable the NRPE service and check the status:

	# systemctl start NRPE
	# systemctl enable NRPE
	# systemctl status NRPE

Continue with the configuration of the monitored objects on the Icinga server. 



## Configure Icinga server
Assume the Icinga server is installed an configured. The default configuration of the Icinga server is located in /etc/Icinga. The default location for the object definition is in /etc/Icinga/objects.


First check that the Icinga server can communicate with the Spectrum Archive nodes using NRPE. For this purpose the check_NRPE plugin of the server can be used. The default location is: /usr/lib/nagios/plugins/check_NRPE. Find an example below:

	/usr/lib/nagios/plugins/check_NRPE -H <IP of Spectrum Archive node>

This command should return the NRPE version. If this is not the case investigate the problem. 


Likewise you can execute a remote check:

	/usr/lib/nagios/plugins/check_NRPE -H <IP of Spectrum Archive node> -c check_ee_state

This command should also return an appropriate response


If the NRPE communication and remote commands work then allow external commands by opening the Icinga configuration file (/etc/Icinga/Icinga.cfg) and adjust this setting:

	check_external_commands=1


Now configure the objects for the Spectrum Archive nodes. It is recommended to create a new file in directory /etc/Icinga/objects. In the example below two Spectrum Archive host (eenode1 and eenode2) are assigned to a host group (eenodes). For this host group a number of services are defined that within the define service stanza. Each service has a name, a host group where it is executed and a check command. The check command specifies a NRPE check and the name of the check that was configured in the NRPE-configuration of the client. For example the check_command check_NRPE!check_ee_state will execute the command /usr/local/bin/check_spectrumarchive.sh -s on the hosts. 

	define hostgroup {
		hostgroup_name  eenodes
		alias           EE Nodes
		members         eenode1,eenode2
		}

	define host {
		use                     generic-host
		host_name               eenode1
		alias                   EE Node 1
		address                 <ip of ee node 1>
		}

	define host {
		use                     generic-host
		host_name               eenode2
		alias                   EE Node 2
		address                 <ip of ee node 2>
		}

	define service {
		use                     generic-service
		hostgroup_name          eenodes
		service_description     Users logged on to the system
		check_command           check_NRPE!check_users
		}

	define service {
		use                     generic-service
		hostgroup_name          eenodes
		service_description     Check EE software state
		check_command           check_NRPE!check_ee_state
		}

	define service {
		use                     generic-service
		hostgroup_name          eenodes
		service_description     Check EE node state
		check_command           check_NRPE!check_ee_nodes
		}

	define service {
		use                     generic-service
		hostgroup_name          eenodes
		service_description     Check EE drive states
		check_command           check_NRPE!check_ee_drives
		}

	define service {
		use                     generic-service
		hostgroup_name          eenodes
		service_description     Check EE tape states
		check_command           check_NRPE!check_ee_tapes
		}

	define service {
		use                     generic-service
		hostgroup_name          eenodes
		service_description     Check EE pool state 
		check_command           check_NRPE!check_ee_pools
		}

	define service {
		use                     generic-service
		hostgroup_name          eenodes
		service_description     Check EE running tasks
		check_command           check_NRPE!check_ee_rtasks
		}

	define service {
		use                     generic-service
		hostgroup_name          eenodes
		service_description     Check EE completed tasks
		check_command           check_NRPE!check_ee_ctasks
		}


Find an example of the object defintion in [config_files/eenodes_icinga.cfg](config_files/eenodes_icinga.cfg). 

Once the object definition has been done and store in default object location /etc/Icinga/objects restart the Icinga process using systemctl or init.d. 


In the example above we have used the old Icinga 1 syntax to define objects. Icinga 2 comes with a new syntax with a similar semantic. More information about migrating from Icinga 1 syntax to Icinga 2 syntax: https://Icinga.com/docs/Icinga2/latest/doc/23-migrating-from-Icinga-1x/ 


Logon to the Icinga server GUI and check your host groups, hosts and services. The following is an example of the host view that was created based on the example above:

<img src="https://github.com/nhaustein/check_spectrumarchive/blob/master/images/icinga_ee2.JPG" width="868" alt="Icinga Spectrum Archive Integration">


Have run using it.
