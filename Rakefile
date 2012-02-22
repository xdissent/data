abort "Please use Ruby 1.9 to build Ember.js!" if RUBY_VERSION !~ /^1\.9/

require "rubygems"
require "net/github-upload"

require "bundler/setup"
require "erb"
require "rake-pipeline"

LICENSE = File.read("generators/license.js")

desc "Build ember-data.js"
task :dist => :clean do
  Rake::Pipeline::Project.new("Assetfile").invoke
  minified = File.read("dist/ember-data.min.js")
  File.open("dist/ember-data.min.js", "w") do |file|
    file.write "#{LICENSE} #{minified}"
  end
end

desc "Clean build artifacts from previous builds"
task :clean do
  sh "rm -rf tmp dist tests/ember-data-tests.js"
end

### UPLOAD LATEST EMBERJS BUILD TASK ###
desc "Upload latest Ember Data build to GitHub repository"
task :upload => :dist do
  # setup
  login = `git config github.user`.chomp  # your login for github
  token = `git config github.token`.chomp # your token for github

  # get repo from git config's origin url
  origin = `git config remote.origin.url`.chomp # url to origin
  # extract USERNAME/REPO_NAME
  # sample urls: https://github.com/emberjs/ember.js.git
  #              git://github.com/emberjs/ember.js.git
  #              git@github.com:emberjs/ember.js.git
  #              git@github.com:emberjs/ember.js

  repo = origin.match(/github\.com[\/:](.+?)(\.git)?$/)[1]
  puts "Uploading to repository: " + repo

  gh = Net::GitHub::Upload.new(
    :login => login,
    :token => token
  )

  puts "Uploading ember-data-latest.js"
  gh.replace(
    :repos => repo,
    :file  => 'dist/ember-data.js',
    :name => 'ember-data-latest.js',
    :content_type => 'application/json',
    :description => "Ember Data Master"
  )

  puts "Uploading ember-data-latest.min.js"
  gh.replace(
    :repos => repo,
    :file  => 'dist/ember-data.min.js',
    :name => 'ember-data-latest.min.js',
    :content_type => 'application/json',
    :description => "Ember.js Data Master (minified)"
  )
end



### RELEASE TASKS ###

EMBER_VERSION = File.read("VERSION").strip

namespace :release do

  def pretend?
    ENV['PRETEND']
  end

  namespace :framework do
    desc "Update repo"
    task :update do
      puts "Making sure repo is up to date..."
      system "git pull" unless pretend?
    end

    desc "Update Changelog"
    task :changelog do
      last_tag = `git describe --tags --abbrev=0`.strip
      puts "Getting Changes since #{last_tag}"

      cmd = "git log #{last_tag}..HEAD --format='* %s'"
      puts cmd

      changes = `#{cmd}`
      output = "*Ember #{EMBER_VERSION} (#{Time.now.strftime("%B %d, %Y")})*\n\n#{changes}\n"

      unless pretend?
        File.open('CHANGELOG', 'r+') do |file|
          current = file.read
          file.pos = 0;
          file.puts output
          file.puts current
        end
      else
        puts output.split("\n").map!{|s| "    #{s}"}.join("\n")
      end
    end

    desc "bump the version to the one specified in the VERSION file"
    task :bump_version, :version do
      puts "Bumping to version: #{EMBER_VERSION}"

      unless pretend?
        # Bump the version of each component package
        Dir["packages/ember*/package.json", "ember.json"].each do |package|
          contents = File.read(package)
          contents.gsub! %r{"version": .*$}, %{"version": "#{EMBER_VERSION}",}
          contents.gsub! %r{"(ember-?\w*)": [^\n\{,]*(,?)$} do
            %{"#{$1}": "#{EMBER_VERSION}"#{$2}}
          end

          File.open(package, "w") { |file| file.write contents }
        end
      end
    end

    desc "Commit framework version bump"
    task :commit do
      puts "Commiting Version Bump"
      unless pretend?
        sh "git reset"
        sh %{git add VERSION CHANGELOG packages/**/package.json}
        sh "git commit -m 'Version bump - #{EMBER_VERSION}'"
      end
    end

    desc "Tag new version"
    task :tag do
      puts "Tagging v#{EMBER_VERSION}"
      system "git tag v#{EMBER_VERSION}" unless pretend?
    end

    desc "Push new commit to git"
    task :push do
      puts "Pushing Repo"
      unless pretend?
        print "Are you sure you want to push the ember.js repo to github? (y/N) "
        res = STDIN.gets.chomp
        if res == 'y'
          system "git push"
          system "git push --tags"
        else
          puts "Not Pushing"
        end
      end
    end

    desc "Prepare for a new release"
    task :prepare => [:update, :changelog, :bump_version]

    desc "Commit the new release"
    task :deploy => [:commit, :tag, :push]
  end
end

task :default => :dist
