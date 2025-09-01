# frozen_string_literal: true
source "https://rubygems.org"

gem "github-pages", "~> 232", group: :jekyll_plugins

gem "jekyll-remote-theme"
gem "jekyll-include-cache"

gem "webrick", "~> 1.8"

group :jekyll_plugins do
  gem "jekyll-archives", "~> 2.2"
  gem "jekyll-paginate", "~> 1.1"
  gem "jekyll-seo-tag", "~> 2.8"
  gem "jekyll-sitemap", "~> 1.4"
  gem "jekyll-feed", "~> 0.17"
end

platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end
gem "wdm", "~> 0.2.0", platforms: [:mingw, :x64_mingw, :mswin]
