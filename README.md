&copy; 
# Building Apache Http Web Server
AT&T;

hello
-

> hello, its me
> > wake me up when its all over
>
>		return shell_exec("echo $input | $markdown_script");
>

[google]:	https://www.google.co.in 	"google"
[foo]: http://example.com/  "Optional Title Here"
**Apache Http 2.4.12** has been successfully built and tested for Linux on z Systems. The following instructions can be used for RHEL 7.1 and SLES 12.

### Version
2.4.12

### Section 1: Install the following dependencies
* git
* wget
* openssl
* gcc
* libtool
* autoconf
* make
* pcre 
* pcre-devel 
* libxml2 
* libxml2-devel
* expat-devel (or libexpat-devel)

RHEL7:
```
yum install -y git \
	wget
	openssl \
	gcc \
	libtool \
	autoconf \
	make \
	pcre \
	pcre-devel \
	libxml2 \
	libxml2-devel \
	expat-devel
```

SLES12:
```
zypper install -y git \
	wget
	openssl \
	gcc \
	libtool \
	autoconf \
	make \
	pcre \
	pcre-devel \
	libxml2 \
	libxml2-devel \
	libexpat-devel
```


### Section 2: Build and install Httpd
1. (For Rhel) Re-install the ca certificates for openssl framework

        yum reinstall -y ca-certificates
2. Get the source

        git clone -b 2.4.12 https://github.com/apache/httpd.git
3. Build and Install Apache Portable Runtime library
    * Get the apr source 
	
            cd httpd/srclib 
            git clone https://github.com/apache/apr.git
    * Configure the build
	
            cd apr 
           ./buildconf
           ./configure
    * Build the source
	
            make
    * Install the binaries
	
            make install
4. Configure the build

        cd httpd
        ./buildconf
        ./configure --prefix=<build-location>
5. Build the httpd source

        make
6. Install the binaries

        make install
7. Run the httpd service

        <build-location>/bin/httpd

### Section 3: Build and install php
1. Download php

        wget http://www.php.net/distributions/php-5.6.8.tar.gz
2. Untar php

        tar xvzf php-5.6.8.tar.gz 
3. Configure

		cd php-5.6.8 && ./configure --prefix=/usr/local/php --with-apxs2=/usr/local/bin/apxs --with-config-file-path=/usr/local/php --with-mysql 
4. Build the source

		make
5. Install the binaries

		make install

6. Enable php module 

		echo "AddType application/x-httpd-php .php" >> /usr/local/conf/httpd.conf
		echo -e  '<Directory />\n DirectoryIndex index.php \n </Directory>'  >> /usr/local/conf/httpd.conf


### References:
https://github.com/apache/httpd	
