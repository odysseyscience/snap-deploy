require 'bundler/setup'
require "bundler/gem_tasks"
require 'rspec/core/rake_task'
require 'tmpdir'

desc "Run units"
RSpec::Core::RakeTask.new("spec:unit") do |t|
  t.pattern = "spec/unit/**/*_spec.rb"
  t.rspec_opts = "--format documentation"
end

desc "Run integrations"
RSpec::Core::RakeTask.new("spec:integration") do |t|
  t.pattern = "spec/integration/**/*_spec.rb"
  t.rspec_opts = "--format documentation"
end

task :spec => ['spec:unit', 'spec:integration']
task :default => :spec

namespace :dist do
  desc "create a RPM package"
  task :rpm do
    pipeline_counter = ENV['GO_PIPELINE_COUNTER'] || 1
    version = "0.0.1.#{pipeline_counter}"
    git_revision = %x[git log -1 --pretty='%h'].chomp

    Bundler.with_clean_env do
      Dir.mktmpdir do |dir|
        root_dir = File.expand_path("../", __FILE__)
        pkg_dir = "#{root_dir}/pkg"

        app_name = 'snap-deploy'
        app_server_root = File.join(dir, "/opt/local", 'snap-deploy')

        puts "App server root #{app_server_root}"

        release = "#{pipeline_counter}.#{git_revision}"

        puts "App server root #{app_server_root}"
        sh "git clone --depth 1 file://#{root_dir} #{app_server_root}"

        cd dir do
          mkdir_p 'usr/local/bin'
          File.open('usr/local/bin/snap-deploy', 'w') do |f|
            f.puts('#!/bin/bash')
            f.puts('unset BUNDLE_BIN_PATH')
            f.puts('unset BUNDLE_GEMFILE')
            f.puts('unset GEM_HOME')
            f.puts('unset GEM_PATH')
            f.puts('unset RUBYLIB')
            f.puts('unset RUBYOPT')
            f.puts('export PATH=/opt/local/ruby/2.0.0-p353/bin:$PATH')
            f.puts('/opt/local/snap-deploy/bin/snap-deploy')
          end
          File.chmod 0755, 'usr/local/bin/snap-deploy'
        end

        cd app_server_root do
          puts "*** Installing gems in standalone mode"
          sh("export NOKOGIRI_USE_SYSTEM_LIBRARIES=1; bundle install --standalone --path bundle --local --without 'development test'")

          puts "*** Cleaning up files that should not be packaged"
          rm_rf Dir['{.git*,**/*.gem}']
          rm_rf Dir["vendor/ruby/*/gems/*/{spec,specs,test,features,tests,examples,example,doc,docs,doc-api,guides,ext,java}"]
          rm_rf Dir['{tmp/*}']
          rm_rf Dir['{.rvmrc,.ruby-version,.rbenv-gemsets,.rspec,.init.sh}']

          puts "*** Writing out the version.txt file"
          File.open("#{app_server_root}/version.txt", 'w') do |f|
            f.puts("Version:  #{version}")
            f.puts("Release:  #{release}")
            f.puts("Revision: #{git_revision}")
          end

          rm_rf pkg_dir
          mkdir pkg_dir
        end

        description_string = %Q{This package contains a continuous deployment tool.}
        sh(%Q{
          bundle exec fpm -s dir -t rpm -C #{dir} --rpm-auto-add-directories --package #{pkg_dir}/#{app_name}-#{version}-#{release}.x86_64.rpm --name #{app_name} --architecture x86_64 --version "#{version}" --rpm-user root --rpm-group root --maintainer snap-ci@thoughtworks.com --vendor snap-ci@thoughtworks.com --url http://snap-ci.com --description "#{description_string}" --iteration #{release} --license 'Private'  --verbose .
        })
      end
    end
  end
end
