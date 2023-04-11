[![made-with-Go](https://img.shields.io/badge/Made%20with-Go-orange)](http://golang.org) [![proc-arch](https://img.shields.io/badge/Arch-x86%20%7C%20AMD64%20%7C%20ARM5%20%7C%20ARM7-blue)](http://golang.org) [![os](https://img.shields.io/badge/OS-Linux%20%7C%20Windows%20%7C%20Darwin-yellowgreen)](http://golang.org)


# Web interface for sending Wake-on-lan (magic packet)

A GoLang based HTTP server which will send a Wake-on-lan package (magic packet) on local network. The request can be send using web interface or directly using HTTP request with mapped device name in the URL. The only computing device I have running 24x7 is handy-dandy Raspberry Pi 4 (4gb) with docker containers. All other devices like server, laptop and NAS as powered only when I need them. I needed a way to easily turn them on specifically when trying to automate things like nightly builds.

I use this application behind NGINX web proxy which is secured with HTTPS certificate. It has no authentication, but it is home network and the reason I built this was to have no authentication. I have the same functionality provided in my home router, but I have to login and go through several clicks. Also, this app runs as docker image so even if it is hacked, it reduces the attack surface.

I have bookmarked direct link to device(s) on my browsers to wake them using single HTTP call for ease of access.

Things I use this for:
- to wake-up my home laptop remotely. I use my home laptop remotely over RDP.
- this is also helpful in building routines which will wake up my server and nightly builds and when it is all done, go back to sleep. I don't keep my home lab running 24x7 as it is a waste of energy.
- to turn on my NAS and laptop to start the weekly backup from laptop to NAS.
- to turn on NAS quickly when we are watching movies stored on NAS.

> It was tricky to configure the wol feature on my Dell Laptop. NAS and Dell servers were easy to configure. Follow this article for [Dell laptop](https://www.dell.com/support/article/en-us/sln305365/how-to-setup-wake-on-lan-wol-on-your-dell-system?lang=en) 

## Bootstrap UI with JS Grid for editing data

![Screenshot](wolweb_ui.png)

The UI features CRUD operation implemented using [js-grid.com](https://github.com/tabalinas/jsgrid) plugin. 

### Wake-up directly using HTTP Request

/wolweb/wake/**&lt;hostname&gt;** -  Returns a JSON object

```json
{
  "success":true,
  "message":"Sent magic packet to device Server with Mac 34:E6:D7:33:12:71 on Broadcast IP 192.168.1.255:9",
  "error":null
}
```

## Configure the app

The application will use the following default values if they are not explicitly configured as explained in sections below.

| Config | Description | Default
| --- | --- | --- |
| Port | Define the port on which the webserver will listen | **8089**
| Virtual Directory | A virtual directory to mount this application under | **/wolweb**
| Broadcast IP and Port | This is broadcast IP address and port for the local network. *Please include the port :9* | **192.168.1.255:9**

You can override the default application configuration by using a config file or by setting environment variables. The application will first load values from config file and look for environment variables and overwrites values from the file with the values which were found in the environment.

**Using config.json:**

```json
{
    "port": 8089,
    "vdir":"/wolweb",
    "bcastip":"192.168.1.255:9"
}
```
**Using Environment Variables:**

*Environment variables takes precedence over values in config.json file.*

| Variable Name | Description
| --- | --- |
| WOLWEBPORT | Override for default HTTP port
| WOLWEBVDIR | Override for default virtual directory
| WOLWEBBCASTIP | Override for broadcast IP address and port

## Devices (targets) - devices.json format
```json
{
    "devices": [
        {
            "name": "Server",
            "mac": "34:E6:D7:33:12:71",
            "ip": "192.168.1.255:9"
        },
        {
            "name": "NAS",
            "mac": "28:C6:8E:36:DC:38",
            "ip": "192.168.1.255:9"
        },
        {
            "name": "Laptop",
            "mac": "18:1D:EA:70:A0:21",
            "ip": "192.168.1.255:9"
        }
    ]
}

```
## Building for Raspberry Pi 3+

+ **Install Go:**
```
sudo apt-get install golang
```

+ **Clone this github:**
```
git clone https://github.com/xatornet/wolweb-rpi
cd wolweb-rpi
```

+ **Config go to add required packages:**
```
go mod init wolweb
go get -d github.com/gorilla/handlers
go get -d github.com/gorilla/mux
go get -d github.com/ilyakaznacheev/cleanenv
```

+ **Start compilation:**
```
GOOS="linux" GOARCH="arm64" go build -o wolweb .
```

+ **Give permission:**

If everything went alright, you should now be able to execute wolweb like this:
```
chmod +x wolweb
```

## Executing Wolweb manually

Execute directly with: 
```
./wolweb
```

Or add it as service, to your liking. 

+ You should be able to access it with http://YOUR-RPI-IP:8089/wolweb/ 

## Executing Wolweb as a service

This tutorial assumes that your RPI uses "pi" as user, and that wolweb-rpi folder is inside home.

+ **Create service file:** 
```
sudo nano /lib/systemd/system/wolweb.service
```

+ **Paste service config:**
```
[Unit]
Description=WolWeb-Server
After=network.target

[Service]
ExecStart=/home/pi/wolweb-rpi/wolweb
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```

Press CTRL+X to save the file.

+ **Enable Wolweb service:** 
```
sudo systemctl enable wolweb.service
```

Now, restart your Raspberry Pi and Wolweb should be working when it boots at http://YOUR-RPI-IP:8089/wolweb/ 

## NGiNX Config

I am already using NGiNX as web-proxy for accessing multiple services (web interfaces) from single IP and port 443 using free Let's Encrypt HTTPS certificate. For accessing this service, I just added the following configuration under my existing server node.
```
	location /wolweb {
		proxy_pass http://192.168.1.4:8089/wolweb;
	}
```
> This is also the reason why I have an option in this application to use virtual directory **/wolweb** as I can easily map all requests for this application. My / is already occupied for other web application in my network.

## Credits
Thank you to David Baumann's project https://github.com/dabondi/go-rest-wol for providing the framework which I modified a little to work within constraints of environment.
