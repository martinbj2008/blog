---
layout: post
title: "how to update github page in multi pos"
date: 2013-11-20 22:42
comments: true
categories: [git]
tags: [git, octopress]
---

### Requirement
I want to write blog with github page and octopress in the office
and in the home.

### Steps
The github page has work well done with pc in the home.
Now setup the environment in office to write github page.
The github page repo:
    https://github.com/martinbj2008/martinbj2008.github.io

the octopress is pushed to git repo as branch "source"

How to write blog and sync them to github while avoid conflict with home?

<!-- more -->

#### 1. clone the 'source' branch of github page repo.
    The branch includes markdown files and config files.

```
    git clone https://github.com/martinbj2008/martinbj2008.github.io blog
    cd blog
    git checkout -b source origin/source
```

#### 2. fake `_deploy` directory to avoid `rake deloy` failure.
    clone the github page repo and checkout branch `master`

```
    git clone git@github.com:martinbj2008/martinbj2008.github.io.git _deploy
```

#### 3. setup octopress
	install dependencies.
```
	sudo apt-get install rbenv
	sudo apt-get install ruby
	sudo bundle update rake
```

```
	gem install bundler
	rbenv rehash    # If you use rbenv, rehash to be able to run the bundle command
	bundle install
```

	Install the default Octopress theme.

```
	rake install
```

#### 4. enjoy it.
```
	rake generate
	rake preview
	rake deploy
```
### Solved Problem

#### 1. rake could not work.
```
    sudo bundle update rake
```

#### 2. rake version does not match, need edit `Gemfile`.

This work has been done in the home.
```
martin@ubuntu:~/git/octopress$ rake --version
rake, version 10.1.0
martin@ubuntu:~/git/octopress$ git diff Gemfile
diff --git a/Gemfile b/Gemfile
index cd8ce57..e7cd276 100644
--- a/Gemfile
+++ b/Gemfile
@@ -1,7 +1,7 @@
 source "https://rubygems.org"

 group :development do
-  gem 'rake', '~> 0.9'
+  gem 'rake', '~> 10.1'
   gem 'jekyll', '~> 0.12'
   gem 'rdiscount', '~> 2.0.7'
   gem 'pygments.rb', '~> 0.3.4'
martin@ubuntu:~/git/octopress$
```

#### 3. "cannot load such file -- mkmf"

sudo aptitude install ruby1.9.1-dev

```
junwei@localhost:~/git/blog$ sudo bundle update rake
[sudo] password for junwei:
Fetching gem metadata from https://rubygems.org/........
Fetching gem metadata from https://rubygems.org/..
Resolving dependencies...
Installing rake (10.1.1)
Installing RedCloth (4.2.9)
Gem::Installer::ExtensionBuildError: ERROR: Failed to build gem native extension.

        /usr/bin/ruby1.9.1 extconf.rb
/usr/lib/ruby/1.9.1/rubygems/custom_require.rb:36:in `require': cannot load such file -- mkmf (LoadError)
        from /usr/lib/ruby/1.9.1/rubygems/custom_require.rb:36:in `require'
        from extconf.rb:1:in `<main>'


Gem files will remain installed in /var/lib/gems/1.9.1/gems/RedCloth-4.2.9 for inspection.
Results logged to /var/lib/gems/1.9.1/gems/RedCloth-4.2.9/ext/redcloth_scan/gem_make.out

An error occurred while installing RedCloth (4.2.9), and Bundler cannot continue.
Make sure that `gem install RedCloth -v '4.2.9'` succeeds before bundling.
```
