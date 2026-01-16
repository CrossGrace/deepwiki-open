# DeepWiki Open Documentation

This directory contains the GitHub Pages documentation for DeepWiki Open.

## Viewing the Documentation

Once GitHub Pages is enabled, the documentation will be available at:
**https://crossgrace.github.io/deepwiki-open**

## Local Preview

To preview the documentation locally:

```bash
# Install Jekyll
gem install bundler jekyll

# Create Gemfile (if not exists)
cat > Gemfile <<EOF
source 'https://rubygems.org'
gem 'github-pages', group: :jekyll_plugins
gem 'webrick'
EOF

# Install dependencies
bundle install

# Serve locally
cd docs
bundle exec jekyll serve

# Open http://localhost:4000/deepwiki-open
```

## GitHub Pages Setup

1. Go to repository **Settings** → **Pages**
2. Set **Source** to "Deploy from a branch"
3. Select branch: `main` (after merging PR)
4. Select folder: `/docs`
5. Click **Save**

GitHub will automatically build and deploy the documentation.

## Structure

```
docs/
├── _config.yml              # Jekyll configuration
├── _layouts/
│   └── default.html         # Custom page layout
├── _includes/
│   └── navigation.html      # Sidebar navigation
├── assets/
│   └── css/
│       └── style.scss       # Custom styles
├── index.md                 # Home page
├── architecture.md          # Architecture documentation
├── usage.md                 # Usage guide
├── api-reference.md         # API documentation
├── configuration.md         # Configuration guide
└── deployment.md            # Deployment guide
```

## Features

- 📱 **Responsive Design**: Mobile-friendly with hamburger menu
- 🎨 **Modern UI**: Professional styling with Inter font
- 🌙 **Dark Mode**: Automatic dark mode support
- 📑 **Sidebar Navigation**: Fixed sidebar with categorized sections
- 🔍 **Code Highlighting**: Syntax highlighting with copy buttons
- ⚡ **Fast Loading**: Optimized CSS and minimal dependencies

## Customization

### Adding a New Page

1. Create a new markdown file in `docs/`:
   ```markdown
   ---
   layout: default
   title: Your Page Title
   description: Page description
   prev_page:
     title: Previous Page
     url: /previous.html
   next_page:
     title: Next Page
     url: /next.html
   ---

   # Your Content Here
   ```

2. Add the page to `_includes/navigation.html`:
   ```html
   <li><a href="{{ '/your-page.html' | relative_url }}">
     <i class="fas fa-icon"></i> Your Page
   </a></li>
   ```

### Modifying Styles

Edit `assets/css/style.scss` to customize:
- Colors (CSS variables in `:root`)
- Typography
- Layout dimensions
- Component styles

### Updating Navigation

Edit `_includes/navigation.html` to:
- Add/remove navigation items
- Change icons (Font Awesome 6)
- Reorganize sections

## Contributing

When adding new documentation:
1. Follow the existing markdown style
2. Update navigation if adding new pages
3. Test locally before committing
4. Ensure mobile responsiveness

## License

Same as the main project (MIT).
