# Ptest2 for SHARK

The framework was initially designed for parallel hive testing.  Now
it is modified so that it can be used for shark qFile tests running in
parallel mode.  The changes were made in HostExecutor.java, bash script templates
batch-exec.vm and source-prep.vm were completely rewritten and
TestScripts.java was removed.  If you'd like to dive into the code, start with
org.apache.hive.ptest.RunTests.

# First you need to do all the preparations needed for parallel hive testing:

# Building

    mvn clean package

# Sizing

We utilize 8 servers for this process and trunk builds complete in 1 hour. Each slave
has 8 physical cores with hyperthreading enabled and 48GB of ram. Each slave is allowed
8 test "threads". I had used more than 8 threads but Zookeeper timed out randomly.

# Configuring

* Create a user such as hiveptest on the master and all slaves.
* Setup passwordless ssh form the master to all slaves.
* Ensure that SSH connection attempts won't fail.

On all slaves add the following to /etc/ssh/sshd_config:

    MaxAuthTries 100
    MaxSessions 100
    MaxStartups 100

# Install git, svn, make, patch, java, ant, and maven

Recent version of git, svn, make, patch, java, ant and maven should be installed. Additionally
environment variables such as MAVEN_OPTS and ANT_OPTS should be configured with large leap sizes:

    $ for item in java maven ant; do echo $item; cat /etc/profile.d/${item}.sh;done
    java
    export JAVA_HOME=$(readlink -f /usr/java/default)
    export PATH=$JAVA_HOME/bin:$PATH
    maven
    export M2_HOME=$(readlink -f /usr/local/apache-maven)
    export PATH=$M2_HOME/bin:$PATH
    export MAVEN_OPTS="-Xmx2g -XX:MaxPermSize=256M"
    ant
    export ANT_HOME=$(readlink -f /usr/local/apache-ant)
    export PATH=$ANT_HOME/bin:$PATH
    export ANT_OPTS="-Xmx1g -XX:MaxPermSize=256m"

# Ensure umask is setup to 0022

    $ cat /etc/profile.d/umask.sh 
    umask 0022

# On all slaves ensure that nproc and nofile are set to high values

# rm /etc/security/limits.d/90-nproc.conf

# cat /etc/security/limits.d/nproc.conf

* soft nproc 32768
* hard nproc 65536

root soft nproc 32768
root hard nproc 65536
# cat /etc/security/limits.d/nofile.conf 

* soft nofile 32768
* hard nofile 32768

root soft nofile 32768
root hard nofile 32768

# If using the Cloud Host Provider:

Ensure the user running the tests has strict host/key checking disabled:

   $ cat ~/.ssh/config
   StrictHostKeyChecking no
   ConnectTimeout 20
   ServerAliveInterval 1

# Additional steps specific for parallel shark testing:

# Configure properties file

See conf/example-apache-trunk.properties

-- unitTests.directories should be empty

-- Shark requires TEST_FILE that explicitly lists all tests to be run.
Ptest2, conversely, executes all tests by default and requires
excluded tests to be listed in exclude list.  Ptest2_shark follows
Ptest2 and also requires exlude list.  Here is an example of how to
convert TEST_FILE content to exclude list of property file:

# Prepare property file for ptest2, $TEST_FILE should be set
QFILES_DIR=run-tests-from-scratch-workspace/hive/ql/src/test/queries/clientpositive
PROPERTY_FILE=testutils/ptest2/apache-trunk.properties
TEMP_PROPERTY_FILE=testutils/ptest2/apache-trunk.properties.tmp
exclude_list_name='qFileTest.clientPositive.groups.minimr'
for f in $(ls $QFILES_DIR); do
        name=$(echo "$f" | cut -d'.' -f1)
        if ! grep -q "testCliDriver_$name\$" $TEST_FILE; then
                exclude_list+=" $f"
        fi
done
sed "s/$exclude_list_name *=.*/$exclude_list_name=$exclude_list/" $PROPERTY_FILE > $TEMP_PROPERTY_FILE
mv $TEMP_PROPERTY_FILE $PROPERTY_FILE
#End of Prepare property file

-- qFileTest.clientPositive.batchSize - sets the desired number of tests in one batch

-- avoid trailing whitespace in property file

-- avoid trailing slashes in path names in property file

# Ensure SCALA_HOME is specified in /etc/profile.d/scala.sh

# Execute

    mvn dependency:copy-dependencies
    java -Xms4g -Xmx4g -cp "target/hive-ptest-1.0-classes.jar:target/dependency/*" org.apache.hive.ptest.execution.PTest --properties apache-trunk.properties
