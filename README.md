GlusterFS Hadoop Plugin
=======================

INTRODUCTION
------------

This document describes how to use GlusterFS (http://www.gluster.org/) as a backing store with Hadoop.

This plugin replaces the hadoop file system (typically, the Hadoop Distributed File System) with the 
GlusterFileSystem, which writes to a local directory which FUSE mounts a proxy to a gluster system.

REQUIREMENTS
------------

* Supported OS is GNU/Linux
* GlusterFS installed on all machines in the cluster
* Java Runtime Environment (JRE)
* Maven 3x (needed if you are building the plugin from source)
* JDK 6+ (needed if you are building the plugin from source)

NOTE: Plugin relies on two *nix command line utilities to function properly. They are:

* mount: Used to mount GlusterFS volumes.
* getfattr: Used to fetch Extended Attributes of a file

Make sure they are installed on all hosts in the cluster and their locations are in $PATH
environment variable.


INSTALLATION
------------

** NOTE: Example below is for Hadoop version 0.20.2 ($GLUSTER_HOME/hdfs/0.20.2) **

* Building the plugin from source [Maven (http://maven.apache.org/) and JDK is required to build the plugin]

  Change to glusterfs-hadoop directory in the GlusterFS source tree and build the plugin.

  # cd $GLUSTER_HOME/hdfs/0.20.2
  # mvn package

  On a successful build the plugin will be present in the `target` directory.
  (NOTE: version number will be a part of the plugin)

  # ls target/
  classes  glusterfs-0.20.2-0.1.jar  maven-archiver  surefire-reports  test-classes

  Copy the plugin to lib/ directory in your $HADOOP_HOME dir.

  # cp target/glusterfs-0.20.2-0.1.jar $HADOOP_HOME/lib

  Copy the sample configuration file that ships with this source (conf/core-site.xml) to conf
  directory in your $HADOOP_HOME dir.

  # cp conf/core-site.xml $HADOOP_HOME/conf

* Installing the plugin from RPM

  See the plugin documentation for installing from RPM.


CLUSTER INSTALLATION
--------------------

  In case it is tedious to do the above steps(s) on all hosts in the cluster; use the build-and-deploy.py script to
  build the plugin in one place and deploy it (along with the configuration file on all other hosts).

  This should be run on the host which is that hadoop master [Job Tracker].

* STEPS (You would have done Step 1 and 2 anyway while deploying Hadoop)

  1. Edit conf/slaves file in your hadoop distribution; one line for each slave.
  2. Setup password-less ssh b/w hadoop master and slave(s).
  3. Edit conf/core-site.xml with all glusterfs related configurations (see CONFIGURATION)
  4. Run the following
     # cd $GLUSTER_HOME/hdfs/0.20.2/tools
     # python ./build-and-deploy.py -b -d /path/to/hadoop/home -c

     This will build the plugin and copy it (and the config file) to all slaves (mentioned in $HADOOP_HOME/conf/slaves).

   Script options:
     -b : build the plugin
     -d : location of hadoop directory
     -c : deploy core-site.xml
     -m : deploy mapred-site.xml
     -h : deploy hadoop-env.sh


CONFIGURATION
-------------

  All plugin configuration is done in a single XML file (core-site.xml) with <name><value> tags in each <property>
  block.

  Brief explanation of the tunables and the values they accept (change them where-ever needed) are mentioned below

  name:  fs.glusterfs.impl
  value: org.apache.hadoop.fs.glusterfs.GlusterFileSystem

         The default FileSystem API to use (there is little reason to modify this).

  name:  fs.default.name
  value: glusterfs:///

         The default name that hadoop uses to represent file as a URI (typically a server:port tuple). Use any host
         in the cluster as the server and any port number. This option has to be in server:port format for hadoop
         to create file URI; but is not used by plugin.

  name:  fs.glusterfs.volname
  value: volume-dist-rep

         The volume to mount.


  name:  fs.glusterfs.mount
  value: /mnt/glusterfs

         This is the directory where the gluster volume is mounted

  name:  fs.glusterfs.server
  value: localhost

         To mount a volume the plugin needs to know the hostname or the IP of a GlusterFS server in the cluster.
         Mention it here.

USAGE
-----

  Once configured, start Hadoop Map/Reduce daemons

  # cd $HADOOP_HOME
  # ./bin/start-mapred.sh

  If the map/reduce job/task trackers are up, all I/O will be done to GlusterFS.


FOR HACKERS
-----------

* Source Layout (./src/)

For the overall architecture, see.  Currently, we use the hadoop RawLocalFileSystem as 
the basis - and wrap it with the GlusterVolume class.  That class is then used by the 
Hadoop 1x (GlusterFileSystem) and Hadoop 2x (GlusterFs) adapters.

 https://forge.gluster.org/hadoop/pages/Architecture

./tools/build-deploy-jar.py                                                  <--- Build and Deployment Script
./conf/core-site.xml                                                         <--- Sample configuration file
./pom.xml                                                                    <--- build XML file (used by maven)

./COPYING                                                                    <--- License
./README                                                                     <--- This file



JENKINS
-------

  #Method 1) Modify JENKINS_USER in /etc/sysconfig/jenkins
  JENKINS_USER=root

  #Method 2) Directly modify /etc/init.d/jenkins 
  #daemon --user "$JENKINS_USER" --pidfile "$JENKINS_PID_FILE" $JAVA_CMD $PARAMS > /dev/null
  echo "WARNING: RUNNING AS ROOT" 
  daemon --user root --pidfile "$JENKINS_PID_FILE" $JAVA_CMD $PARAMS > /dev/null


BUILDING 
--------

Building requires a working gluster mount for unit tests. 
The unit tests read test resources from glusterconfig.properties - a file which should be present 

1) edit your .bashrc, or else at your terminal run : 

export GLUSTER_MOUNT=/mnt/glusterfs
export HCFS_FILE_SYSTEM_CONNECTOR=org.apache.hadoop.fs.test.connector.glusterfs.GlusterFileSystemTestConnector 
export HCFS_CLASSNAME=org.apache.hadoop.fs.glusterfs.GlusterFileSystem

(in eclipse - see below , you will add these at the "Run Configurations" menu,
in VM arguments, prefixed with -D, for example, "-DGLUSTER_MOUNT=x -DHCFS_FILE_SYSTEM_CONNECTOR=y ...")

2) run: 
   mvn clean package 
   
3) The jar artifact will be in target/

DEVELOPING
----------

0) Create a mock gluster mount: 
 
 #Create raw disk and format it...
 truncate -s 1G /export/debugging_fun.brick
 sudo mkfs.xfs  /export/debugging_fun.brick

 #Mount it as loopback fs
 mount -o loop /export/debugging_fun.brick /mnt/mybrick ;

 #Now make a mount point for it, and also, for gluster itself
 mkdir /mnt/mybrick/glusterbrick
 mkdir /mnt/glusterfs
 MNT="/mnt/glusterfs"
 BRICK="/mnt/mybrick/glusterbrick"
 
 #Create a gluster volume that writes to the brick
 sudo gluster volume create HadoopVol 10.10.61.230:$BRICK 

 #Mount the volume on top of the newly created brick
 mount -t glusterfs mount -t glusterfs $(hostname):HadoopVol $MNT

1) Run "mvn eclipse:eclipse", and import into eclipse.

2) Add the exported env variables above via Run Configurations as described in the above section.

3) Develop and run unit tests as you would any other java app.

VAGRANT
-------

You can use Vagrant to quickly get an development environment.

1) Install Vagrant.

2) Install vagrant-cachier plugin: vagrant plugin install vagrant-cachier

3) Run vagrant up
