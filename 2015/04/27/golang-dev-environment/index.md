# Golang Development Environment

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2015-04-27
- Tags: Go, Tutorial
- Canonical: https://kunalkushwaha.github.io/2015/04/27/golang-dev-environment/

Setting up golang environment is quite simple good docs are already present in golang.org.
But I couldn’t find any simple doc, where complete setup with GOROOT and GOPATH along with github is explained.

So I thought it might be helpful to others too.
My dev environment is ubuntu based [ElementryOS “Freya”](https://elementary.io/), So it would work on all Ubuntu based distros.

### Golang Installer

Golang comes with single tar file setup can be downloaded from [here](https://golang.org/dl/)
extract the tar file to your /usr/local using below command

```shell
tar -C /usr/local -xzf go$VERSION.$OS-$ARCH.tar.gz
```

this will install your all go binaries under following dir structure.

base folder where go is installed.

```shell
/usr/local/go
```

All standard go binaries are in

```shell
/usr/local/go/bin
```

### Golang environment setup.

Now next step is to define golang related environment variable.
There are two important golang Env variables that need to be defined.

1. GOROOT : This variable have value, where golang is installed.
2. GOPATH : This is like workspace. This folder will be root of all go getable packages and projects.

So you need to create a folder for GOPATH. My fav is ~/go folder.

```shell
kunal@kunal-Aspire-5670:~$ mkdir ~/go
```

So now define GOROOT, GOPATH and append PATH variables.

```shell
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```

Its better to append your ~/.profile file with these 3 lines. So no need to export these variables every time you restart machine.

After setting environment variables check if everything is set as per expected or not!
Use “go env” command.

```shell
kunal@kunal-Aspire-5670:~$ go env
GOARCH="386"
GOBIN=""
GOCHAR="8"
GOEXE=""
GOHOSTARCH="386"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/home/kunal/go"
GORACE=""
GOROOT="/usr/local/go"
GOTOOLDIR="/usr/local/go/pkg/tool/linux_386"
CC="gcc"
GOGCCFLAGS="-fPIC -m32 -pthread -fmessage-length=0"
CXX="g++"
CGO_ENABLED="1"
```

verify GOPATH and GOROOT.

### go getable.

One of beauty of golang is “go get” command.
It automatically downloads the all required packages along with your program from git repo or mercurial.

Prerequisite to this is you should install git and mercurial both on your machine.
mercurial is required as lot of official libraries of golang is still hosted at google code and it hosted with mercurial.

```shell
kunal@kunal-Aspire-5670:~$ sudo apt-get install mercurial
```

So we are set with all basics, now ready for go getable your code!

after your go get github.com/, You will see your ~/go folder have sum folder structure created.

```shell
kunal@kunal-Aspire-5670:~$ ll go
total 32
drwxrwxr-x  6 kunal kunal  4096 Apr 20 20:28 ./
drwx------ 24 kunal kunal 12288 Apr 27 22:45 ../
drwxrwxr-x  2 kunal kunal  4096 Apr 16 03:27 bin/
drwxrwxr-x  3 kunal kunal  4096 Apr 12 21:26 pkg/
drwxrwxr-x  8 kunal kunal  4096 Apr 16 03:24 src/
```

These are folder which will have all your go getable code/packages and binaries.
\*bin : folder will have binaries build after you build go project using “go build”
\*pkg : this will have all pakages downloaded due to dependencies of your program. These packages compiled packages.
\*src : this will have your source code under folder of source of code like github.com , bitbucket.com, code.google.com etc
e.g

```shell
kunal@kunal-Aspire-5670:~$ ll go/src/
total 32
drwxrwxr-x  8 kunal kunal 4096 Apr 16 03:24 ./
drwxrwxr-x  6 kunal kunal 4096 Apr 20 20:28 ../
drwxrwxr-x  3 kunal kunal 4096 Apr 16 03:24 bitbucket.org/
drwxrwxr-x  3 kunal kunal 4096 Apr 12 21:28 code.google.com/
drwxrwxr-x 38 kunal kunal 4096 Apr 20 20:05 github.com/
drwxrwxr-x  3 kunal kunal 4096 Apr 12 21:29 golang.org/
drwxrwxr-x  3 kunal kunal 4096 Apr 12 21:29 google.golang.org/
drwxrwxr-x  6 kunal kunal 4096 Apr 16 03:26 gopkg.in/
```

So you are now all set for code/test/ship in golang. Happy Coding :)
