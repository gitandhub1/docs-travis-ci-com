#!/usr/bin/env rake
# frozen_string_literal: true

require 'ipaddr'
require 'json'
require 'yaml'

require 'faraday'
require 'html-proofer'

def print_line_containing(file, str)
  File.open(file).grep(/#{str}/).each { |line| puts "#{file}: #{line}" }
end

def dns_resolve(hostname, rectype: 'A')
  JSON.parse(
    Faraday.get("https://dnsjson.com/#{hostname}/#{rectype}.json").body
  ).fetch('results').fetch('records')
end

def define_ip_range(nat_hostname, dest)
  data = dns_resolve(nat_hostname)

  File.write(
    dest,
    YAML.dump(
      'host' => nat_hostname,
      'ip_range' => data.sort { |a, b| IPAddr.new(a) <=> IPAddr.new(b) }
    )
  )

  puts "Updated #{dest}"
end

task default: :test

desc 'Runs the tests!'
task test: %i[build run_html_proofer]

desc 'Builds the site'
task build: %i[remove_output_dir regen] do
  rm_f '.jekyll-metadata'
  sh 'bundle exec jekyll build --config=_config.yml'
end

desc 'Remove the output dir'
task :remove_output_dir do
  rm_rf('_site')
end

desc 'Lists files containing beta features'
task :list_beta_files do
  files = FileList.new('**/*.md')
  files.exclude('_site/*', 'STYLE.md')
  files.each do |f|
    print_line_containing(f, '\.beta')
  end
end

desc 'Runs the html-proofer test'
task :run_html_proofer => [:build] do
  # seems like the build does not render `%3*`,
  # so let's remove them for the check
  url_swap = {
    /%3A\z/ => '',
    /%3F\z/ => '',
    /-\.travis\.yml/ => '-travisyml'
  }

  HTMLProofer.check_directory(
    './_site',
    url_swap: url_swap,
    internal_domains: ['docs.travis-ci.com'],
    connecttimeout: 600,
    only_4xx: true,
    typhoeus: {
      ssl_verifypeer: false, ssl_verifyhost: 0, followlocation: true
    },
    url_ignore: [
      'https://www.appfog.com/',
      /itunes\.apple\.com/,
      /coverity.com/,
      /articles201769485/
    ],
    file_ignore: %w[
      ./_site/api/index.html
      ./_site/user/languages/erlang/index.html
      ./_site/user/languages/objective-c/index.html
      ./_site/user/reference/osx/index.html
    ]
  ).run
end

desc 'Runs the html-proofer test for internal links only'
task :run_html_proofer_internal => [:build] do
  # seems like the build does not render `%3*`,
  # so let's remove them for the check
  url_swap = {
    /%3A\z/ => '',
    /%3F\z/ => '',
    /-\.travis\.yml/ => '-travisyml'
  }

  HTMLProofer.check_directory(
    './_site',
    url_swap: url_swap,
    disable_external: true,
    internal_domains: ['docs.travis-ci.com'],
    connecttimeout: 600,
    only_4xx: true,
    typhoeus: {
      ssl_verifypeer: false, ssl_verifyhost: 0, followlocation: true
    },
    file_ignore: %w[
      ./_site/api/index.html
      ./_site/user/languages/erlang/index.html
      ./_site/user/languages/objective-c/index.html
      ./_site/user/reference/osx/index.html
    ]
  ).run
end

file '_data/trusty-language-mapping.json' do |t|
  source = File.join(
    'https://raw.githubusercontent.com',
    'travis-infrastructure/terraform-config/master/aws-production-2',
    'generated-language-mapping.json'
  )

  File.write(t.name, Faraday.get(source).body)
end

file '_data/trusty_language_mapping.yml' => [
  '_data/trusty-language-mapping.json'
] do |t|
  File.write(
    t.name,
    YAML.dump(JSON.parse(File.read('_data/trusty-language-mapping.json')))
  )

  puts "Updated #{t.name}"
end

file '_data/ip_range.yml' do |t|
  define_ip_range('nat.travisci.net', t.name)
end

file '_data/ec2_ip_range.yml' do |t|
  define_ip_range('nat.aws-us-east-1.travisci.net', t.name)
end

file '_data/gce_ip_range.yml' do |t|
  define_ip_range('nat.gce-us-central1.travisci.net', t.name)
end

file '_data/macstadium_ip_range.yml' do |t|
  define_ip_range('nat.macstadium-us-se-1.travisci.net', t.name)
end

desc 'Refresh generated files'
task regen: (%i[clean] + %w[
  _data/ec2_ip_range.yml
  _data/gce_ip_range.yml
  _data/ip_range.yml
  _data/macstadium_ip_range.yml
  _data/trusty_language_mapping.yml
])

desc 'Remove generated files'
task :clean do
  rm_f(%w[
         _data/ec2_ip_range.yml
         _data/gce_ip_range.yml
         _data/ip_range.yml
         _data/macstadium_ip_range.yml
         _data/trusty-language-mapping.json
         _data/trusty_language_mapping.yml
       ])
end

desc 'Start Jekyll server'
task serve: :regen do
  sh 'bundle exec jekyll serve --config=_config.yml'
end

namespace :assets do
  task precompile: :build
end
