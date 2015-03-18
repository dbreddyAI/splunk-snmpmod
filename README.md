SnmpMod
=======

Splunk SNMP Modular Input

This project was originally based on [SplunkModularInputsPythonFramework](https://github.com/damiendallimore/SplunkModularInputsPythonFramework).
I have taken the SNMP modular input, refactored the python code to be more re-usable and added an extra stanza for polling interfaces.

Deployment
==========

    cd $SPLUNK_HOME/etc/apps
    git clone https://github.com/oxo42/snmpmod.git
    cd snmpmod
    mkdir local
	vim local/inputs.conf

SNMP v3
-------
If you are using SNMP version 3 , you have to obtain, build and add the [PyCrypto](https://www.dlitz.net/software/pycrypto/) package yourself :

The simplest way is to build pycrypto and drop the `Crypto` directory in `$SPLUNK_HOME/etc/apps/snmpmod/bin`. I don't recommend installing the pycrypto package to the Splunk Python runtime's site-packages, this could have unforeseen side effects.

### Building and installing PyCrypto

The pycrypto module is not bundled with the core release, because :

* you need to build it for each separate platform
* US export controls for encrypted software


So, here are a few instructions for building and installing pycrypto yourself :

* Download the pycrypto package from [https://www.dlitz.net/software/pycrypto/](https://www.dlitz.net/software/pycrypto/)
* Then run these 3 commands (note : you will need to use a System python 2.7 runtime , not the Splunk python runtime)

    # From Python 2.7.9 onwards pip is included, in which case
    pip2 install pycrypto
    
    # For older versions of Python run the following
    python setup.py build
    python setup.py install
    python setup.py test

* Browse to where the Crypto module was installed to ie: `/usr/local/lib/python2.7/dist-packages/Crypto`
* Copy the `Crypto` directory to `$SPLUNK_HOME/etc/apps/snmpmod/bin`

snmp Stanza
===========

	[snmp://host]
	destination = host
	object_names = 1.3.6.1.2.1.2.2.1.1.35,1.3.6.1.2.1.2.2.1.2.35,1.3.6.1.2.1.2.2.1.3.35,1.3.6.1.2.1.2.2.1.4.35,1.3.6.1.2.1.2.2.1.5.35,1.3.6.1.2.1.2.2.1.6.35,1.3.6.1.2.1.2.2.1.7.35,1.3.6.1.2.1.2.2.1.8.35,1.3.6.1.2.1.2.2.1.9.35,1.3.6.1.2.1.2.2.1.10.35,1.3.6.1.2.1.2.2.1.11.35,1.3.6.1.2.1.2.2.1.12.35,1.3.6.1.2.1.2.2.1.13.35,1.3.6.1.2.1.2.2.1.14.35,1.3.6.1.2.1.2.2.1.15.35,1.3.6.1.2.1.2.2.1.16.35,1.3.6.1.2.1.2.2.1.17.35,1.3.6.1.2.1.2.2.1.18.35,1.3.6.1.2.1.2.2.1.19.35,1.3.6.1.2.1.2.2.1.20.35,1.3.6.1.2.1.2.2.1.21.35,1.3.6.1.2.1.2.2.1.22.35
	mib_names = IF-MIB
	snmp_version = 3
	v3_securityName = username
	v3_authKey = password
	snmpinterval = 300
	index = main
	# Must be snmp_ta for the extracts to work
	sourcetype = snmp_ta
	disabled = 1

snmpif Stanza
=============

    [snmpif://hostname]
    destination = hostname
    snmp_version = 3
    v3_securityName = username
    v3_authKey = password
    snmpinterval = 300
    interfaces = 1,5,8,9
    index = network
	# The sourcetype can be whatever you want
    sourcetype = snmpif


Response Handlers
=================

destination, host and /etc/hosts
--------------------------------
Currently, all response handlers set the Splunk host to the value of destination.  If you don't have DNS (bad sysadmin!) add an entry to /etc/hosts.  I'd be very happy to take a pull request that will look at a `host` config option and override `destination` with that value.

SNMP Interface Search Query
===========================

I strongly recommend you [create a search macro](http://docs.splunk.com/Documentation/Splunk/latest/Search/Usesearchmacros) `snmpif_parse` that uses `streamstats` to calculate the bits per second from the raw `snmpif` data. My macro is:

    stats first(*) as * by _time host ifIndex 
    | streamstats window=2 global=false current=true range(if*Octets) as delta*, range(_time) as secs by host, ifIndex 
    | where secs>0 
    | eval bpsIn=coalesce(deltaHCIn, deltaIn)*8/secs 
    | eval bpsOut=coalesce(deltaHCOut, deltaOut)*8/secs 
    | eval mbpsIn=bpsIn/1000000 | eval mbpsOut=bpsOut/1000000

Then to call it and display the results as a graph:

    index=snmpif host=foo ifIndex=17 | `snmpif_parse` | timechart bins=500 avg(mbpsIn) as "Mbps IN", avg(mbpsOut) as "Mbps OUT"

And calculate 95th percentile figures

    index=snmpif host=foo ifIndex=17 | `snmpif_parse` | stats perc95(mbpsIn) as "IN", perc95(mbpsOut) as "OUT"

Summary Collection
==================

The search term shown above is quite expensive.  I am running the query above and collecting the data into a new index.

    [search index=network sourcetype=snmp_traffic | stats first(_time) as earliest] index=network sourcetype="snmpif" 
    | stats first(*) as * by _time host ifIndex 
    | streamstats window=2 global=false current=true range(if*Octets) as delta*, range(_time) as secs by host, ifIndex 
    | where secs>0 
    | eval bpsIn=coalesce(deltaHCIn, deltaIn)*8/secs 
    | eval bpsOut=coalesce(deltaHCOut, deltaOut)*8/secs 
    | eval mbpsIn=bpsIn/1000000 
    | eval mbpsOut=bpsOut/1000000 
    | fields _time host ifIndex bpsIn bpsOut ifAdminStatus ifDescr ifMtu ifOperStatus ifPhysAddress ifSpecific ifSpeed ifType mbpsIn mbpsOut 
    | collect index=network sourcetype=snmp_traffic

There is a trick there of using the most recent snmp_traffic event to start the next round of collections.  I run this search every 30 minutes.
