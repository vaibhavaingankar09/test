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
	
	**Note:** Add the 		json and *json_pure* gems.
	
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
		
	10. Update opscode-solr4.rb

			vi files/private-chef-cookbooks/private-chef/recipes/opscode-solr4.rb

		Apache Solr is relies on a non-standard option -Xloggc which doesn't exist on IBM's JDK, however there is an equivalent -Xverbosegclog so we replace it as below:

			# Enable GC Logging (very useful for debugging issues)
			node.default['private_chef']['opscode-solr4']['command'] << " -Xverbosegclog:#{File.join(solr_log_dir, "gclog.log")} -verbose:gc -XX:+PrintHeapAtGC -XX:+PrintGCTimeStamps -XX:+PrintGCDetails -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCApplicationConcurrentTime -XX:+PrintTenuringDistribution"
			
		**Note:** All we have done is replaced *-Xloggc* with *-Xverbosegclog*
	
	11. Update old_postgres_cleanup.rb

			vi files/private-chef-cookbooks/private-chef/recipes/old_postgres_cleanup.rb
			
		postgres has changed name between chef versions to postgresql, however the cleanup script doesn't correctly handle a new installation situation so we add a simple check to prevent errors during installation

			runit_service "postgres" do
				action [:stop, :disable]
				not_if { not File.exist?('/opt/opscode/service/postgres') }
			end

		**Note:** The issue is only when attempting to stop a non-existant *postgres* service, so we only protect that *runit_service* call
		
	12. Update ncurses.rb

			cp ../../omnibus-software/config/software/ncurses.rb config/software/.
			vi config/software/ncurses.rb

		Like earlier items the download site for ncurses no longer provides this specific version, to fix this simply change the default version to be downloaded as below:

			name "ncurses"
			default_version "5.9"
	
			dependency "libtool" if aix?
			dependency "patch" if solaris2?

		**Note:** The only change is to the *default_version* in order to download *5.9* rather than *5.9-20150530*
	
7. Install or stub fakeroot and build Chef Server Omnibus

	You may already have fakeroot available on your system, but if not (and as we are building as root) you can use the following to temporarily stub fakeroot:

		echo '"$@"' > /usr/bin/fakeroot
		chmod +x /usr/bin/fakeroot
	
	**Note:** Don't forget to remove the fakeroot script after the build has completed

		bin/omnibus build chef-server

	**Note:** Occasionally during the download phase there will be errors similar to *...net_fetcher.rb:180:in 'each': comparison of NilClass with 861 failed (ArgumentError)...,* if you see these just start the build process again and it should download the second time.

	Once this completes your built RPM will be in the *pkg* subdirectory.

	If you wanted to clean your build process and start again you can do some of the following:

		bin/omnibus clean chef-server
		# or #
		bin/omnibus clean chef-server --purge
	
	These will clean the build tree in the first case and purge the downloaded files in the second (causing it to redownload everything). Removing the git cache is a little more direct:

		rm -rf /var/cache/omnibus/cache/git_cache/
	
	Which will cause the build process to rebuild everything even if nothing has changed.
