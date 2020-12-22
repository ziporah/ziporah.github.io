---
layout: post
title:  "Welcome to my first Jekyll post!"
date:   2020-12-22 19:00:00 +0100
categories: jekyll update
tag: jekyll arm64 beautiful-jekyll-theme
---
This is my first Jekyll post. I've tried to layout the site already and applied some templates, now the site is just waiting to be filled with content.

I tested this site first on my brand new macbook M1, which has the new arm64 cpu.
At least you could call it challenging to get the system functional.

It should be a breeze to install jekyll, at least according to the documentation :eyes:.

First you install homebrew. As suggested on mac arm, install in /opt/homebrew
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
````


After installing homebrew, I added Ruby and rbenv to select between different Ruby environments, but in the end I only managed to get jekyll working using the system ruby 2.7.2

```
brew install rbenv
brew install ruby
ruby --version
ruby 2.7.2p137 (2020-10-01 revision 5445e04352) [arm64-darwin20]
```

There was a problem with the ffi installation on arm64, in the past days they seem to have released 1.14.2 and this version installs without problems.
I however installed 1.13.1 from source as follows:
```
$ cd ~/development/github
$ git clone https://github.com/ffi/ffi
$ cd ffi
$ bundle install --path vendor/bundle
$ bundle exec rake build
$ gem install --user pkg/ffi-1.13.1.gem
```

Then I installed jekyll from the gem repository and initiated a new environment.
I intercepted the installation when it asked for the sudo password

```
gem install --user jekyll -v 4.2.0
gem install --user bundler
$ cd ~/development/github
$ mkdir ziporah.github.io
$ cd !$
$ jekyll new .
Running bundle install in /.../ziporah.github.io/...


Your user account isn't allowed to install to the system RubyGems.
  You can cancel this installation and run:

      bundle config set --local path 'vendor/bundle'
      bundle install

  to install the gems into ./vendor/bundle/, or you can enter your password
  and install the bundled gems to RubyGems using sudo.

  Password:
$ bundle config set --local path 'vendor/bundle'
$ bundle install
Fetching gem metadata from https://rubygems.org/..........
Fetching gem metadata from https://rubygems.org/.
Resolving dependencies...
Using bundler 2.2.2
Fetching public_suffix 4.0.6
Fetching ffi 1.14.2
Fetching forwardable-extended 2.6.0
Fetching concurrent-ruby 1.1.7
Fetching rb-fsevent 0.10.4
Fetching colorator 1.1.0
Fetching eventmachine 1.2.7
Fetching http_parser.rb 0.6.0
Installing colorator 1.1.0
Installing forwardable-extended 2.6.0
Installing rb-fsevent 0.10.4
Fetching rexml 3.2.4
Fetching liquid 4.0.3
Fetching mercenary 0.4.0
Installing public_suffix 4.0.6
Fetching rouge 3.26.0
Installing http_parser.rb 0.6.0 with native extensions
Installing eventmachine 1.2.7 with native extensions
Installing mercenary 0.4.0
Installing liquid 4.0.3
Installing rexml 3.2.4
Installing concurrent-ruby 1.1.7
Installing rouge 3.26.0
Fetching unicode-display_width 1.7.0
Fetching pathutil 0.16.2
Fetching safe_yaml 1.0.5
Installing pathutil 0.16.2
Installing unicode-display_width 1.7.0
Installing safe_yaml 1.0.5
Installing ffi 1.14.2 with native extensions
Fetching kramdown 2.3.0
Fetching i18n 1.8.5
Fetching terminal-table 2.0.0
Fetching addressable 2.7.0
Installing terminal-table 2.0.0
Installing i18n 1.8.5
Installing addressable 2.7.0
Installing kramdown 2.3.0
Fetching sassc 2.4.0
Fetching kramdown-parser-gfm 1.1.0
Fetching rb-inotify 0.10.1
Fetching em-websocket 0.5.2
Installing rb-inotify 0.10.1
Installing kramdown-parser-gfm 1.1.0
Installing em-websocket 0.5.2
Installing sassc 2.4.0 with native extensions
Fetching listen 3.3.3
Installing listen 3.3.3
Fetching jekyll-watch 2.2.1
Installing jekyll-watch 2.2.1
Fetching jekyll-sass-converter 2.1.0
Installing jekyll-sass-converter 2.1.0
Fetching jekyll 4.2.0
Installing jekyll 4.2.0
Fetching jekyll-feed 0.15.1
Fetching jekyll-seo-tag 2.7.1
Installing jekyll-feed 0.15.1
Installing jekyll-seo-tag 2.7.1
Fetching minima 2.5.1
Installing minima 2.5.1
Bundle complete! 6 Gemfile dependencies, 31 gems now installed.
Bundled gems are installed into `./vendor/bundle`
Post-install message from i18n:

HEADS UP! i18n 1.1 changed fallbacks to exclude default locale.
But that may break your application.

If you are upgrading your Rails application from an older version of Rails:

Please check your Rails app for 'config.i18n.fallbacks = true'.
If you're using I18n (>= 1.1.0) and Rails (< 5.2.2), this should be
'config.i18n.fallbacks = [I18n.default_locale]'.
If not, fallbacks will be broken in your app by I18n 1.1.x.

If you are starting a NEW Rails application, you can ignore this notice.

For more info see:
https://github.com/svenfuchs/i18n/releases/tag/v1.1.0

$  bundle exec jekyll serve

Configuration file: /.../ziporah.github.io/_config.yml
            Source: /.../ziporah.github.io
       Destination: /.../ziporah.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
       Jekyll Feed: Generating feed for posts
                    done in 0.207 seconds.
 Auto-regeneration: enabled for '/.../ziporah.github.io'
    Server address: http://127.0.0.1:4000/
  Server running... press ctrl-c to stop.
```


I copied over the required files from the beautiful-jekyll-theme and changed back to jekyll 3.9.0,
since the beautiful theme doesn't seem to work yet with 4.2.0

```
$ bundle update
$ bundle exec jekyll server
```


As a reference I keep this here:

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
