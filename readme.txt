1. Do NOT rename this repo. This to create our plabble.github.io domain
2. To run locally, follow these steps:
    - a. Install Ruby and base-devel. Make sure on your distro ruby ships with RubyGems (gem)
    - b. Add ~/.local/share/gem/ruby/x.x.x/bin to your PATH
    - c. Install Bundler with: gem install bundler
    - c2. You might need to chown /usr/lib/ruby and /usr/bin or repeat steps with root accessed
    - d. Go to this directory and run: bundle install
    - e. To start the dev server, run bundle exec jekyll serve. If this does not work, it might still have the _site directory with the static files
