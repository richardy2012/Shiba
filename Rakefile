require 'json'
require 'fileutils'
include FileUtils

PREFIX = `npm prefix`.chomp
BIN_DIR = "#{PREFIX}/node_modules/.bin"

Dir.chdir PREFIX

def cmd_exists?(cmd)
  File.exists?(cmd) && File.executable?(cmd)
end

def ensure_cmd(cmd)
  $cmd_cache ||= []
  return true if $cmd_cache.include? cmd

  paths = ENV['PATH'].split(':').uniq
  unless paths.any?{|p| cmd_exists? "#{p}/#{cmd}" }
    raise "'#{cmd}' command doesn't exist"
  else
    $cmd_cache << cmd
  end
end

file "node_modules" do
  ensure_cmd 'npm'
  sh 'npm install'
  sh "#{BIN_DIR}/electron-rebuild"
end

file "bower_components" do
  ensure_cmd 'bower'
  sh 'bower install'
end

file "typings" do
  raise "'typings' command doesn't exist" unless cmd_exists? "#{BIN_DIR}/typings"
  sh "#{BIN_DIR}/typings install"
end

task :dep => [:node_modules, :bower_components, :typings]

task :build_slim do
  ensure_cmd 'slimrb'
  mkdir_p 'build/static'

  Dir['renderer/*.slim'].each do |slim_file|
    sh "slimrb #{slim_file} build/static/#{File.basename(slim_file, '.slim')}.html"
  end
end

task :build_typescript => [:typings] do
  ensure_cmd 'tsc'
  mkdir_p 'build/src/renderer'
  sh 'tsc -p ./browser'
  sh 'tsc -p ./renderer'
end

task :compile => [:build_slim, :build_typescript]

task :build => [:dep, :compile]

task :npm_publish => [:build] do
  mkdir 'npm-publish'
  %w(bower.json package.json build bin README.md).each{|p| cp_r p, 'npm-publish' }
  cd 'npm-publish' do
    sh 'bower install --production'
    sh 'npm install --save electron-prebuilt'
    sh 'npm publish'
  end
  rm_rf 'npm-publish'
end

task :prepare_release => [:build] do
  mkdir_p "archive"
  %w(bower.json package.json build).each{|p| cp_r p, 'archive' }
  cd 'archive' do
    sh 'npm install --production'
    sh 'bower install --production'
    sh 'npm uninstall electron-prebuilt'
  end
end

task :package do
  mkdir_p 'packages'
  def release(options)
    cd 'archive' do
      sh "#{BIN_DIR}/electron-packager ./ Shiba #{options.join ' '}"
      Dir['Shiba-*'].each do |dst|
        cp_r '../README.md', dst
        cp_r '../docs', dst
        sh "zip --symlinks #{dst}.zip -r #{dst}"
        mv "#{dst}.zip", '../packages'
        rm_r dst
      end
    end
  end

  electron_json = JSON.load(File.open('node_modules/electron-prebuilt/package.json'))
  electron_ver = electron_json['version']
  app_json = JSON.load(File.open('package.json'))
  ver = app_json['version']
  release %W(
    --platform=darwin
    --arch=x64
    --version=#{electron_ver}
    --asar
    --icon=../resource/image/icon/shibainu.icns
    --app-version='#{ver}'
    --build-version='#{ver}'
  )
  release %W(
    --platform=win32
    --arch=ia32
    --version=#{electron_ver}
    --asar
    --icon=../resource/image/icon/shibainu.ico
    --app-version='#{ver}'
    --build-version='#{ver}'
  )
  release %W(
    --platform=win32
    --arch=x64
    --version=#{electron_ver}
    --asar
    --icon=../resource/image/icon/shibainu.ico
    --app-version='#{ver}'
    --build-version='#{ver}'
  )
  release %W(
    --platform=linux
    --arch=ia32
    --version=#{electron_ver}
    --asar
    --icon=../resource/image/icon/shibainu.ico
    --app-version='#{ver}'
    --build-version='#{ver}'
  )
  release %W(
    --platform=linux
    --arch=x64
    --version=#{electron_ver}
    --asar
    --icon=../resource/image/icon/shibainu.ico
    --app-version='#{ver}'
    --build-version='#{ver}'
  )
end

task :release => [:prepare_release, :package]

task :build_test do
  ensure_cmd 'tsc'
  sh 'tsc -p test/main'
  sh 'tsc -p test/renderer'
end

task :test => [:build_test] do
  sh "#{BIN_DIR}/mocha --require intelli-espower-loader test/main/test/main test/renderer/test/renderer"
end

task :lint do
  ensure_cmd 'tslint'
  ts = `git ls-files`.split("\n").select{|p| p =~ /.ts$/}.join(' ')
  sh "tslint #{ts}"
end

task :clean do
  %w(npm-publish build/src build/static archive).each{|tmpdir| rm_rf tmpdir}
end

task :watch do
  sh 'guard --watchdir browser renderer typings test'
end
