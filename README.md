# ace-remoteQM-policy
Non-production examples of remote default queue manager policies for ACE v11 servers; these examples are intended to illustrate 
how such policies are used and some common errors, but should not be viewed as best practice for actual deployments.

Container | Description | Use
| :--- | :--- | :--- |
[simple-server](README-simple-server.md) | Most basic with no MQ | Sanity check for image build before proceeding to MQ images
[simple-baked-everything](README-simple-baked-everything.md) | Baked-in MQ config | Local testing with MQ; contains credentials in image
[simple-fried-userpw](README-simple-fried-userpw.md) | Baked-in MQ config; fried user/pw | Local testing with MQ
[simple-fried-allconfig](README-simple-fried-allconfig.md) | No MQ details baked in | Local testing with MQ


## Remote default QM overview
ACE and predecessor products have always been able to use MQ, both for explicit messaging (MQ nodes in a flow) and implicit 
usage for state storage (Aggregate nodes, etc), and historically this required a local queue manager to be co-located with 
the ACE servers. IIB v10 enabled client links to remote queue managers for the explicit usage, with MQEndpoint policies for
the various MQ nodes; ACE v11 has taken this further by allowing a remote queue manager to be used by default, with no extra
configuration on the MQ nodes, and also allowing remote queue managers for Aggregate and other nodes.
 
Product docs at https://www.ibm.com/docs/en/app-connect/11.0.0?topic=mq-using-remote-default-queue-manager explain this in
more detail, and https://community.ibm.com/community/user/integration/viewdocument/when-does-ace-need-a-local-mq-serve also
describes when a local queue manager might still be needed. Note that this capability is currently (as of ACE 11.0.0.12) only
available for independent servers.

For the MQ nodes, the configuration is the same as it would be for a local default queue manager:

![MQInput node properties](mqinput-default-mq.png)
 
and although the properties say "Local queue manager" for historical reasons, the node will in fact use a remote default
queue manager (and will fail to start if no queue manager (either local or remote) is configured).
 
## server.conf.yaml example
Enabling a remote default queue manager requires setting remoteDefaultQueueManager in server.conf.yaml (could be set in 
overrides/server.conf.yaml also); this would involve changing the default of
```
#remoteDefaultQueueManager: ''  # Specify an MQEndpoint policy in the format {policy project}:policy
```
to something similar to this
```
remoteDefaultQueueManager: '{MQOnCloudPolicies}:MQoC'  # Specify an MQEndpoint policy in the format {policy project}:policy
```
which will cause the server to use the policy named "MQoC" in the "MQOnCloudPolicies" policy project. Note that this 
setting will not make use of the "policyProject" setting under the "Defaults" section of server.conf.yaml; it must be
qualified with a policy project in braces as shown and cannot be simply "MQoC".

## Policies as XML files on disk
MQEndpoint policies are XML files containing configuration data, and must be placed in a directory that has a policy.descriptor
file (see eclipse-projects/MQOnCloudPolicies in this repo), but they are not required to be deployed in a BAR file nor created
using a toolkit. As long as the format is correct, then they can be generated from scripts, from templates and the "sed" 
command, or any other mechanism which provides the correct data on disk before the server starts.

Some of the projects in this repo use a toolkit-created policy project with policies, and others create them either during 
the image build itself, or at container start time. BAR files do not have to be used anywhere in the process.
 
## Overrides versus run
Both the remoteDefaultQueueManager setting and the MQEndpoint policies themselves can be placed in the "overrides" directory
as well as (or instead of) the "run" directory. 

If policies are present in both directories, as shown here
```
/home/aceuser/ace-server/overrides/MQOnCloudPolicies/policy.descriptor
/home/aceuser/ace-server/overrides/MQOnCloudPolicies/MQoC.policyxml
/home/aceuser/ace-server/run/MQOnCloudPolicies/policy.descriptor
/home/aceuser/ace-server/run/MQOnCloudPolicies/MQoC.policyxml
```
then the overrides version is used (standard practice for ACE), and the same is true for the server.conf.yaml in overrides.

## Credentials
Some MQEndpoint policies rely on user/password information for connections to remote queue managers, and these can be
provided using mqsisetdbparms or mqsicredentials to create credentials that can be referenced from the "securityIdentity"
section of the policy. These credentials must be in place before the server starts, and cannot be added later.

Note that not all MQ connections require user/password information: TLS mutual authentication may be used instead, and
that would be configured using other mechanisms. This repo uses user/password for connections because it is one of the
easiest ways to use MQ on Cloud, which is itself one of the easiest ways to experiment with remote queue managers and ACE.

## Startup sequence
When using a remote default queue manager, the server will attempt to connect immediately on startup to determine the 
queue manager's CCSID setting, and will shut down if the connection attempt is unsuccessful. This process involves a
number of steps:

 - The server will attempt to find the named policy in overrides (e.g., "overrides/myProject/myPolicy.policyxml")
     - If that is not successful, the server will look in the run directory (e.g., "run/myProject/myPolicy.policyxml")
     - If neither of those policies can be read, then the server issues a BIP1388 message and exits.
 - Assuming the policy has been read, then the server attempts to read required credentials, and will exit if they are unavailable.
 - The server then calls MQCONNX to connect, passing the policy and credential information to MQ.
 - If the connection is successful, then the server reads the CCSID from the queue manager, and continues to start as normal.

The initial startup sequence is where most (if not all) errors in configuration will be encountered, and this can pose
some challenges in a container-based environment where the container exits if the server fails to start up. This repo 
contains various different approaches to configuration, with various error cases documented in (?? TODO create an errors page).

## Auto-shutdown
The server will shut down if the default queue manager becomes unavailable, regardless of whether the default queue manager
is local or remote. 
