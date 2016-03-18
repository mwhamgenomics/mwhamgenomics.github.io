---
layout: post
title: Setting up a Ruby virtual environment using RVM
categories: ['programming', 'dev_ops']
---

# RVM install

## fetch the RVM install script
curl -o rvm_install.sh https://raw.githubusercontent.com/wayneeseguin/rvm/master/binscripts/rvm-installer

## run the install script, installing the lastest stable release of RVM
sh rvm_install.sh stable

source ~/.profile
source ~/.rvm/scripts/rvm

rvm list
rvm list known
rvm install 2.3.0
(can specify an isolated install of Homebrew)
which ruby
which gem

# Jekyll install
gem install bundler
bundle install
