# Introduction
Docker image with Owasp Zed Attack Proxy preinstalled.
# Details
## Install Instructions:
For the stable release:
```bash
docker pull owasp/zap2docker-stable
```
For the latest weekly release:
```bash
docker pull owasp/zap2docker-weekly
```
For the live release (built whenever the [zaproxy](https://github.com/zaproxy/zaproxy) project is changed):
```bash
docker pull owasp/zap2docker-live
```
For the bare release (a very small Docker image, contains only the necessary required dependencies to run ZAP, ideal for CI environments):
```bash
docker pull owasp/zap2docker-bare
```
The Dockerfiles can be found [here](https://github.com/zaproxy/zaproxy/tree/develop/docker).

### Healthcheck
The docker file now supports [healthcheck](https://docs.docker.com/engine/reference/builder/#healthcheck). The check uses the `zap-cli status` to check that ZAP completed loading. If you are running ZAP with port other than the default `8080`, you need to set the `ZAP_PORT` environment variable. Otherwise, the healthcheck will fail.

## Usage Instructions:
### ZAP GUI in a Browser:
Yes, you can run the ZAP Desktop GUI in a browser. You can use it in just the same way as the Swing UI and can even proxy via it.<br>
See the [WebSwing](WebSwing) wiki page for details.

### ZAP Headless:
You can also start the ZAP in headless mode with following command:
```bash
docker run -u zap -p 8080:8080 -i owasp/zap2docker-stable zap.sh -daemon -host 0.0.0.0 -port 8080 -config api.addrs.addr.name=.* -config api.addrs.addr.regex=true -config api.key=<api-key>
```
<sub>**Note**: `-config api.addrs.addr.name=.*` opens the API up for connections from any other host, it is prudent to configure this more specifically for your network/setup.</sub>

### ZAP Headless with xvfb:
You can start the ZAP in headless mode with xvfb following command:

```bash
docker run -u zap -p 8080:8080 -i owasp/zap2docker-stable zap-x.sh -daemon -host 0.0.0.0 -port 8080 -config api.addrs.addr.name=.* -config api.addrs.addr.regex=true
```
<sub>**Note**: `-config api.addrs.addr.name=.*` opens the API up for connections from any other host, it is prudent to configure this more specifically for your network/setup.</sub>

This first starts xvfb (X virtual frame buffer) which allows add-ons that use Selenium (like the Ajax Spider and DOM XSS scanner) to run in a headless environment. Firefox is also installed so can be used with these add-ons.

### ZAP Baseline Scan:
The [[ZAP Baseline Scan]] runs the ZAP spider against the specified target for (by default) 1 minute and then waits for the passive scanning to complete before reporting the results.

To run it with no 'file' params use:
```
docker run -t owasp/zap2docker-weekly zap-baseline.py -t https://www.example.com
```
If you use 'file' params then you need to mount the directory those file are in or will be generated in, eg
```
docker run -v $(pwd):/zap/wrk/:rw -t owasp/zap2docker-weekly zap-baseline.py \
    -t https://www.example.com -g gen.conf -r testreport.html
```

For more details see the [[ZAP Baseline Scan]] page.
### ZAP CLI:
[ZAP CLI](https://github.com/Grunny/zap-cli) is a ZAP wrapper written in Python. It provides a simple way to do scanning from the command line:

```bash
docker run -i owasp/zap2docker-stable zap-cli quick-scan --self-contained \
    --start-options '-config api.disablekey=true' http://target
```

### ZAPR:
Zapr is ruby script for ZAP which allows commandline active scanning for desired target:

```bash
docker run -u zap -i owasp/zap2docker-stable zapr --debug --summary http://target
```


## Accessing the API from outside of the Docker container

To access the API when ZAP is running as a daemon inside a container you need to perform the following steps

1. Obtain the external IP Address or hostname of the container
2. Configure the proxy settings of your browser to point to that address
3. Navigate to http://zap


If you just try to browse to the hostname of the container, Zap assumes that you are trying to proxy via it to that url, which is itself.

You will then get an error like the following appearing in the browser:

	Failed to read http://ExternalIPAddressOrHostNameOfContainer:8090/ within 20 seconds, 
	check to see if the site is available and if so consider adjusting ZAP's read time out in the Connection options panel.


#### 1. Obtain the IP Address or Hostname

Docker appears to assign 'random' IP addresses, so an approach that appears to work is:

Run ZAP as a daemon listening on "0.0.0.0":

```bash
docker run -p 8090:8090 -i owasp/zap2docker-stable zap.sh -daemon -port 8090 -host 0.0.0.0
```
Find out the container id:
```bash
docker ps
```
Find out which address has been assigned to it:
```bash
docker inspect <CONTAINER ID> | grep IPAddress
```
You should be then able to point your browser at the specified host/port and access the ZAP API, eg http://172.17.0.8:8090/

Note that on Macs the IP will be the IP of the Docker VM host.  This is accessible with:  
```bash
docker-machine ip <host>
```

#### 2. Configure the proxy settings

There are various ways to configure a browsers proxy settings.

Both Chrome and Firefox have addons that allow you to easily switch settings on the fly without interferring with your operating system settings.

#### 3. Navigate to http://zap from your browser

You should then be presented with the Welcome page and a link named 'Local API'
 

#### Using the API Automation Libraries

The automation libraries such as *zap-api-dotnet* work in the same manner. They proxy to external IP Address or hostname of the container and then interact with the API by using **http://zap** as the base address.


## Using the same Root CA Certificate between docker container instances  

Each time you issue the docker run command a new container is created from the image specified. This results in the zap daemon generating a new root certificate. In a test environment this would result in having to retrieve the public key certificate using the API and then install it on the machines that the test browsers are using each time the container is run.

By running the following command with the -certfulldump parameter you can save the auto generated certificate for future use. The \<directory\> would be a volume that you have mounted against the container.

    zap.sh -daemon -certfulldump <directory>/CertificatePrivateAndPublic.pem

Then on subsequent runs of the container you specify the previously generated certificate with the -certload parameter.

    zap.sh -daemon -certload <directory>/CertificatePrivateAndPublic.pem .... other parameters ....

To access the public key certificate just run the zap UI and starting from the menu navigate to the Tools -> Options -> Dynamic SSL Certificates option. Click the Import button and select the CertificatePrivateAndPublic.pem file. Then click the save button to save the certificate which contains just the public key. This certificate can then be installed on your test machines for use with the browsers.

### Exploring a owasp/zap2docker-stable container

An easy way to run the container without immediately launching the zap daemon is to use the tail command.

```bash
    docker run -u zap -p 8080:8080 -i owasp/zap2docker-stable tail -f /dev/null
```




Rather than specify the command that should be executed when the container is started 

docker run -it ......

Entrypoint -- 



### Scanning an app running on the host OS

IP addresses like localhost and 127.0.0.1 cannot be used to access an app running on the host OS from within a docker container.
To get around this you can use the following code to get an IP address that will work:
```bash
$(ip -f inet -o addr show docker0 | awk '{print $4}' | cut -d '/' -f 1)
```
For example:
```bash
docker run -t owasp/zap2docker-weekly zap-baseline.py -t http://$(ip -f inet -o addr show docker0 | awk '{print $4}' | cut -d '/' -f 1):10080
```