# frozen_string_literal: true

source "https://rubygems.org"

gem "jekyll-theme-chirpy", "~> 7.5"

gem "html-proofer", "~> 5.0", group: :test

platforms :windows, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# wdm gives native file-watching on Windows, but its 0.2.0 C-extension fails to
# build on Ruby 3.3+ (ucrt). It only affects local `jekyll serve --watch` on
# Windows (CI runs on Linux and never installs it), so it is disabled here.
# gem "wdm", "~> 0.2.0", :platforms => [:windows]
