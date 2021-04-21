## simple-baked-evertyhing container image

This image is built with ACE v11 Developer Edition and an MQ v9 client, with a sample application unpacked
into the work directory plus an MQ policy and credentials. The [Dockerfile](Dockerfile.simple-baked-everything) 
requires several build arguments:
 - LICENSE=accept 
 - MQUSER=a valid MQ userid
 - MQPASS=a matching MQ password

Before building, it is usually necessary to customise the hostname and port in the 
[MQoC policy](eclipse-projects/MQOnCloudPolicies/MQoC.policyxml) to avoid connection errors on startup.

Note that this image will contain the credentials for the MQ user, and these may be visible in the resulting 
Docker image even though they appear to be used and discarded in the Docckerfile itself. This may be appropriate
for local testing, but it would be most unwise to publish the resulting image to a public registry on Dockerhub 
(for example).


Instructions are included at the top of the Dockerfile, but the essential commands are of the form
```
docker build -t simple-baked-everything --build-arg LICENSE=accept --build-arg MQUSER=user --build-arg MQPASS=pwd -f Dockerfile .
docker run -p 7600:7600 -p 7800:7800 --rm -ti simple-baked-everything
```
at which point it is possible to use curl (or other equivalent) to run the flows to ensure MQ connectivity 
has been successful; the flows listen on URLs /putFlow and /getFlow and behave as follows (using SYSTEM.DEFAULT.LOCAL.QUEUE):
```
[kenya:/Development/tdolby] curl http://localhost:7800/putFlow
{"result":"successfully put message"}
[kenya:/Development/tdolby] curl http://localhost:7800/getFlow
{"messageText":"This is an MQ message from putFlow"}
```

This image is provided as a way to ensure that the server in the container can connect to MQ; it is not intended 
to be used in production.

## Error scenarios

### Invalid hostname
Incorrect hostname or port configuration will lead to the initial connection failing with an 
MQ code of 2538 (MQRC_HOST_NOT_AVAILABLE):

```
.....2021-04-21 14:02:20.315236: .2021-04-21 14:02:20.315444: BIP1990I: Integration server 'ace-server' starting initialization; version '11.0.0.12' (64-bit) 
.................................node.js process.on('exit') event emitted with exitCode of 0
main checker destroyed before initialisation completed, latest stage was MQ default CCSID, pid 1, tid 1, attempted stage was 40 and 39 stages completed
2021-04-21 14:02:25.416076: BIP2116E: IBM App Connect Enterprise internal error: diagnostic information 'Fatal Error; exception thrown before initialisation completed', 'MQ default CCSID', '1', '1', '40', '39'. 
2021-04-21 14:02:25.416176: BIP2203E: An integration server has encountered a problem whilst starting. 
2021-04-21 14:02:25.416224: BIP2677E: Failed to make a client connection to queue manager 'MQoC' using hostname 'mqoc-419f.qm.eu-gb.mq.appdomainWRONGWRONG.cloud' on port '31175': MQCC=2; MQRC=2538. 
2021-04-21 14:02:25.416298: BIP1992I: Integration server 'ace-server' stopped. 
```
In this case, the credentials will not have been used, and may also be invalid, but the server never had a chance to find out.

### Invalid user/password
Incorrect user/password information will lead to the initial connection failing with an 
MQ code of 2035 (MQRC_NOT_AUTHORIZED):
```
.....2021-04-21 13:59:23.306262: .2021-04-21 13:59:23.306478: BIP1990I: Integration server 'ace-server' starting initialization; version '11.0.0.12' (64-bit) 
.................................node.js process.on('exit') event emitted with exitCode of 0
main checker destroyed before initialisation completed, latest stage was MQ default CCSID, pid 1, tid 1, attempted stage was 40 and 39 stages completed
2021-04-21 13:59:37.404816: BIP2116E: IBM App Connect Enterprise internal error: diagnostic information 'Fatal Error; exception thrown before initialisation completed', 'MQ default CCSID', '1', '1', '40', '39'. 
2021-04-21 13:59:37.404912: BIP2203E: An integration server has encountered a problem whilst starting. 
2021-04-21 13:59:37.404952: BIP2677E: Failed to make a client connection to queue manager 'MQoC' using hostname 'mqoc-419f.qm.eu-gb.mq.appdomain.cloud' on port '31175': MQCC=2; MQRC=2035. 
2021-04-21 13:59:37.405016: BIP1992I: Integration server 'ace-server' stopped. 
```
