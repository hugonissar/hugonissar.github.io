source "https://rubygems.org"

# Latest Jekyll (not the older GitHub-Pages-bundled version, since we control
# the build via GitHub Actions and deploy the static output ourselves).
gem "jekyll", "~> 4.3"

# Required for the bundler/runtime on Ruby 3.x
gem "csv"
gem "base64"
gem "logger"
gem "bigdecimal"

# SEO + content plugins (referenced in _config.yml)
group :jekyll_plugins do
  gem "jekyll-feed",          "~> 0.17"
  gem "jekyll-sitemap",       "~> 1.4"
  gem "jekyll-seo-tag",       "~> 2.8"
  gem "jekyll-paginate",      "~> 1.1"
  gem "jekyll-redirect-from", "~> 0.16"
  gem "jekyll-include-cache", "~> 0.2"
end

# Windows + JRuby support (harmless on Linux runners)
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]
