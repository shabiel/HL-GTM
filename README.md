# HL7 and HLO for GT.M/YottaDB
This codebase provides HL7 support for GT.M and YottaDB. There is one small
change in the regular HL7 package, and many modifications for HLO. I am
grateful to Lloyd Milligan for doing the initial port for HLO. This work builds
upon his initial work, and adds some new features for the port.

The aim of the port is to provide all the features of the original work. As
such, there is not much to document here, besides how to set it up on GT.M/
YottaDB; and how to test that it is working.

HLO has two manuals, which you should consult for actual HLO usage and
operations. The information supplied here is limited to getting HLO running on
GT.M/YottaDB.

 * [HLO System Manager Manual](https://www.va.gov/vdl/documents/Infrastructure/Health_Level_7_%28HL7%29/hlo_system_manager_manual.pdf)
 * [HLO Developer Manual](https://www.va.gov/vdl/documents/Infrastructure/Health_Level_7_%28HL7%29/vms_hlo_developer_manual.pdf)

## Installation
[Download and install the KIDS build located in releases](https://github.com/shabiel/HL-GTM/releases/download/HL-1.6-10001/HL_1p6_10001.KID).
Patch XU\*8.0\*10005 is required, as it provides `$$ENV^%ZOSV`. Install this
and the preceeding Kernel patches by going to
https://github.com/shabiel/Kernel-GTM.

## Usage on GT.M/YottaDB
Set-up an xinetd service like the following. As usual with any xinetd service,
make sure you put a valid user and server, and make sure that the server has
the execute bit set (`chmod +x`).

```
service vehu-vista-hlo
{
  port = 5026
  socket_type = stream
  protocol = tcp
  type = UNLISTED
  user = vehu
  server = /home/vehu/bin/hlo.sh
  wait = no
  disable = no
}
```

The server just needs to run `GTMLNX^HLOSRVR` or `LINUX1^HLOSRVR` (the former
was written for compatibility with other xinetd services; the latter is what
Cache uses). Here's a sample hlo.sh. Remember to fix the paths.

```sh
#!/bin/bash
#
#  This is a file to run VistA HLO as a Linux Service
#
export HOME=/home/vehu
export REMOTE_HOST=`echo $REMOTE_HOST | sed 's/::ffff://'`
source $HOME/etc/env

LOG=$HOME/log/hlo.log
${gtm_dist}/mumps -run GTMLNX^HLOSRVR                          2>>  ${LOG}
```

## Some testing hints
The KIDS build includes the routines HLODEM1 and HLODEM5 from the HLO Developer
Manual plus the HLO Config that allows these to work. These are good learning
examples for how to use HLO.

In order to test HLO, you can do the following:

### Start HLO Background processes.
First, allocate keys `HLOMAIN` and `HLOMGR` to your VistA User.

Navigate to EVE > HL7 Main Menu > HL7 (Optimized) MAIN MENU > HLO SYSTEM
MONITOR. On the Listman screen, type "START HLO" to start the HLO system.

### Testing the Listener
Send the following HL7 message:

```
MSH|^~\&|HLO DEMO SENDING APPLICATION|999|HLO DEMO RECEIVING APPLICATION|050|20120303105447||ADT^A08|43042.3|T|2.4|||AL|NE
PID|1||3333^^^USVHA&&0363^NI~517509835^^^USSSA&&0363^SS||HOFFMAN^JOHN^^^^||19470605||||4876 25TH AVE.^^SAN FRANCISCO^CALIFORNIA^94111^^^
NK1|1|DOE^JOHN^^^^|^|^^^^^^^|||EP^EMERGENCY CONTACT PERSON^0131
```

You should get the following ack:

```
MSH|^~\&|HLO DEMO RECEIVING APPLICATION|500^HL7.VEHU.DOMAIN.GOV^DNS|HLO DEMO SENDING APPLICATION|999^^|20200403204631-0500||ACK|500 100000000003|T|2.4|||NE|NE
MSA|CA|43042.3||
```

This confirms that the listener is working.

### Testing the Client
First, you need to fix the Logical Link to tell where your client to send your
data. I, for example, set-up Mirth to listen on a machine on 6661, and so
I will be using the IP address of the machine and the port 6661 to send the
message. Setting up the Logical Link is done under the regular HL7 Main Menu, 
in "Filer and Link Management Options > Link Edit". Unlike the HL7 1.6 system,
you do not need to shutdown the link before editing it. Go to the TCP part of
the page, and fill in the fields TCP/IP ADDRESS and TCP/IP PORT (OPTIMIZED).
You will also need to put in an INSTITUTION on the first page.

Once you do that, run this code from Direct Mode to send a message. Replace the
number in the first parameter with a valid DFN:

```
W $$A08^HLODEM1(1)
```

You should see following message received on the receiving end:

```
MSH|^~\&|HLO DEMO SENDING APPLICATION|050^HL7.VISTA.DOMAIN.EXT:5026^DNS|HLO DEMO RECEIVING APPLICATION|500^:6661^DNS|20200408134151-0500||ADT^A08^|050 47|T^|2.4|||AL|NE|
PID|1||500000001^^^USVHA&&0363^NI~323554567^^^USSSA&&0363^SS||TWENTYFOUR^PATIENT^^^^||19480302||||^^^^^^^
NK1|1|^^^^^|^|^^^^^^^|||EP^EMERGENCY CONTACT PERSON^0131
```
