# Creating HTTP Service using Golang (Quick Start)

Golang is a statically types, compilled programming language designed by Google, by Robert Griesemer, Rob Pike, and Ken Thompson (who designed and implemented UNIX operating system in Bell Labs). Golang adopted c-style syntax and provides many features over C language, such as garbage collection, memory safety, and structural typing. 

During a phase of creating POC, I asked team member to create HTTP service using Golang to test and measure samll operation. He wasn't Golang developer, hence it took sometime from him to figure out the cycle, from installing Language service, SDK, setting up development environment and deploy the service to the Test Servers. 

I wrote this quick start guide for developers who are new to Golang and want to get a brief of the development cycle of a running HTTP service.

# Summary
1. Setup development environment.
2. Write a samll program in Golang.
3. Build (compile) program into executable binary.
4. Move binary to production server / machine
5. Configure binary as a service

## 1. Setup development environment
This article assumes using Ubuntu as a development environment (works also for WSL2).  To set up development environment we will need to install Golang binary files, extract it, and setting paths.

### Downloading Go binary files
You can use `curl` to download binary files from official Golang download page
```bash
curl -O https://storage.googleapis.com/golang/go1.18.1.linux-amd64.tar.gz
```
`go1.18.1.linux-amd64.tar.gz` is the latest stable release at time of writing this article. You can replace it by any other release. Check the official download page for list of releases [Downloads - The Go Programming Language](https://go.dev/dl/)

### Extracting tar.gz
You can use `tar` to unpack the tarball package 
```bash
tar -xvf go1.13.5.linux-amd64.tar.gz
sudo mv go /usr/local
```

### Setting Path
You will need to set paths to access go binaries. Setting path is done by editing `.profile` file located in user home directory. You can use any editor, for me, I prefer `vim`
```bash
sudo vim ~/.profile
```

then add the required path to the end of file
```
export GOPATH=$HOME/work
export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin
```

Then you need to refresh your local session, either by logoff and logon, or by using `source`
```bash
source ~/.profile
```

To test your installtion and settings, check Go veriosn, if everything is ok, Go version will be displayed on screen
```bash
go version
```

## 2. Write small Golang HTTP service
Use your favourite editor or IDE to write a samll HTTP service in Golang. Create a file `greet.go` 

you can use this source code a sample.
```go
package main

import (
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Server", "Hello world...!")
	    w.WriteHeader(200)
	})
    http.ListenAndServe(":4000", nil)
}
```

It is very simple http service that listens to port `4000` on `/` and responds with **Hello World...!** 

The next step is to test your code, this can be done by running the program using Go
```bash
go run greet.go
```
Server now is ready to receive HTTP requests and respond. From any browser or Postman, call your development machine on port 4000 (be sure to allow port 4000 in `ufw`).

## 3. Build (compile) program into executable binayr
For program to be executable, we must complie it into binary, this is done usig `GOARCH`
```bash
go build greet.go
```
Once build is done, a new binary file is created `greet`, that can be executed using `./greet`

Congratulations, you have built your first Golang binary!

## 4. Move binary to production server/machine
It is common that we need to move compiled binaries to production servers or publish it. In this article I assume you have no CI/CD pipeline and just want to deploy your binary into production manually. Moving binary can be done by copying binary file using any copy method such as `scp`, `rsync`, etc.

It is crucial here to mention target location on the production server. Generally, binaries are located in `/usr/local/bin` directory.

For the binary to work we need to change owner to `sudo` and give it execution permission as we need it to run as service.
```bash
sudo chown root /usr/local/bin/greet
chmod +x /usr/local/bin/greet
```

## 5. Configure binary as a service
For our `greet` binary to run every time server starts and run in the background, we need to define it asa service. This can be achieved using `systemd` by configuring service definition for our binary.
1. Create service file
    ```bash
    cd /lib/systemd/system
    touch greet.service
    ```
2. Edit service file using `vim` or you favourite text editor and insert the following service definition
    ```
    [Unit]
    Description=Golang Web application
    [Service]
    Type=simple
    Restart=always
    RestartSec=5s
    ExecStart=/usr/local/bin/greet
    [Install]
    WantedBy=multi-user.target
    ```
3. Reload `systemd` to read newly added service configuration file
    ```bash
    systemctl daemon-reload
    ```
4. Start new service
    ```bash
    systemctl start greet.service
    ```
5. Enable service to start with system start
    ```bash
    systemctl enable greet.service
    ```

---
## Final Word
I hope this article help you have an overview of Golang service development cycle.<br>
Author: Hany E. Mamdouh
