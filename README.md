# Pranavj17.github.io

Personal engineering blog. Built with [Jekyll](https://jekyllrb.com/) + minima theme. Hosted via GitHub Pages.

Live at [pranavj17.github.io](https://pranavj17.github.io/).

## Topics

- Production AI infrastructure
- LLM benchmarks on real-world workloads
- Observability engineering
- Elixir / Python / OpenClaw

## Local development

```bash
# One-time setup
gem install bundler jekyll
bundle init
echo 'gem "jekyll", "~> 4.3"' >> Gemfile
echo 'gem "minima", "~> 2.5"' >> Gemfile
echo 'gem "jekyll-feed"' >> Gemfile
echo 'gem "jekyll-seo-tag"' >> Gemfile
echo 'gem "jekyll-sitemap"' >> Gemfile
bundle install

# Serve locally
bundle exec jekyll serve
# → http://localhost:4000
```

## Adding a post

1. Create `_posts/YYYY-MM-DD-slug.md`
2. Add front-matter: `title`, `date`, `categories`, `tags`
3. Commit + push to `main` — GitHub Pages auto-builds within ~1 minute
