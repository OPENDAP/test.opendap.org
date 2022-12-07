# test.opendap.org

This project contains all the assests need to recreate our
test.opendap.org system in which we run both Hyrax and Apache httpd services
in docker containers.

I created this so that we could better isolate the content we have added
to the docker host system utilized by test.opendap.org. The project 
contains configuration files, data, and other related content.

Also included is a shell script, hyraxctl, that can be used to control the
dockerized services.

## External Dependencies
1. **htdocs** - The _test.opendap.org_ site runs not only Hyrax, but also Apache
httpd. The files that httpd serves on test.opendap.org are not part of
this project (too big). They can be downloaded from:
`https://www.opendap.org/pub/too/htdocs.tgz` and unpacked onto the mount point 
for that Apache httpd data, typically `$TOO_HOME/httpd/htdocs`
2. 

## Maintenance 

1. New Hyrax images mean new UID & GID for the bes user. That means that the 
deployer needs to launch the hyrax docker image and inspect the _/etc/etc/passwd_
file in order to locate the bes user's entry. The bes UID and GID should be 
used to locate or create (as needed) a user on the docker host with the same 
UID and GID. Then the BES log directory, $HYRAX_HOME/log/bes should be assigned 
UID and GID of the bes user in the docker container.

2. New Apache httpd docker images will generate new ETags for assets served. 
This will cause the _libdap4/unit-tests/HTTPConnectTest_ to fail. The test _setup()_
method will need to be edited to adopt the new ETag. Note that running the test
in debug (_-d_) will reveal the new ETag for the asset in question.


# hyraxctl

The included shell script _hyraxctl_, can be used to control the
dockerized services. Running _hyraxctl_ with no parameters will cause it 
to print its usage statrment. The file can also be sourced into the current
shell and all the commands can then be utilized directly from the 
current commandline.

For more information, read the hyraxctl script: The functions are simple,
and the script is commented.

# Technical Notes

## Docker Volume Permissions

We use mounted volumes on the docker containers to:
1. Capture **Hyrax** and **httpd** logs onto the docker host. 
2. Inject configuration for both **Hyrax** and **httpd**
3. Inject data from our external EFS volume for Hyrax to serve.

Making sure the file system permissions for owner and
group are resolved between the docker container and the docker
host are crucial to getting this working correctly, if at all.

The _UID/GID_ of the processes in the container must match the _UID/GID_ that is
granted access to mounted volumes.

**Example:** _Configure test.opendap.org running Hyrax in Docker and mounting an EFS data volume onto the Docker host._

To make this work for the BES I made 3 changes to the docker host system.

1. I started the Hyrax docker image and checked its _/etc/passwd_ 
and found the entry for the BES user. I then went to the docker host 
/etc passwd file and checked for the presence of a use with the same 
UID (997 in this example). Since there was no such user I added the 
following to the _/etc/passwd_ file on the docker host system so that 
it would have a user matching the BES user in the docker container:

       bes:x:997:994:BES daemon:/home/ubuntu/hyrax/log/bes:/sbin/nologin

2. I repeated the same as 1. for groups checking `/etc/group` on the 
running Hyrax docker image, checking the docker host system and then
padding the following line to the /etc/group on the docker host:
 
       bes:x:994:

3. On the docker host I created the BES home/log directory, and assigned
ownership:

       mkdir -p /home/ubuntu/hyrax/log/bes
       sudo chown -R bes:bes /home/ubuntu/hyrax/log/bes

4. The volume invocation to make the mount:

       --volume /home/ubuntu/hyrax/log/bes:/var/log/bes 

**Principle**: _Each system has a user with UID 997 and GID 994 that
owns /home/ubuntu/hyrax/log/bes on the docker host system, and
/var/log/bes in the docker container._

**NB**: Currently Tomcat (and thus the OLFS) are running as the root user
in the docker container and so these permissions issues seem to be moot
for those logs. Here's a listing of the log directories after Hyrax was started:

    root@ip-172-31-26-77:~/hyrax ls -l log/*
    log/bes:
    total 4
    -rw-r--r-- 1 bes bes 420 Dec  1 17:06 bes.log
    
    log/olfs:
    total 4
    -rw-r----- 1 root root   0 Dec  1 16:58 AnonymousAccess.log
    -rw-r----- 1 root root   0 Dec  1 16:58 BESCommands.log
    -rw-r----- 1 root root   0 Dec  1 16:58 HyraxAccess.log
    -rw-r----- 1 root root 204 Dec  1 16:58 HyraxErrors.log
    
    log/tomcat:
    total 4
    -rw-r--r-- 1 root root 16 Dec  1 17:06 console.log

@TODO - Eventually, when the Tomcat app is running as the tomcat user, similar steps will need to be taken for it and tge OLFS.

## OLFS

Mounting just the OLFS logs can be accomplished with
this volume mount:

    --volume ${log_dir}/olfs:/usr/share/tomcat/webapps/opendap/WEB-INF/conf/logs \

However, if a localized OLFS configuration is desired this can be
acheived by mounting the desired source directory, say:

    olfs_conf_dir="/home/ubuntu/hyrax/olfs";

on to the docker host system at the well-known location /etc/olfs:

    --volume ${olfs_conf_dir}:/etc/olfs \

If the directory is empty (the preferred starting point) then at the
next startup of the Hyrax docker and this mount, the OLFS will 
automatically populate it with its default configuration. Since this is 
a volume mount this will now persist on the Docker host and can be 
localized for the deployment, and the configuration will persist between
new installations of the hyrax docker image.

## Apache httpd and Tomcat

Interfacing the hyrax docker container hyrax with Apache **httpd** 
running in different Docker container all within the same Docker engine. 
Both **Tomcat** and **httpd** require configuration modifications.

### Tomcat

1. Since we are going to connect to Apache httpd using AJP we
use a volume mount to inject a modified _server.xml_ file into Tomcat's 
conf directory. This _server.xml_ is special because it contains an AJP 
socket definition:
 
       <!-- Define an AJP 1.3 Connector on port 8009 -->
       <Connector
          protocol="AJP/1.3"
          address="0.0.0.0"
          port="8009"
          redirectPort="443"
          secret="ur-secret-phrase"
       />

2. The AJP connector typically operates on port 8009. When the Hyrax
docker container is launched we need to publish the 8009 socket:

       --publish 8009:8009 \

**QUESTION**: What happens if the Dockerfile doesn't EXPOSE 8009?*  
**ANSWER**: _It does not work. I changed the hyrax docker container to EXPOSE 8009._

3. We will run the Apache httpd in a separate Docker container in the
same docker engine.

**One might ask why?** **Why not run httpd on the Docker host system?**
**Answer**: _I tried that but I was unable to get the AJP connection 
to Hyrax in the Docker container sorted out._ 

### Apache httpd

To pull the official Apache httpd image from docker hub:

    docker pull httpd:latest

We inject three things into the Apache httpd image when we launch it:

1. **httpd.conf** - A modified _httpd.conf_ file that enables **AJP** 
using the "name" of the hyrax docker container for the hyrax hostname. In this example we called the container _hyrax_:

       docker run -d --name hyrax ... \

The additions to **httpd.conf** load the proxy_module and 
proxy_ajp_module modules, define the Proxy conditions and establish both
a forward and reverse Proxy for the Hyrax service:

      LoadModule proxy_module modules/mod_proxy.so
      LoadModule proxy_ajp_module modules/mod_proxy_ajp.so

      <Proxy *>
      AddDefaultCharset Off
      #    Require all granted
          Order deny,allow
          Allow from all
      </Proxy>
      
      ProxyTimeout 300
      
      
      ProxyPass /opendap ajp://hyrax:8009/opendap retry=0 secret=ur-secret-phrase
      ProxyPassReverse /opendap ajp://hyrax:8009/opendap retry=0
      
      ProxyPass /dap ajp://hyrax:8009/opendap retry=0 secret=ur-secret-phrase
      ProxyPassReverse /dap ajp://hyrax:8009/opendap retry=0
      
      
      Redirect 302 /data/nothing_is_here.html /data/httpd_catalog/READTHIS
      
**NB**: The snippet of httpd.conf included above may be out of date, 
check the httpd/httpd.conf file in this project for the most recent.

3. **htdocs** - This is all of the test data that _test.opendap.org_
serves via Apache http, It's not a large collection (~340MB) but it's
needed for testing. In the case of some files that are in use with 
dmr++ file (which may or may not be located in this collection on 
test.opendap.org) the Range Get capabilities of httpd are needed.

4. **expires.sh** - This is the lone file in 
http://test.opendap.org/cgi-bin/expires directory and 


* _**@TODO Is expires.sh needed?**_ _YES! It is used by libdap4 unit-tests._

