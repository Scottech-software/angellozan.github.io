require 'fileutils'

desc "Build the Jekyll site"
task :build do
  puts "Building Jekyll site..."
  sh "bundle exec jekyll build"
end

desc "Build and test the site content using HTMLProofer"
task :test => :build do
  puts "Running HTMLProofer check..."
  sh 'bundle exec htmlproofer _site --disable-external --ignore-urls "/^http:\/\/127.0.0.1/,/^http:\/\/0.0.0.0/,/^http:\/\/localhost/"'
end

desc "Clean build artifacts"
task :clean do
  puts "Cleaning build artifacts..."
  FileUtils.rm_rf('_site')
  FileUtils.rm_rf('.jekyll-cache')
end

desc "Serve the Jekyll site locally"
task :serve do
  puts "Starting Jekyll server..."
  sh "bundle exec jekyll serve"
end

# Set default task
task :default => :test
