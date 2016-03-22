#  Building GCC on Ubuntu
The following instructions can be used to build **GCC 5.3.0** on Ubuntu 15.10

### Version
5.3.0

### Prerequisites
- git
- wget
- gcc
- flex
- bison
- binutils
- libgmp-dev
- libmpfr-dev
- libmpc-dev
- libc6-dev

#### Installing the dependencies
```
apt-get update && apt-get install -y git \
	wget \
	build-essential \
	flex \
	bison \
	binutils \
	libgmp-dev \
	libmpfr-dev \
	libmpc-dev \
	libc6-dev
```

### Downloading the source 
	git clone https://github.com/gcc-mirror/gcc.git

### Configuration 
	mkdir build-gcc
	cd build-gcc
	../gcc/configure --disable-nls --enable-languages=c,c++  --disable-multilib

### Building
	make
	make install

### Reference 
https://github.com/gcc-mirror/gcc
