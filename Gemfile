source "https://rubygems.org"
gem "jekyll", "~> 4.4"
gem "jekyll-remote-theme", "~> 0.5"
gem "jekyll-feed"
gem "jekyll-seo-tag"
gem "jekyll-paginate"
# Runtime dep of the no-style-please theme gemspec (required via
# require_theme_deps). Loaded at startup so its after_init hook runs before
# pre_render; not whitelisted on GitHub Pages, so inert there.
group :jekyll_plugins do
  gem "jektex", "~> 0.2.0"
end
