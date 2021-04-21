## simple-server container image

This image is built with ACE v11 Developer Edition and an MQ v9 client, and provides the most basic 
configuration only. The [Dockerfile](Dockerfile.simple-server) requires a build argument of LICENSE=accept 
in order to build successfully, and downloads the required software from IBM download sites.

Instructions are included at the top of the Dockerfile, but the essential commands are of the form
```
docker build -t simple-server --build-arg LICENSE=accept -f Dockerfile.simple-server .
docker run -p 7600:7600 -p 7800:7800 --rm -ti simple-server
```
at which point an ACE server will be running in the container, and can be connected to a toolkit, or
the webUI can be used to administer the server.

This image is provided as a way to ensure that the build-and-run process is working; it is not intended 
to be used in production.
