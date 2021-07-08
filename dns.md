



### Zone File Resource Records

The primary component of a zone file is its resource records.

There are many types of zone file resource records. The following are used most frequently:

* A

This refers to the Address record, which specifies an IP address to assign to a name, as in this example:

```
<host> IN A <IP-address> 
```

If the *`<host>`* value is omitted, then an `A` record points to a default IP address for the top of the namespace. This system is the target for all non-FQDN requests.

Consider the following `A` record examples for the `example.com` zone file

```sh
server1	IN	A	10.0.1.3
		    IN	A	10.0.1.5
```

Requests for `example.com` are pointed to 10.0.1.3 or 10.0.1.5.

* CNAME

This refers to the Canonical Name record, which maps one name to another. This type of record can also be referred to as an *alias record*.

The next example tells `named` that any requests sent to the *`<alias-name>`* should point to the host, *`<real-name>`*. `CNAME` records are most commonly used to point to services that use a common naming scheme, such as `www` for Web servers.

```
<alias-name> IN CNAME <real-name> 
```

In the following example, an `A` record binds a hostname to an IP address, while a `CNAME` record points the commonly used `www` hostname to it.

```
server1 IN A 10.0.1.5
www IN CNAME server1
```

* MX

This refers to the Mail eXchange record, which tells where mail sent to a particular namespace controlled by this zone should go.

```
IN MX <preference-value> <email-server-name> 
```

Here, the *`<preference-value>`* allows numerical ranking of the email servers for a namespace, giving preference to some email systems over others. The `MX` resource record with the lowest *`<preference-value>`* is preferred over the others. However, multiple email servers can possess the same value to distribute email traffic evenly among them.

The *`<email-server-name>`* may be a hostname or FQDN.

```
IN MX 10 mail.example.com.
              IN MX 20 mail2.example.com.
```

In this example, the first `mail.example.com` email server is preferred to the `mail2.example.com` email server when receiving email destined for the `example.com` domain.

* NS

This refers to the NameServer record, which announces the authoritative nameservers for a particular zone.

The following illustrates the layout of an `NS` record:

```
IN NS <nameserver-name> 
```

Here, *`<nameserver-name>`* should be an FQDN.

Next, two nameservers are listed as authoritative for the domain. It is not important whether these nameservers are slaves or if one is a master; they are both still considered authoritative.

```
IN NS dns1.example.com.
              IN NS dns2.example.com.
```

* PTR

This refers to the PoinTeR record, which is designed to point to another part of the namespace.

`PTR` records are primarily used for reverse name resolution, as they point IP addresses back to a particular name. Refer to [Section 7.3.4, “Reverse Name Resolution Zone Files”](https://docs.fedoraproject.org/en-US/Fedora/12/html/Deployment_Guide/s2-bind-configuration-zone-reverse.html) for more examples of `PTR` records in use.

* SOA

This refers to the Start Of Authority resource record, which proclaims important authoritative information about a namespace to the nameserver.

Located after the directives, an `SOA` resource record is the first resource record in a zone file.

The following shows the basic structure of an `SOA` resource record:

```sh
@     IN     SOA    <primary-name-server> <hostmaster-email> (
	            <serial-number>
              <time-to-refresh>
              <time-to-retry>
              <time-to-expire>
              <minimum-TTL> )
```

The `@` symbol places the `$ORIGIN` directive (or the zone's name, if the `$ORIGIN` directive is not set) as the namespace being defined by this `SOA` resource record. The hostname of the primary nameserver that is authoritative for this domain is the *`<primary-name-server>`* directive, and the email of the person to contact about this namespace is the *`<hostmaster-email>`* directive.

The *`<serial-number>`* directive is a numerical value incremented every time the zone file is altered to indicate it is time for `named` to reload the zone. The *`<time-to-refresh>`* directive is the numerical value slave servers use to determine how long to wait before asking the master nameserver if any changes have been made to the zone. The *`<serial-number>`* directive is a numerical value used by the slave servers to determine if it is using outdated zone data and should therefore refresh it.

The *`<time-to-retry>`* directive is a numerical value used by slave servers to determine the length of time to wait before issuing a refresh request in the event that the master nameserver is not answering. If the master has not replied to a refresh request before the amount of time specified in the *`<time-to-expire>`* directive elapses, the slave servers stop responding as an authority for requests concerning that namespace.

In BIND 4 and 8, the *`<minimum-TTL>`* directive is the amount of time other nameservers cache the zone's information. However, in BIND 9, the *`<minimum-TTL>`* directive defines how long negative answers are cached for. Caching of negative answers can be set to a maximum of 3 hours (`3H`).

When configuring BIND, all times are specified in seconds. However, it is possible to use abbreviations when specifying units of time other than seconds, such as minutes (`M`), hours (`H`), days (`D`), and weeks (`W`). The table in [Table 7.1, “Seconds compared to other time units”](https://docs.fedoraproject.org/en-US/Fedora/12/html/Deployment_Guide/s3-bind-zone-rr.html#tb-bind-seconds) shows an amount of time in seconds and the equivalent time in another format.



| Seconds    | Other Time Units |
| :--------- | :--------------- |
| `60`       | `1M`             |
| `1800`     | `30M`            |
| `3600`     | `1H`             |
| `10800`    | `3H`             |
| `21600`    | `6H`             |
| `43200`    | `12H`            |
| `86400`    | `1D`             |
| `259200`   | `3D`             |
| `604800`   | `1W`             |
| `31536000` | `365D`           |

The following example illustrates the form an `SOA` resource record might take when it is populated with real values.

```sh
@     IN     SOA    dns1.example.com.     hostmaster.example.com. (
			2001062501 ; serial
			21600      ; refresh after 6 hours
			3600       ; retry after 1 hour
			604800     ; expire after 1 week
			86400 )    ; minimum TTL of 1 day
```



### Example Zone File

Seen individually, directives and resource records can be difficult to grasp. However, when placed together in a single file, they become easier to understand.

The following example shows a very basic zone file.

```
$ORIGIN example.com.
$TTL 86400
@	SOA	dns1.example.com.	hostmaster.example.com. (
		2001062501 ; serial
		21600      ; refresh after 6 hours
		3600       ; retry after 1 hour
		604800     ; expire after 1 week
		86400 )    ; minimum TTL of 1 day
;
;
	NS	dns1.example.com.
	NS	dns2.example.com.
dns1	A	10.0.1.1
	AAAA	aaaa:bbbb::1
dns2	A	10.0.1.2
	AAAA	aaaa:bbbb::2
;
;
@	MX	10	mail.example.com.
	MX	20	mail2.example.com.
mail	A	10.0.1.5
	AAAA	aaaa:bbbb::5
mail2	A	10.0.1.6
	AAAA	aaaa:bbbb::6
;
;
; This sample zone file illustrates sharing the same IP addresses for multiple services:
;
services	A	10.0.1.10
		AAAA	aaaa:bbbb::10
		A	10.0.1.11
		AAAA	aaaa:bbbb::11

ftp	CNAME	services.example.com.
www	CNAME	services.example.com.
;
;
```

In this example, standard directives and `SOA` values are used. The authoritative nameservers are set as `dns1.example.com` and `dns2.example.com`, which have `A` records that tie them to `10.0.1.1` and `10.0.1.2`, respectively.

The email servers configured with the `MX` records point to `mail` and `mail2` via `A` records. Since the `mail` and `mail2` names do not end in a trailing period (`.`), the `$ORIGIN` domain is placed after them, expanding them to `mail.example.com` and `mail2.example.com`. Through the related `A` resource records, their IP addresses can be determined.

Services available at the standard names, such as `www.example.com` (WWW), are pointed at the appropriate servers using a `CNAME` record.

This zone file would be called into service with a `zone` statement in the `named.conf` similar to the following:

```
zone "example.com" IN {
	type master;
	file "example.com.zone";
	allow-update { none; };
};
```



### Reverse Name Resolution Zone Files

A reverse name resolution zone file is used to translate an IP address in a particular namespace into an FQDN. It looks very similar to a standard zone file, except that `PTR` resource records are used to link the IP addresses to a fully qualified domain name.

The following illustrates the layout of a `PTR` record:

```
<last-IP-digit> IN PTR <FQDN-of-system> 
```

The *`<last-IP-digit>`* is the last number in an IP address which points to a particular system's FQDN.

In the following example, IP addresses `10.0.1.1` through `10.0.1.6` are pointed to corresponding FQDNs. It can be located in `/var/named/example.com.rr.zone`.

```
$ORIGIN 1.0.10.in-addr.arpa.
$TTL 86400
@	IN	SOA	dns1.example.com.	hostmaster.example.com. (
			2001062501 ; serial
			21600      ; refresh after 6 hours
			3600       ; retry after 1 hour
			604800     ; expire after 1 week
			86400 )    ; minimum TTL of 1 day
;
;
1	IN	PTR	dns1.example.com.
2	IN	PTR	dns2.example.com.
;
5	IN	PTR    server1.example.com.
6	IN	PTR    server2.example.com.
;
3	IN	PTR    ftp.example.com.
4	IN	PTR    ftp.example.com.
```

This zone file would be called into service with a `zone` statement in the `named.conf` file similar to the following:

```
zone "1.0.10.in-addr.arpa" IN {
	type master;
	file "example.com.rr.zone";
	allow-update { none; };
};
```

There is very little difference between this example and a standard `zone` statement, except for the zone name. Note that a reverse name resolution zone requires the first three blocks of the IP address reversed followed by `.in-addr.arpa`. This allows the single block of IP numbers used in the reverse name resolution zone file to be associated with the zone.