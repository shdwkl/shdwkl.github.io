# frozen_string_literal: true

source "https://rubygems.org"

gemspec

gem "html-proofer", "~> 5.0", group: :test

platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.2.0", :platforms => [:mingw, :x64_mingw, :mswin]

gem 'jekyll-compose', group: [:jekyll_plugins]

gem "jekyll-github-alerts", "~> 0.1.0"

gem "jekyll-octicons", "~> 19.8"

gem "jekyll-wikirefs", "~> 0.0.16"
