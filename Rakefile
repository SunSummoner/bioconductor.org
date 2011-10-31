require 'rubygems'
require 'nanoc3/tasks'
require 'yaml'
require 'fileutils'
require './lib/data_sources/gmane_list.rb'
require './scripts/search_indexer.rb'
require './scripts/parse_bioc_views.rb'
require 'open3'

include Open3


@clear_search_index_commands = [
  "curl -s http://localhost:8983/solr/update --data-binary '<delete><query>*:*</query></delete>' -H 'Content-type:text/xml; charset=utf-8'",
  "curl -s http://localhost:8983/solr/update --data-binary '<optimize/>' -H 'Content-type:text/xml; charset=utf-8'",
  "curl -s  http://localhost:8983/solr/update --data-binary '<commit/>' -H 'Content-type:text/xml; charset=utf-8'"
  ]



desc "write version info to doc root for javascript to find"
task :write_version_info do
  site_config = YAML.load_file("./config.yaml")
  js = %Q(var develVersion = "#{site_config["devel_version"]}";\nvar releaseVersion="#{site_config["release_version"]}";\n)
  js += %Q(var versions = [")
  js += site_config["versions"].join(%Q(",")) + %Q("];\n)
  f = File.open("#{site_config["output_dir"]}/js/versions.js", "w")
  f.puts js
  f.close
end

desc "copy assets to output directory"
task :copy_assets do
  site_config = YAML.load_file("./config.yaml")
  output_dir = site_config["output_dir"]
  system "rsync -gprt --partial --exclude='.svn' assets/ #{output_dir}"
end

desc "Run nanoc3 compile"
task :compile => [ :real_compile, :post_compile]

task :real_compile do
  system "nanoc3 co"
end

task :post_compile do
  puts "running post-compilation tasks..."
  site_config = YAML.load_file("./config.yaml")
  src = "#{site_config["output_dir"]}/packages/#{site_config["release_version"]}/BiocViews.html"
  other_versions = site_config["versions"] - [site_config["release_version"]]
  for version in other_versions
    dest = "#{site_config["output_dir"]}/packages/#{version}"
    FileUtils.mkdir_p dest
    FileUtils.cp(src, dest)
    puts "copied output/packages/#{site_config["release_version"]}/BiocViews.html to output/packages/#{version}/BiocViews.html"
  end
  cwd = FileUtils.pwd
  FileUtils.cd "#{site_config["output_dir"]}/packages"

  FileUtils.rm_f "devel"
  FileUtils.rm_f"release"
  FileUtils.ln_s "#{site_config["release_version"]}", "release"
  FileUtils.ln_s "#{site_config["devel_version"]}", "devel"
  puts "Generated symlinks for release and devel"
  FileUtils.cd cwd
end

desc "Nuke output directory !! uses rm -rf !!"
task :real_clean do
  site_config = YAML.load_file("./config.yaml")
  output_dir = site_config["output_dir"]
  FileUtils.rm_rf(output_dir)
  FileUtils.mkdir_p(output_dir)
end

desc "Build the bioconductor.org site (default)"
task :build => [ :compile, :copy_assets, :write_version_info ]

task :default => :build

desc "deploy (sync) to staging (run on merlot2)"
task :deploy_staging do
  dst = '/loc/www/bioconductor-test.fhcrc.org'
  site_config = YAML.load_file("./config.yaml")
  output_dir = site_config["output_dir"]
  system "rsync -av --links --partial --partial-dir=.rsync-partial --exclude='.svn' #{output_dir}/ #{dst}"
  chmod_cmd = "chmod -R a+r /loc/www/bioconductor-test.fhcrc.org/packages/json"
  system chmod_cmd
end

desc "deploy (sync) to production"
task :deploy_production do
  site_config = YAML.load_file("./config.yaml")
  src = '/loc/www/bioconductor-test.fhcrc.org'
  dst = site_config["production_deploy_root"]
  system "rsync -av --links --partial --partial-dir=.rsync-partial --exclude='.svn' #{src}/ #{dst}/"
end


desc "Clear search index (and local cache)"
task :clear_search_index do
  for command in @clear_search_index_commands
    puts command
    system command
  end
  dir = (`hostname`.chomp == 'merlot2') ? "/home/biocadmin" : "."
  
  FileUtils.rm_f "#{dir}/search_indexer_cache.yaml"
end


#todo fix this
desc "Clear search index (and local cache) on production. This will cause searches to fail until indexing is re-done!"
task :clear_search_index_production do
  for command in @clear_search_index_commands
    puts %Q(ssh webadmin@krait "#{command}")
    system %Q(ssh webadmin@krait "#{command}")
  end
  puts "ssh webadmin@krait rm -f /home/webadmin/search_indexer_cache.yaml"
  system "ssh webadmin@krait rm -f /home/webadmin/search_indexer_cache.yaml"
end

desc "Re-index the site for the search engine"
task :search_index do
  if (SearchIndexer.is_solr_running?)
    hostname = `hostname`.chomp
    args = [] # directory to index, location of cache file, location of output shell script, url of site
    if (hostname =~ /^dhcp/i) # todo - don't test this, instead see if nanoc is installed
      args = ['./output', './', 'scripts', 'http://localhost:3000'] 
    elsif (hostname == 'merlot2')
      args = ['/loc/www/bioconductor-test.fhcrc.org', '/home/biocadmin', '/home/biocadmin', 'http://bioconductor-test.fhcrc.org']
    end
    pwd = FileUtils.pwd
    si = SearchIndexer.new(args)
    FileUtils.cd pwd # just in case
    cmd = "#{args[2]}/index.sh"

    chmod_cmd = "chmod u+x #{cmd}"
    system chmod_cmd
    # todo fork the indexer to a background task because it could take a while in some cases

    stdin, stdout, stderr = Open3.popen3("sh ./scripts/index.sh")
    puts "running indexer, stdout = "
    puts stdout.readlines
    puts "stderr = "
    puts stderr.readlines
  else
    puts "solr is not running, not re-indexing site."
  end
end

desc "Re-run search indexing on production"
task :index_production do
  system("scp config.yaml webadmin@krait~")
  system("scp scripts/search_indexer.rb webadmin@krait:~")
  system("scp scripts/get_links.rb webadmin@krait:~")
  system(%Q(ssh webadmin@krait "cd /home/webadmin && /home/webadmin/do_index.rb"))
  #system("ssh webadmin@krait chmod +x /home/webadmin/index.sh")
  system(%Q(ssh webadmin@krait "/bin/sh /home/webadmin/index.sh"))
end

desc "Re-run search indexing cran package home pages on production"
task :index_cran_production do
  system("scp scripts/cran_search_indexer.rb webadmin@krait:~")
  system("ssh webadmin@krait /home/webadmin/do_index_cran.rb")
  system("ssh webadmin@krait /home/webadmin/index_cran.sh")
end

desc "Runs nanoc's dev server on localhost:3000"
task :devserver => [:build] do
  system "nanoc3 aco"
end

desc "Get JSON files required for BiocViews pages"
task :get_json do
  json_dir = "assets/packages/json"
  #todo - nuke json_dir before starting?
  FileUtils.mkdir_p json_dir
  site_config = YAML.load_file("./config.yaml")
  versions = site_config["versions"]
  devel_version = site_config["devel_version"]
  devel_repos = site_config["devel_repos"]
  #version_str = %Q("#{site_config["release_version"]}","#{site_config["devel_version"]}")
  version_str = '"' + versions.join('","') + '"'
  # todo - make sure destination directories exist
  r_cmd = %Q(R CMD BATCH -q --vanilla --no-save --no-restore '--args versions=c(#{version_str}) outdir="#{json_dir}"' scripts/getBiocViewsJSON.R getBiocViewsJSON.log)
system(r_cmd)
  
  
#  repos = ["data/annotation", "data/experiment", "bioc"]
  
  
  #todo remove
  system("scp scripts/get_vignette_titles.rb webadmin@krait:~")
  #end remove
  
  for version in versions
    if version == devel_version
      repos = devel_repos
    else
      repos = ["data/annotation", "data/experiment", "bioc"]
    end
    
    fullpaths = repos.map{|i| "#{json_dir}/#{version}/#{i}/biocViews.json"}
    
    #todo remove
    repos.each do |repo|
      system %Q(ssh webadmin@krait "ruby ./get_vignette_titles.rb /extra/www/bioc/packages/#{version}/#{repo} > ~/vignette_titles.json")
      system("scp webadmin@krait:~/vignette_titles.json #{json_dir}/#{version}/#{repo}")
    end
    #end remove
    
    
    
    args = [fullpaths, "#{json_dir}/#{version}/tree.json"]
    #pp args
    
    ParseBiocViews.new(args)
  end
end

