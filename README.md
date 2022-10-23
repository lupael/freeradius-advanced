# FreeRADIUS Advanced Use Cases

# Contents

[Introduction](#introduction)

[Installation](#installation)

[Expiration](#expiration)

[Changing IP Pool for Expired Username](#changing-ip-pool-for-expired-username)

[Expiration After Certain Connection Time](#expiration-after-certain-connection-time)

[Wrong Password Notification](#wrong-password-notification)

[User MAC Binding](#user-mac-binding)

[User Binding with NAS](#user-binding-with-nas)

[Restrict Service Type](#restrict-service-type)

[SQLCounter](#sqlcounter)

[User Disconnection from RADIUS](#user-disconnection-from-radius)

[pfSense PPPoE Server MPD Bandwidth Rate Limiting](#pfsense-pppoe-server-mpd-bandwidth-rate-limiting)

# Introduction

This document presents the configurations related to some advanced

FreeRADIUS use cases.

# Installation

*Install MySQL*

```

apt-get install -y mysql-server

```

*Install FreeRADIUS*

```

apt-get install freeradius freeradius-mysql freeradius-utils

```

*Setup MySQL*

Open MySQL:

```

mysql

```

Enter the following commands:

```

create database radius;

CREATE USER '<username>'@'localhost' IDENTIFIED BY '<password>';

GRANT ALL PRIVILEGES ON radius.* TO '<username>'@'localhost';

FLUSH PRIVILEGES;

exit

```

*Import MySQL Schema*

```

mysql radius < /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql

```

*Setup FreeRADIUS-MySQL Integration*

Open SQL configuration file:

```

nano /etc/freeradius/3.0/mods-available/sql

```

Set the following options:

```

dialect = "mysql"

```

Comment:

```

# driver = "rlm_sql_null"

```

Uncomment:

```

driver = "rlm_sql_${dialect}"

```

Comment out `tls` section in `mysql` section:

```

mysql {

                # If any of the files below are set, TLS encryption is enabled

#               tls {

#                       ca_file = "/etc/ssl/certs/my_ca.crt"

#                       ca_path = "/etc/ssl/certs/"

#                       certificate_file = "/etc/ssl/certs/private/client.crt"

#                       private_key_file = "/etc/ssl/certs/private/client.key"

#                       cipher = "DHE-RSA-AES256-SHA:AES128-SHA"

#

#                       tls_required = yes

#                       tls_check_cert = no

#                       tls_check_cert_cn = no

#               }

```

Uncomment and add login credentials created earlier:

```

server = "localhost"

port = 3306

login = "<username>"

password = "<password>"

```

Uncomment:

```

read_clients = yes

```

Save and exit.

*Enable SQL Module*

```

ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/

```

*Set Permissions*

```

chgrp -h freerad /etc/freeradius/3.0/mods-available/sql

chown -R freerad:freerad /etc/freeradius/3.0/mods-enabled/sql

```

*Restart FreeRADIUS Service*

```

systemctl restart freeradius

```

*Create Dummy User*

Open MySQL:

```

mysql

```

Add user:

```

use radius;

insert into radcheck (username,attribute,op,value) values("fredf", "Cleartext-Password", ":=", "wilma");

```

*Authentication Test Using radtest*

```

radtest fredf wilma localhost 0 testing123

```

You should receive `Access-Accept` in response.

# Expiration

The Expiration attribute is entered in `radcheck` table as follows:

<img width="611" alt="image2" src="https://user-images.githubusercontent.com/38311694/83622383-5443b180-a5a9-11ea-985f-d7e0c2ac6e99.png">

This means that this username will expire on 25 Apr 2020 at 10:42:00.

Session-Timeout is given to user based on this.

Note: `expiration` keyword needs to be present in authorize section in

`/etc/freeradius/3.0/sites-enabled/default` (it is present by default in

FreeRADIUS 3.x).

# Changing IP Pool for Expired Username

Add the following code in authorize section of

`/etc/freeradius/3.0/sites-enabled/default`:

```

expiration{

	userlock = 1

}

if(userlock){

	# Let him connect with EXPIRED pool in reply

	ok

	update reply {

	Reply-Message := "Your account has expired, %{User-Name} / Reason: DATE LIMIT REACHED"

	Framed-Pool := "Expired"

	}

}

```

Testing results:

<img width="689" alt="image3" src="https://user-images.githubusercontent.com/38311694/83622543-940a9900-a5a9-11ea-83f1-f84c64ff1c08.png">

<img width="878" alt="image4" src="https://user-images.githubusercontent.com/38311694/83622579-9d940100-a5a9-11ea-960c-3489f6a6bd4a.png">

# Expiration After Certain Connection Time

Suppose a user has paid for a certain amount of time and their username

needs to be expired after that much time online. We can use the

`Expire-After` attribute for that:

<img width="355" alt="image5" src="https://user-images.githubusercontent.com/38311694/83622591-a1c01e80-a5a9-11ea-8d45-d6f92ab0aa04.png">

# Wrong Password Notification

Enter the following code in `Post-Auth-Type REJECT` section of

`/etc/freeradius/3.0/sites-enabled/default`:

```

update reply {

	Reply-Message = 'Wrong Password'

}

```

<img width="491" alt="image6" src="https://user-images.githubusercontent.com/38311694/83622614-aab0f000-a5a9-11ea-89bd-76f60f2c7e17.png">

# User MAC Binding

A new table is created to store username to MAC address bindings:

```

CREATE TABLE IF NOT EXISTS `macs` (

	`id` int unsigned NOT NULL AUTO_INCREMENT,

	`username` varchar(100) COLLATE utf8mb4_unicode_ci NOT NULL,

	`callingstationid` varchar(50) COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT 'na',

	PRIMARY KEY (`id`)

);

```

The following code is added to authorize section for MAC addition and

authorization. When a username is initially created it has the MAC

address set to `na`. Upon first connection the MAC address is saved in

`macs` table, and subsequently it is checked.

```

update control{

	#check if callingstationid does not exist in mac_limit Table

	Tmp-String-0 = "%{sql: SELECT callingstationid FROM macs WHERE username = '%{User-Name}'}"

	#check if mac is available or not in mac_limit table

	Tmp-Integer-2 = "%{sql: SELECT count(*) FROM macs WHERE username = '%{User-Name}' AND callingstationid='%{Calling-Station-Id}'}"

}

if(&control:Tmp-String-0 == "na"){

	"%{sql: UPDATE macs SET callingstationid = '%{Calling-Station-Id}' WHERE Username = '%{User-Name}'}"

	update reply {

		Reply-Message := "New mac Added"

	}

}

elsif(&control:Tmp-Integer-2 == 0){

	update reply {

		Reply-Message += "MAC auth Failed"

	}

	reject

}

```

# User Binding with NAS

A new table is created to store huntgroups:

```

CREATE TABLE radhuntgroup (

    id int(11) unsigned NOT NULL auto_increment,

    groupname varchar(64) NOT NULL default '',

    nasipaddress varchar(15) NOT NULL default '',

    nasportid varchar(15) default NULL,

    PRIMARY KEY  (id),

    KEY nasipaddress (nasipaddress)

) ;

```

NAS IP addresses are assigned to various huntgroups:

<img width="253" alt="image7" src="https://user-images.githubusercontent.com/38311694/83622621-ac7ab380-a5a9-11ea-8b48-bea96048c130.png">

Users are assigned to usergroups:

<img width="245" alt="image8" src="https://user-images.githubusercontent.com/38311694/83622623-ad134a00-a5a9-11ea-8ceb-c0652cc403f1.png">

The following code is added in authorize section after preprocess. This

code adds the Huntgroup-Name attribute to RADIUS Access Request. Then,

it allows users of usergroup `hg1users` to only be connected on NASs

belonging to `hg1`. Similarly, users of usergroup `usergroup2` are only

allowed to be connected to NASs belonging to `hg2`.

```

update request{

	Huntgroup-Name := "%{sql:SELECT groupname FROM radhuntgroup WHERE nasipaddress='%{NAS-IP-Address}'}"

}

# bind hg1users usergroup to hg1 huntgroup

if (SQL-Group == "hg1users") {

	if (Huntgroup-Name == "hg1") {

		#ok

	}

	else {

		update reply {

			Reply-Message += "Error: User not allowed connection on this device"

		}		

		reject

	}

}

# bind hg2users usergroup to hg2 huntgroup

elsif (SQL-Group == "hg2users") {

	if (Huntgroup-Name == "hg2") {

		#ok

	}

	else {

		update reply {

			Reply-Message += "Error: User not allowed connection on this device"

		}	

		reject

	}

}

```

[Reference](https://wiki.freeradius.org/guide/SQL-Huntgroup-HOWTO)

# Restrict Service Type

Suppose we only want to allow requests to RADIUS of the following

Service-Type:

1.  Framed

2.  Login

And we want to reject all other requests. The following code should be

added to authorize section:

```

#Only allow service types "Framed-User" or "Login-User"; reject all others

if((&request:Service-Type=="Framed-User") || (&request:Service-Type=="Login-User")){

	#ok

}

else{

	update reply {

		Reply-Message += "Wrong service type"

	}

	reject

}

```

# SQLCounter

***Time Based Quota***

We can use `Session-Timeout` attribute to limit session time of a user.

For example, if we wanted to limit a user to only 30 minutes of network

access daily we could set Session-Timeout to 1800 (1800 s = 30 min).

This will ensure that the user automatically gets disconnected after 30

min. But the problem with this approach is that the user can connect

again to get 30 more minutes. To solve this problem, FreeRADIUS has some

pre-defined counters that can be used to assign time-based session

limits (like daily, monthly, etc).

The file `mods-enabled/sqlcounter` contains several default counters. This

is the configuration of daily counter:

```

sqlcounter dailycounter {

        sql_module_instance = sql

        dialect = mysql

        counter_name = Daily-Session-Time

        check_name = Max-Daily-Session

        reply_name = Session-Timeout

        key = User-Name

        reset = daily

        $INCLUDE ${modconfdir}/sql/counter/${dialect}/${.:instance}.conf

}

```

This counter checks for an internal attribute `Max-Daily-Session` and uses

the session data in DB to calculate the remaining session time of the

user for that day. It then adds that remaining `Session-Timeout` to RADIUS

Access Accept packet.

We need to add `dailycounter` to `sites-available/default` in authorize

section. In DB add the `Max-Daily-Session` attribute to radcheck table:

<img width="354" alt="image9" src="https://user-images.githubusercontent.com/38311694/83622628-adabe080-a5a9-11ea-88bf-4af48a89f213.png">

Enable the `sqlcounter` module:

```

cd /etc/freeradius/3.0/mods-enabled

ln -s ../mods-available/sqlcounter

```

***Volume Based Quota***

We can use `sqlcounter` module to query accounting data in DB and impose

volume-based limits. Suppose we want to allow a user to have 25 MB of

volume daily.

In the file `mods-enabled/sqlcounter` create a new counter:

```

sqlcounter total_volume {

        sql_module_instance = sql

        dialect = sql

        counter_name = Max-Total-Volume

        check_name = Max-Volume

        key = User-Name

        reset = daily

        query = "SELECT SUM(acctinputoctets) + SUM(acctoutputoctets) FROM radacct WHERE UserName='%{${key}}'"

}

```

Register the counter `total\_volume` in authorize section in

`sites-available/default`. Add the `Max-Volume` attribute for that user in

bytes in `radcheck` table:

<img width="351" alt="image10" src="https://user-images.githubusercontent.com/38311694/83622631-aedd0d80-a5a9-11ea-8325-206ba5c069b3.png">

Once the user has exceeded their volume limit they will not be

authorized on their next connection attempt. Please note that users who

are already connected will not be affected by this restriction. To

perform any action on such already-connected sessions that have gone

over their volume limit some script can be written that periodically

queries the DB and if it finds some user has used up their data limit it

performs some action on them (like disconnection, rate-limiting etc).

# User Disconnection from RADIUS

To disconnect a user from RADIUS we use the user’s username and IP

address and secret of NAS in radclient to disconnect that user:

```

root@ubuntu:~# echo 'User-Name = test1' | radclient -x 192.168.100.35:3799 disconnect ''1234''

Sent Disconnect-Request Id 28 from 0.0.0.0:35685 to 192.168.100.35:3799 length 27

	User-Name = "test1"

Received Disconnect-ACK Id 28 from 192.168.100.35:3799 to 0.0.0.0:0 length 30

	NAS-Identifier = "MikroTik"

```

***Enabling incoming messages in Mikrotik NAS***

<img width="485" alt="image11" src="https://user-images.githubusercontent.com/38311694/83622636-af75a400-a5a9-11ea-859b-0362c7196d96.png">

# pfSense PPPoE Server MPD Bandwidth Rate Limiting

[Reference](https://forum.netgate.com/topic/141034/rate-limit-on-radius-reply-attributes-for-pppoe-connections-not-working/2)

[MPD RADIUS Documentation](http://mpd.sourceforge.net/doc5/mpd30.html)

1. Create `dictionary.mpd` in `/usr/share/freeradius/`:

```

# dictionary.mpd                                                                                   

                                                                                                   

VENDOR          mpd             12341                                                              

                                                                                                   

BEGIN-VENDOR	mpd

ATTRIBUTE	mpd-rule	1	string

ATTRIBUTE	mpd-pipe	2	string

ATTRIBUTE	mpd-queue	3	string

ATTRIBUTE	mpd-table	4	string

ATTRIBUTE	mpd-table-static	5	string

ATTRIBUTE	mpd-filter	6	string

ATTRIBUTE	mpd-limit	7	string

ATTRIBUTE	mpd-input-octets	8	string

ATTRIBUTE	mpd-input-packets	9	string

ATTRIBUTE	mpd-output-octets	10	string

ATTRIBUTE	mpd-output-packets	11	string

ATTRIBUTE	mpd-link	12	string

ATTRIBUTE	mpd-bundle	13	string

ATTRIBUTE	mpd-iface	14	string

ATTRIBUTE	mpd-iface-index	15	integer

ATTRIBUTE	mpd-input-acct	16	string

ATTRIBUTE	mpd-output-acct	17	string

ATTRIBUTE	mpd-action	18	string

ATTRIBUTE	mpd-peer-ident	19	string

ATTRIBUTE	mpd-iface-name	20	string

ATTRIBUTE	mpd-iface-descr	21	string

ATTRIBUTE	mpd-iface-group	22	string

ATTRIBUTE	mpd-drop-user	154	integer

END-VENDOR	mpd

```

2. Add link to `dictionary.mpd` in `/usr/share/freeradius/dictionary`:

```

$INCLUDE dictionary.mpd

```

3. Add rate limit to user using `mpd-limit` attribute in the following way:

```

steve  Cleartext-Password := "testing"

        mpd-limit = "in#1=all rate-limit 3000000",

        mpd-limit = "out#1=all rate-limit 3000000"

```
