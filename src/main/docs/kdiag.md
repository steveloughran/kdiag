<!---
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
  
   http://www.apache.org/licenses/LICENSE-2.0
  
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->



Troubleshooting Hadooop and Kerberos with `KDiag`
---------------------------

`KDiag` is the Kerberos Diagnostics entry point in Hadoop 2.8+.
It is designed to work standalone against older versions (indeed: this is how it was actually first used).
This is a standalone version, which you need to get on your Hadoop classpath
—after which the same set of commands as the built in one are supported 


Hadoop has a tool to aid validating setup: `KDiag`

It contains a series of probes for the JVM's configuration and the environment,
dumps out some system files (`/etc/krb5.conf`, `/etc/ntp.conf`), prints
out some system state and then attempts to log in to Kerberos as the current user,
or a specific principal in a named keytab.
 
The output of the command can be used for local diagnostics, or forwarded to
whoever supports the cluster.

The `KDiag` command has its own entry point; it is currently not hooked up
to the end-user CLI. 

It is invoked simply by passing its full classname to one of the `bin/hadoop`,
`bin/hdfs` or `bin/yarn` commands. Accordingly, it will display the kerberos client
state of the command used to invoke it.

```
hadoop jar kdiag-1.0.jar org.apache.hadoop.security.KDiag 
hdfs jar kdiag-1.0.jar org.apache.hadoop.security.KDiag
yarn jar kdiag-1.0.jar org.apache.hadoop.security.KDiag
```

The command returns a status code of 0 for a successful diagnostics run.
This does not imply that Kerberos is working —merely that the KDiag command
did not identify any problem from its limited set of probes. In particular,
as it does not attempt to connect to any remote service, it does not verify
that the client is trusted by any service.

If unsuccessful, exit codes are

* -1: the command failed for an unknown reason
* 41: Unauthorized (== HTTP's 401). KDiag detected a condition which causes
Kerberos to not work. Examine the output to identify the issue.

### Usage

```
KDiag: Diagnose Kerberos Problems
  [-D key=value] : Define a configuration option.
  [--jaas] : Require a JAAS file to be defined in java.security.auth.login.config.
  [--keylen <keylen>] : Require a minimum size for encryption keys supported by the JVM. Default value : 256.
  [--keytab <keytab> --principal <principal>] : Login from a keytab as a specific principal.
  [--nofail] : Do not fail on the first problem.
  [--nologin] : Do not attempt to log in.
  [--out <file>] : Write output to a file.
  [--resource <resource>] : Load an XML configuration resource.
  [--secure] : Require the hadoop configuration to be secure.
```

#### `--jaas`: Require a JAAS file to be defined in `java.security.auth.login.config`.

If `--jaas` is set, the Java system property `java.security.auth.login.config` must be
set to a JAAS file; this file must exist, be a simple file of non-zero bytes,
and readable by the current user. More detailed validation is not performed.

JAAS files are not needed by Hadoop itself, but some services (such as Zookeeper)
do require them for secure operation.

#### `--keylen <length>`: Require a minimum size for encryption keys supported by the JVM".

If the JVM does not support this length, the command will fail.

The default value is to 256, as needed for the `AES256` encryption scheme.
A JVM without the Java Cryptography Extensions installed does not support
such a key length. Kerberos will not work unless configured to use
an encryption scheme with a shorter key length.

#### `--keytab <keytab> --principal <principal>`: Log in from a keytab.

Log in from a keytab as the specific principal.

1. The file must contain the specific principal, including any named host.
That is, there is no mapping from `_HOST` to the current hostname.
1. KDiag will log out and attempt to log back in again. This catches
JVM compatibility problems which have existed in the past. (Hadoop's
Kerberos support requires use of/introspection into JVM-specific classes).

#### `--nofail` : Do not fail on the first problem

KDiag will make a best-effort attempt to diagnose all Kerberos problems,
rather than stop at the first one.

This is somewhat limited; checks are made in the order which problems
surface (e.g keylength is checked first), so an early failure can trigger
many more problems. But it does produce a more detailed report.

#### `--nologin`: Do not attempt to log in.

Skip trying to log in. This takes precedence over the `--keytab` option,
and also disables trying to log in to kerberos as the current kinited user.

This is useful when the KDiag command is being invoked within an application,
as it does not set up Hadoop's static security state —merely check for
some basic Kerberos preconditions.

#### `--out outfile`: Write output to file.

```
hadoop jar kdiag-1.0.jar org.apache.hadoop.security.KDiag --out out.txt
```

Much of the diagnostics information comes from the JRE (to `stderr`) and
from Log4j (to `stdout`).
To get all the output, it is best to redirect both these output streams
to the same file, and omit the `--out` option.

```
hadoop jar kdiag-1.0.jar org.apache.hadoop.security.KDiag --keytab zk.service.keytab --principal zookeeper/devix.example.org@REALM > out.txt 2>&1
```

Even there, the output of the two streams, emitted across multiple threads, can
be a bit confusing. It will get easier with practise. Looking at the thread
name in the Log4j output to distinguish background threads from the main thread
helps at the hadoop level, but doesn't assist in JVM-level logging.

#### `--resource <resource>` : XML configuration resource to load.

When using the `hdfs` and `yarn` commands, it is often useful to force
load the `hdfs-site.xml` and `yarn-site.xml` resource files, to pick up any Kerberos-related
configuration options therein.
The `core-default` and `core-site` XML resources are always loaded.

```
hdfs jar kdiag-1.0.jar org.apache.hadoop.security.KDiag --resource hbase-default.xml --resource hbase-site.xml
yarn jar kdiag-1.0.jar org.apache.hadoop.security.KDiag --resource yarn-default.xml --resource yarn-site.xml
```

For extra logging during the operation, set the logging and `HADOOP_JAAS_DEBUG` 
environment variable to the values listed in "Troubleshooting". The JVM 
options are automatically set in KDiag.

#### `--secure`: Fail if the command is not executed on a secure cluster.

That is: if the authentication mechanism of the cluster is explicitly
or implicitly set to "simple":

```xml
<property>
  <name>hadoop.security.authentication</name>
  <value>simple</value>
</property>
```

Needless to say, an application so configured cannot talk to a secure Hadoop cluster.

### Example

```
hdfs jar kdiag-1.0.jar org.apache.hadoop.security.KDiag \
  --nofail \
  --resource hbase-default.xml --resource hbase-site.xml \
  --keylen 1024 \
  --keytab zk.service.keytab --principal zookeeper/devix.example.org@REALM
```
 
This attempts to to perform all diagnostics without failing early, load in
the HDFS and YARN XML resources, require a minimum key length of 1024 bytes,
and log in as the principal `zookeeper/devix.example.org@REALM`, whose key must be in
the keytab `zk.service.keytab`

