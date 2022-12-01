# GSSAPI Authentication - PostgreSQL

GSSAPI is an industry-standard protocol for secure authentication defined in RFC 2743. PostgreSQL supports GSSAPI for authentication, communications encryption, or both. GSSAPI provides automatic authentication (single sign-on) for systems that support it. The authentication itself is secure. If GSSAPI encryption or SSL encryption is used, the data sent along the database connection will be encrypted. Otherwise, it will not.

## Configuration of Kerberos

Kerberos is a network *authentication protocol* used to verify the identity of two or more *trusted hosts* across an *untrusted network*. It uses *secret-key cryptography* and a *trusted third party* (Kerberos Key Distribution Center) for authenticating client-server applications. Key Distribution Cente (KDC) gives clients tickets representing their network credentials. The Kerberos ticket is presented to the servers after the connection has been established.

![How does Kerberos work ?](https://www.datasunrise.com/wp-content/uploads/2016/11/Kerberos.png)


*Note: Since Kerberos protocol has a timestamp involved, all three machines clocks need to be synchronized.*

### Network configuration

We will need three ***ubuntu*** machines inside of VMware:
* A client machine
* A KDC machine
* A Postgres machine

Virtual machines have a NAT adapter. This is my network configuration

* Client machine: **192.168.152.166**
* Postgres machine: **192.168.152.165**
* KDC machine: **192.168.152.164**


Now we wil define the hostname for each machine :

* KDC machine       
      `hostnamectl set-hostname kdc.tekup.com`  <br>
![Change KDC machine hostname](/projet_kerberos/kdc/1.PNG)
  
* Postgres machine        
      `hostnamectl set-hostname pg.tekup.com`  <br>
![Change Postgres machine hostname](/projet_kerberos/pg/1.PNG)
  
* Client machine      
      `hostnamectl set-hostname client.tekup.com`
      <br>
![Change Client machine hostname](/projet_kerberos/client/1.PNG)
 


So now we come to the next point, which is DNS host mapping through the directory /etc/hosts. <br> 
  `sudo vi /etc/hosts`
  
The content of /etc/hosts should have the same hierarchy shown in the figure above.


![DNS Host mapping](/projet_kerberos/1.PNG)

### Configuration of Key Distribution Center Machine 

 Those are the packages that we need to install it on the KDC machine : <br>
 ```
    $ sudo apt-get update
    $ sudo apt-get install krb5-kdc krb5-admin-server krb5-config
 ```
 
KDC settings

 * the realm : 'TEKUP.COM' (must be *all uppercase*)
![Realm](/projet_kerberos/kdc/2.PNG)

 * the Kerberos server : 'kdc.tekup.com'
![Kerberos server](/projet_kerberos/kdc/3.PNG)

 * the administrative server : 'kdc.tekup.com'
![Administrative server](/projet_kerberos/kdc/4.PNG)
 

The master key for this KDC database needs to be set once the installation is complete :
   
```
sudo krb5_newrealm
```
![Master key](/projet_kerberos/kdc/5.PNG)

*The users and services in a realm are defined as a **principal** in Kerberos.* These principals are managed by an *admin user* that we need to create manually :

```
    $ sudo kadmin.local
    kadmin.local:  add_principal root/admin
```

![Create Kerberos admin user](/projet_kerberos/kdc/6.PNG)

[kadmin.local](https://web.mit.edu/kerberos/krb5-1.12/doc/admin/admin_commands/kadmin_local.html) is a KDC database administration program. We used this tool to create a new principal in the TEKUP.COM realm (`add_principal`).

We can check if the user *root/admin* was successfully created by running the command : `kadmin.local: list_principals`. We should see the 'root/admin@TEKUP.COM' principal listed along with other default principals.


![Check admin principal](/projet_kerberos/kdc/7.PNG)


Next, we need to grant all access rights to the Kerberos database to admin principal *root/admin* using the configuration file */etc/krb5kdc/kadm5.acl* . <br>
 `sudo vi /etc/krb5kdc/kadm5.acl`

In this file, we need to add the following line :

    */admin@TEKUP.COM   *

![Grant all priviliges to admin](/projet_kerberos/kdc/8.PNG)

For changes to take effect, we need to restart the following service : `sudo service krb5-admin-server restart`

Once the admin user who manages principals is created, we need to create the principals. We will to create principals for both the client machine and the service server machine.

**Create a principal related to the client**

```
    $ sudo kadmin.local
    kadmin.local:  add_principal sejda
```

![Create a principal for the client](/projet_kerberos/kdc/9.PNG)


### Configuration of Postgres machine

#### Configuration of Kerberos client

##### Installation of Packages  

Following are the packages that need to be installed on the Service server machine : <br>

```
$ sudo apt-get update
$ sudo apt-get install krb5-user libpam-krb5 libpam-ccreds
```

##### Preparation of the *keytab file*

We need to extract the service principal from KDC principal database to a keytab file.

1. In the KDC machine run the following command to generate the keytab file in the current folder :

```
   $ ktutil 
   ktutil:  add_entry -password -p postgres/pg.tekup.com@TEKUP.COM -k 1 -e aes256-cts-hmac-sha1-96
   Password for postgres/pg.tekup.com@TEKUP.COM: 
   ktutil:  wkt postgres.keytab
```
![generate keytab file](/projet_kerberos/pg/6.PNG)

2. Send the keytab file from the KDC machine to Postgres machine :

In the Postgres machine, we have create the following directories :
   
   `mkdir -p /home/ubuntu/pgsql/data`

In the KDC machine send the keytab file to the Postgres server :

   `scp postgres.keytab ubuntu@<PG_SERVER_IP_ADDRESS>:/home/ubuntu/pgsql/data`

   

![Send keytab file](/projet_kerberos/kdc/11.PNG)

3. Verify that the service principal was succesfully extracted from the KDC database :

    1. List the current keylist 

        `ktutil:  list`

    2. Read a krb5 keytab into the current keylist

        `ktutil:  read_kt pgsql/data/postgres.keytab`

    3. List the current keylist again

        `ktutil:  list`


![Verify keytab file extraction](/projet_kerberos/pg/7.PNG)

#### Configuration of the service (PostgreSQL)

##### Installation of PostgreSQL 

1. Update the package lists 
 
   `sudo apt-get update`
 
2. Install necessary packages for Postgres

   `sudo apt-get install postgresql postgresql-contrib`

3. Ensure that the service is started

   `sudo systemctl start postgresql`


![Verify posgresql](/projet_kerberos/pg/8.PNG)

##### Create a Postgres Role for the Client 

We will need to :

* create a new role for the client
    
    `create user sejda with encrypted password 'some_password';`

* create a new database

    `create database sejda;`


![Create postgres role for the client](/projet_kerberos/pg/10.PNG)

* grant all privileges on this database to the new role

    `grant all privileges on database sejda to sejda;`
<br>
![grant](/projet_kerberos/pg/11.PNG)


To ensure the role was successfully created run the following command :

```
postgres=# SELECT usename FROM pg_user WHERE usename LIKE 'sejda';
```

![Check client role](/projet_kerberos/pg/12.PNG)


The client sejda has now a role in Postgres and can access its database 'sejda'.


##### Update Postgres Configuration files (*postgresql.conf* and *pg_hba.conf* )

* Updating *postgresql.conf*

To edit the file run the following command :

`sudo nano /etc/postgresql/12/main/postgresql.conf`

By default, Postgres Server only allows connections from localhost. Since the client will connect to the Postgres server remotely, we will need to modify *postgresql.conf* so that Postgres Server allows connection from the network :

```
listen_addresses = '*'
```

 We will also need to specify the keytab file location :
 ```
 krb_server_keyfile = '/home/ubuntu/pgsql/data/postgres.keytab'
 ```

![Changes made to postgresql.conf file](/projet_kerberos/pg/13.PNG)


* Updating *pg_hba.conf*

HBA stands for host-based authentication. *pg_hba.conf* is the file used to control clients authentication in PostgreSQL. It is basically a set of records. Each record specifies a **connection type**, a **client IP address range**, a **database name**, a **user name**, and the **authentication method** to be used for connections matching these parameters.

The first field of each record specifies the **type of the connection attempt** made. It can take the following values :

* `local` : Connection attempts using *Unix-domain sockets* will be matched. 
* `host` : Connection attempts using *TCP/IP* will be matched (SSL or non-SSL as well as GSSAPI encrypted or non-GSSAPI encrypted connection attempts).
* `hostgssenc` : Connection attempts using *TCP/IP* will be matched, but only when the connection is made with GSSAPI encryption.


Some of the possible choices for the **authentication method** field are the following :

* `trust` : Allow the connection unconditionally. 
* `reject` : Reject the connection unconditionally. 
* `md5` : Perform SCRAM-SHA-256 or MD5 authentication to verify the user's password.
* `gss` : Use GSSAPI to authenticate the user. 
* `peer` : Obtain the client's operating system user name from the operating system and check if it matches the requested database user name. This is only available for *local connections*.

So to allow the user 'sejda' to connect remotely using  Kerberos we will add the following line :

```
# IPv4 local connections:
hostgssenc   sejda    sejda          <IP_ADDRESS_RANGE>         gss include_realm=0 krb_realm=TEKUP.COM
```

And comment other connections over TCP/IP.


![Changes made to pg_hba.conf file](/projet_kerberos/pg/14.PNG)

### Configuration of Client Machine 

Following are the packages that need to be installed on the Client machine : <br>

```
$ sudo apt-get update
$ sudo apt-get install krb5-user libpam-krb5 libpam-ccreds
```
 
## User Authentication

Once the setup is complete, it's time for the client to authenticate using kerberos.

First, try to connect to PostgreSQL remotely :

`$ psql -d sejda -h pg.tekup.com -U sejda`

![Connect before TGT](/projet_kerberos/client/6.PNG)

*-d* specifies the database, *-U* specifies the postgres role and *-h* specifies the ip address of the machine hosting postgres.

In the client machine check the cached credentials :

`$ klist`

Then initial the user authentication :

`$ kinit sejda

And check the ticket granting ticket (TGT) :

`$ klist`

![TGT](/projet_kerberos/client/7.PNG)

Now try to connect once again :

![Connect after TGT](/projet_kerberos/client/9.PNG)


Now you can check the service ticket :
`$ klist`

![Service ticket](/projet_kerberos/client/8.PNG)


