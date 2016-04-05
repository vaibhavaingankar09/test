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
	
	Update the Gemfile to add a couple of missing Gems (and on SLES only point omnibus to the git repo downloaded earlier)
	
		gem 'omnibus', path: '/<source_root>/omnibus'
		gem 'omnibus-software', github: 'opscode/omnibus-software'
		gem 'rake'
		gem 'json'
		gem 'json_pure'
	
	**Note:** Add the *json* and *json_pure* gems.
	
	Finally install the build time Gems

		bundle install --binstubs
		git config --global user.email "you@example.com"
	
6. Update a number of different files
	
	1. Update omnibus.rb
	
			vi omnibus.rb
	
		Turn off s3 caching (we need to build a number of items differently from the s3 cached version), and optionally turn off git caching (it is recommended to keep git caching on).

			# Disable git caching
			# ------------------------------
			# use_git_caching false
			
			# Enable S3 asset caching
			# ------------------------------
			use_s3_caching false
			s3_access_key  ENV['AWS_ACCESS_KEY_ID']
			s3_secret_key  ENV['AWS_SECRET_ACCESS_KEY']
			s3_bucket      'opscode-omnibus-cache'
			
	2. Update chef-server.rb

			vi config/projects/chef-server.rb

		Here we need to remove an override and update the version of Ruby bundled - this is because the s3 cache contains an old version of the cacerts file which is no longer available outside of s3 caching so just use the latest instead, and the 2.1.4 version of Ruby has some SSL issues:

			#override :cacerts, version: '2014.08.20'
			override :rebar, version: "2.0.0"
			override :berkshelf2, version: "2.0.18"
			override :rabbitmq, version: "3.3.4"
			override :erlang, version: "17.5"
			override :ruby, version: "2.1.6"
			override :chef, version: "9a3e6e04f3bb39c2b2f5749719f0c21dd3f3f2ec"
