# Keycloak Kerberos Demo

This repository contains the files for a Keycloak demo with Kerberos user federation towards two seperate KDCs.

[Video of this demo](https://youtu.be/j0v2Vz8KbeY)

## Prerequisits

This demo has been tested with podman 3.4.4 and podman-compose 1.0.6.

* Have podman and podman-compose installed.
* Have krb5 user tools installed.
    * On Debian derivates this is krb5-user
    * On CentOS derivates (RHEL)  this is krb5-workstation
* Have firefox installed

## Update hosts file 

Add the following to  `/etc/hosts` 

```
127.0.2.1   s1.example.com
127.0.2.2   s2.example.com
127.0.2.3   s3.example.com
```

## Start keycloak and kerberos service

```
podman-compose up
```

Check startup to see that all services has started up correctly.

## Keycloak setup

The provided keycloak realm, that is loaded on start up, contains a realm called "kerberostest" that has two Kerberos providers "kerberos-se" and "kerberos-dk".
They are ordered so that "kerberos-se" has higher priority.


## Setup client

Add, or update, `/etc/krb5.conf'

```
[libdefaults]
	default_realm = SE.EXAMPLE.COM

# The following krb5.conf variables are only for MIT Kerberos.
	kdc_timesync = 1
	ccache_type = 4
	forwardable = true
	proxiable = true

# The following encryption type specification will be used by MIT Kerberos
# if uncommented.  In general, the defaults in the MIT Kerberos code are
# correct and overriding these specifications only serves to disable new
# encryption types as they are added, creating interoperability problems.
#
# The only time when you might need to uncomment these lines and change
# the enctypes is if you have local software that will break on ticket
# caches containing ticket encryption types it doesn't know about (such as
# old versions of Sun Java).

#	default_tgs_enctypes = des3-hmac-sha1
#	default_tkt_enctypes = des3-hmac-sha1
#	permitted_enctypes = des3-hmac-sha1

# The following libdefaults parameters are only for Heimdal Kerberos.
	fcc-mit-ticketflags = true

[realms]
	DK.EXAMPLE.COM = {
		kdc = s1.example.com
		admin_server = s1.example.com
	}
 
    SE.EXAMPLE.COM = {
        kdc = s3.example.com
        admin_server = s3.example.com
    }
```

Start firefox and go to `about:config` add `.example.com` to the `network.negotiate-auth.trusted-uri` preference.

## Perform tests


First use the terminal login as sven in the SE.EXAMPLE.COM kerberos realm.

```
ponu@devws:~$ kinit sven@SE.EXAMPLE.COM
Password for sven@SE.EXAMPLE.COM: abc123
ponu@devws:~$ klist
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: sven@SE.EXAMPLE.COM

Valid starting       Expires              Service principal
2023-08-09 22:01:10  2023-08-10 10:01:10  krbtgt/SE.EXAMPLE.COM@SE.EXAMPLE.COM
	renew until 2023-08-10 22:01:08
```

Then access the Account console http://s2.example.com:8080/realms/kerberostest/account/ . Press sign-in

Validate that the sign in was successfull and sing out.

Now destroy the kerberos ticket 

```
ponu@devws:~$ kdestroy
```

Login as preben in the DK.EXAMPLE.COM kerberos realm

```
ponu@devws:~$ kinit preben@DK.EXAMPLE.COM
Password for preben@DK.EXAMPLE.COM: 
ponu@devws:~$ klist 
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: preben@DK.EXAMPLE.COM

Valid starting       Expires              Service principal
2023-08-09 22:03:24  2023-08-10 10:03:24  krbtgt/DK.EXAMPLE.COM@DK.EXAMPLE.COM
	renew until 2023-08-10 22:03:22
```

Once again access the Account console http://s2.example.com:8080/realms/kerberostest/account/ . Press sign-in.

This should login the user as preben, but due to the bug the login is not successfull.

## Test fix

Change the image used for the `keycloak` service in `docker-compose.yaml` to use `quay.io/ponu/keycloak:fix-for-two-kerberos-kdc`.  Then rerun the test scenario above. You will now be able to login both as sven@SE.EXAMPLE.COM and as preben@DK.EXAMPLE.COM.


## Clean up

* Destroy all kerberos tokens on the host
    * `kdestroy`
* Stop processes using `podman-compose down`
* Prune the system, to remove stoped containers
    * `podman system prune` 
* Remove the volumes (optional)
    * `podman volume prune`
* Remove the localhost/keycloak-with-two-kdc_kerberos-* images
    * `podman rmi localhost/keycloak-with-two-kdc_kerberos-se:latest`
    * `podman rmi localhost/keycloak-with-two-kdc_kerberos-dk:latest`

## Acknowledgements

This demo was produced using inspiration from Rishabh Singh blog [Kerberos integration with Keycloak](https://medium.com/@rishabhsvats/red-hat-single-sign-on-integration-with-kerberos-user-federation-f9c9e757ace).

The `docker-krb5-server` container is build on Gabriel Abdalla Cavalcante [docker-krb5-server](https://github.com/gcavalcante8808/docker-krb5-server) project.
