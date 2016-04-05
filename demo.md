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
				
		**Note:** Uncomment (remove the #) from the # *use_git_caching false* line to disable git caching
		
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
		
		**Note:** Just remove the *:cacerts* version line by commenting it out, and change the *:ruby version* to be *2.1.6*
	
	3. Update openresty.rb

			cp ../../omnibus-software/config/software/openresty.rb config/software/.
			vi config/software/openresty.rb
		
		We had to copy the default openresty.rb file in order to be able to make the changes otherwise the build process will download a fresh copy.
				
			# Options inspired by the OpenResty Cookbook
			#'--with-md5-asm',
			#'--with-sha1-asm',
			#'--with-pcre-jit',
			'--with-lua51',
			'--without-http_ssi_module',

		**Note:** Comment out the 3 *asm* and *jit* modules as they are not supported, but you have to change the *--with-luajit* option to *--with-lua51* as the default for *lua* (if not specified) is now with jit which again is not supported.
		
	4. Update libossp-uuid.rb

			vi config/software/libossp-uuid.rb
			
		The original source is no longer available, so change the url as below:

			source url: "https://gnome-build-stage-1.googlecode.com/files/uuid-1.6.2.tar.gz",
			md5: "5db0d43a9022a6ebbbc25337ae28942f"

		**Note:** The *md5* checksum should not change as it is the same file just from a different location
		
	5. Update sqitch.rb

			cp ../../omnibus-software/config/software/sqitch.rb config/software/.
			vi config/software/sqitch.rb

		Sqitch has updated a few times since the s3 caching version was stored and the original version is no longer available for download, so update the version and md5 checksum

			name "sqitch"
			default_version "0.999"

			dependency "perl"
			dependency "cpanminus"

			source url: "http://www.cpan.org/authors/id/D/DW/DWHEELER/App-Sqitch-#{version}.tar.gz",
			md5: "b3a9cac1254e0e90e4cc09fc84a66c93"
			
	6. Create nodejs.rb

			vi config/software/nodejs.rb
		
		Nodejs doesn't support Linux on z Systems, however there is a fork of nodejs that does, adding the below as the content of the empty file edited above downloads that version of nodejs

			name "nodejs"   

			default_version "0.10.36"
			dependency "python"

			version "0.10.36" do
				source md5: "02de00cb56c976f71a5a9eb693511fe7"
			end

			source url: "https://github.com/andrewlow/node/archive/V8-3.14.5.9-Node.js-#{version}-201501281023.tar.gz"
			relative_path "node-8-3.14.5.9-Node.js-#{version}-201501281023"

			build do
				env = with_standard_compiler_flags(with_embedded_path)
				env["LINK"]="g++"
				command "#{install_dir}/embedded/bin/python ./configure --dest-cpu=s390x --prefix=#{install_dir}/embedded", env: env
				make "-j #{workers}", env: env
				make "install", env: env
			end	
	
	7. Update openresty-lpeg.rb

			vi config/software/openresty-lpeg.rb

		As we turned off --with-luajit earlier we need to alter the openresty include path, so update the make command to be the same as below

			build do
				env = with_standard_compiler_flags(with_embedded_path)

				make "LUADIR=#{install_dir}/embedded/lua/include/", env: env
				command "install -p -m 0755 lpeg.so #{install_dir}/embedded/lualib", env: env
			end
	
	8. Update ohai.rb

			cp ../../omnibus-software/config/software/ohai.rb config/software/.
			vi config/software/ohai.rb

		Here there is an issue with the available gems, the simplest solution to convert the install to work within bundler - which guarantees that the relevant gems are in the correct location, so update the file to match the below:

			build do
				env = with_standard_compiler_flags(with_embedded_path)

				bundle "install --without development", env: env

				gem "build ohai.gemspec", env: env
				bundle "exec gem install ohai*.gem --no-ri --no-rdoc", env: env
			end
		
		**Note:** All we have done is perform the *gem install* inside of the *bundle exec*, this means that all the gems installed a few lines earlier with *bundle "install..* are available
		
	9. Update opscode-chef-mover.rb

			vi config/software/opscode-chef-mover.rb

		HiPE or High Performance Erlang is a native compiler for Erlang, but isn't supported on Linux on z Systems but has unfortunately been added as a requirement for chef-mover. Removing the hipe requirement still works so we remove it from the relx.config

			build do
				env = with_standard_compiler_flags(with_embedded_path)
	
				make "distclean", env: env
				command "cat relx.config | grep -v hipe > relx.config.mod", env: env
				command "mv -f relx.config.mod relx.config", env: env
				make "rel", env: env

		**Note:** We don't directly edit the *relx.config* file as it is updated each time the build is run, but simply remove the *hipe* dependency at build time with the two additional *command* lines above
