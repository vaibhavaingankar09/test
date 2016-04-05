# Building Chef Server Omnibus on Ubuntu

1. Install the dependencies 
	apt-get install -y git ruby-dev ruby make build-essential patch libreadline6-dev libreadlibe6 bzip2 tar wget curl libxml2-dev openjdk-7-jre openjdk-7-jdk vim 

2. Build Ruby from source

3. Correct the gem environment and install the bundler
	
	export GEM_HOME=/home/<USER>/.gem/ruby
	export PATH=$GEM_HOME/bin:$PATH
	gem install bundler
	
4. Download the build process code
	
	cd /<source_root>/
	git clone https://github.com/chef/omnibus-software
	git clone https://github.com/chef/chef-server

5. Checkout the required version of Chef Server and download Ruby Gems requirements

	cd chef-server/
	git checkout 12.1.2
	cd omnibus
	vi Gemfile
	

