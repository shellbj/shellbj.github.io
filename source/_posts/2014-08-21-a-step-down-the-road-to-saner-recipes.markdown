---
layout: post
title: "A Step Down the Road to Saner Recipes"
date: 2014-08-21 21:31:46 -0500
comments: true
categories: 
- chef
- docker
---

## The Tool Corner
We'll be using a number of different products to move our
recipes from the dark ages into the modern era.

- [Test Kitchen](http://kitchen.ci)
- [Foodcritic](http://www.foodcritic.io)
- [Chefspec](http://sethvargo.github.io/chefspec)
- [Serverspec](http://serverspec.org)
- [Kitchen-Docker](https://github.com/portertech/kitchen-docker)
- [Guard](http://guardgem.org)
- [Berkshelf](http://berkshelf.com)
- [rbenv](https://github.com/sstephenson/rbenv)


## Berkshelf
Managing dependencies for your cookbooks

```ruby Berksfile with non-standard locations for dependencies
# change me to an orbitz source
source "https://supermarket.getchef.com"
metadata
cookbook "yum", git: "ssh://git@git/chef/cookbook-yum.git"
cookbook "logging", git: "ssh://git@git/chef/cookbook-logging.git"
cookbook "system", git: "ssh://git@git/chef/cookbook-system.git"
```

## Foodcritic
Linting recipes for bad style and old practices

```bash Foodcritic run before things were cleaned up
✘ ↳ % bundle exec foodcritic .
FC001: Use strings in preference to symbols to access node attributes: ./attributes/default.rb:2
FC001: Use strings in preference to symbols to access node attributes: ./attributes/default.rb:3
FC001: Use strings in preference to symbols to access node attributes: ./attributes/default.rb:4
FC001: Use strings in preference to symbols to access node attributes: ./attributes/default.rb:6
FC001: Use strings in preference to symbols to access node attributes: ./attributes/default.rb:7
FC001: Use strings in preference to symbols to access node attributes: ./attributes/default.rb:8
FC001: Use strings in preference to symbols to access node attributes: ./attributes/graphite.rb:1
FC001: Use strings in preference to symbols to access node attributes: ./attributes/graphite.rb:2
FC001: Use strings in preference to symbols to access node attributes: ./attributes/graphite.rb:3
FC001: Use strings in preference to symbols to access node attributes: ./attributes/graphite.rb:4
FC001: Use strings in preference to symbols to access node attributes: ./attributes/graphite.rb:8
FC001: Use strings in preference to symbols to access node attributes: ./attributes/graphite.rb:10
FC001: Use strings in preference to symbols to access node attributes: ./attributes/graphite.rb:12
FC001: Use strings in preference to symbols to access node attributes: ./recipes/default.rb:55
FC001: Use strings in preference to symbols to access node attributes: ./recipes/default.rb:58
FC001: Use strings in preference to symbols to access node attributes: ./recipes/default.rb:65
FC001: Use strings in preference to symbols to access node attributes: ./recipes/default.rb:66
FC001: Use strings in preference to symbols to access node attributes: ./recipes/default.rb:70
FC001: Use strings in preference to symbols to access node attributes: ./templates/default/storm.yaml.erb:2
FC001: Use strings in preference to symbols to access node attributes: ./templates/default/storm.yaml.erb:11
FC001: Use strings in preference to symbols to access node attributes: ./templates/default/storm.yaml.erb:14
FC001: Use strings in preference to symbols to access node attributes: ./templates/default/storm.yaml.erb:15
FC001: Use strings in preference to symbols to access node attributes: ./templates/default/storm.yaml.erb:16
FC001: Use strings in preference to symbols to access node attributes: ./templates/default/storm.yaml.erb:19
FC001: Use strings in preference to symbols to access node attributes: ./templates/default/storm.yaml.erb:20
FC001: Use strings in preference to symbols to access node attributes: ./templates/default/storm.yaml.erb:21
FC002: Avoid string interpolation where not required: ./recipes/default.rb:58
FC002: Avoid string interpolation where not required: ./recipes/default.rb:70
FC003: Check whether you are running with chef server before using server-specific features: ./recipes/default.rb:45
FC019: Access node attributes in a consistent manner: ./attributes/graphite.rb:6
```

## Test Kitchen
And Integration testing tool for isolated environments.

This example uses docker for the driver.

```ruby .kitchen.docker.yml
---
driver:
  name: docker
  use_sudo: false
  require_chef_omnibus: true

provisioner:
  name: chef_solo

platforms:
  - name: centos-6.4
    driver_config:
      image: centos:6.4

suites:
  - name: default
    run_list:
      - recipe[system::repo]
      - recipe[system::java7]
      - recipe[kafka::default]
    attributes:
      datacenter: "eg"
      orbitz_env: "fqa1"
```

Note: The docker image, centos-6.4, no longer exists in the official repository for CentOS.

```bash How to use a non-standard kitchen file linenos:false
$ KITCHEN_YAML=.kitchen.docker.yml bundle exec kitchen list
```

## Chefspec
Rspec, unit style, tests for chef recipes

```ruby
require 'spec_helper'

describe 'kafka::default' do
  TerminatedExecutionError = Class.new(StandardError)

  let (:chef_run) { ChefSpec::Runner.new.converge(described_recipe) }

  it 'create the monitoring yum repository' do
    expect(chef_run).to create_yum_repository('monitoring')
  end

  it 'installs kafka and metrics plugin' do
    expect(chef_run).to install_yum_package('kafka')
    expect(chef_run).to install_yum_package('kafka-graphite')
  end

  it 'creates the user and group' do
    expect(chef_run).to create_group('kafka')
    expect(chef_run).to create_user('kafka')
  end

  it 'creates the data root' do
    expect(chef_run).to create_directory('/kafka').with(
                              owner: 'kafka',
                              group: 'kafka'
                            )
  end
```

## Serverspec
Integration tests to be run after convergence

```ruby
require 'spec_helper'

describe file('/etc/sysconfig/kafka') do
  it { should be_file }
  it { should be_mode 644 }
  it { should be_owned_by 'kafka' }
  it { should be_grouped_into 'kafka' }
end

describe user('kafka') do
   it { should exist }
   it { should belong_to_group 'kafka' }
   it { should have_home_directory '/home/kafka' }
   it { should have_login_shell '/bin/noshell' }
end

describe file('/kafka') do
  it { should be_directory }
  it { should be_mode 755 }
  it { should be_owned_by 'kafka' }
  it { should be_grouped_into 'kafka' }
end

describe yumrepo('monitoring') do
  it { should exist }
  it { should be_enabled }
end
```

## Guard
Bring all the tools and test together in an automation harness
              
```ruby
guard 'kitchen' do
  watch(%r{test/.+})
  watch(%r{^recipes/(.+)\.rb$})
  watch(%r{^attributes/(.+)\.rb$})
  watch(%r{^files/(.+)})
  watch(%r{^templates/(.+)})
  watch(%r{^providers/(.+)\.rb})
  watch(%r{^resources/(.+)\.rb})
end

guard 'foodcritic', :cookbook_paths => ".", :all_on_start => false do
  watch(%r{^attributes/(.+)\.rb$})
  watch(%r{^providers/(.+)\.rb})
  watch(%r{^recipes/(.+)\.rb$})
  watch(%r{^resources/(.+)\.rb})
  watch(%r{^templates/(.+)})
  watch('metadata.rb')
end

guard :rspec, cmd: 'bundle exec rspec', :all_on_start => false do
  watch(%r{^spec/.+_spec\.rb$})
  watch(%r{^(recipes)/(.+)\.rb$})   { |m| "spec/#{m[1]}_spec.rb" }
  watch('spec/spec_helper.rb')  { "spec" }
end
```

