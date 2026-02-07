# mitchscobell.github.io

Personal blog and portfolio website built with Jekyll and hosted on GitHub Pages.

This project is [forked from jekyll-now](https://github.com/barryclark/jekyll-now) and uses the [jekyll-theme-hacker](https://github.com/pages-themes/hacker) remote theme.

## About This Site

This is a personal website and blog featuring posts about software development, cloud architecture, and technology. The site includes:

- Blog posts written in Markdown
- An about page with professional background
- Custom styling and layout
- Integration with social media links

## Prerequisites

Before running this project locally, you need to have the following installed:

- **Ruby** (version 2.5 or higher recommended)
- **RubyGems** (comes with Ruby)
- **Bundler** gem
- **Jekyll** gem
- **Build tools** (for native extensions)

## Installation

### Ubuntu/Debian/WSL

```bash
# Update system packages
sudo apt-get update -y && sudo apt-get upgrade -y

# Install Ruby and build tools
sudo apt-get install -y ruby-full build-essential zlib1g-dev

# Install Bundler and Jekyll
gem install bundler jekyll

# If you encounter permission issues, see the Troubleshooting section below for solutions
```

### macOS

```bash
# Using Homebrew
brew install ruby

# Install Bundler and Jekyll
gem install bundler jekyll
```

### Windows

1. Download and install Ruby+Devkit from [RubyInstaller](https://rubyinstaller.org/)
2. Run the `ridk install` step at the end of installation
3. Open a new command prompt and run:

```bash
gem install bundler jekyll
```

## Running Locally

1. **Clone the repository** (if you haven't already):

   ```bash
   git clone https://github.com/mitchscobell/mitchscobell.github.io.git
   cd mitchscobell.github.io
   ```

2. **Serve the site locally**:

   ```bash
   jekyll serve
   ```

   Or with live reload and detailed logging:

   ```bash
   jekyll serve --livereload --trace
   ```

3. **View in browser**:
   Open [http://localhost:4000](http://localhost:4000) or [http://127.0.0.1:4000](http://127.0.0.1:4000)

The site will automatically regenerate when you make changes to files.

## Building the Site

To build the site without serving it:

```bash
jekyll build
```

This generates the static site in the `_site` directory.

### Build Options

- **Build with drafts**: `jekyll build --drafts`
- **Build for production**: `JEKYLL_ENV=production jekyll build`
- **Clean build**: `jekyll clean && jekyll build`

## Project Structure

```
.
├── _config.yml          # Site configuration
├── _includes/           # Reusable HTML components
├── _layouts/            # Page templates
├── _posts/              # Blog posts (Markdown)
├── _sass/               # Sass stylesheets
├── images/              # Image assets
├── about.md             # About page
├── index.html           # Home page
├── style.scss           # Main stylesheet
└── 404.md              # 404 error page
```

## Writing Blog Posts

To create a new blog post:

1. Create a new file in the `_posts/` directory
2. Name it using the format: `YYYY-MM-DD-title.md`
3. Add front matter at the top:

   ```markdown
   ---
   layout: post
   title: Your Post Title
   ---

   Your content here...
   ```

See `_posts/_template.md` for a template.

## Deployment

This site is automatically deployed to GitHub Pages when changes are pushed to the main branch. GitHub Pages has built-in support for Jekyll, so no manual build step or custom GitHub Actions workflow is required for deployment.

The site is available at: [https://mitchscobell.com](https://mitchscobell.com)

GitHub Pages automatically:

- Builds the Jekyll site
- Deploys the generated static files
- Serves the content at the configured domain

## Configuration

Main configuration is in `_config.yml`. You can customize:

- Site name and description
- Social media links
- Google Analytics tracking
- Theme settings
- Permalink structure

## Troubleshooting

### Permission Issues (WSL/Linux)

If you encounter permission errors when running Jekyll, you have a few options:

1. **Configure gem installation path** (recommended):

   ```bash
   echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
   echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
   source ~/.bashrc
   gem install bundler jekyll
   ```

2. **Use sudo** (not recommended, but works):
   ```bash
   sudo jekyll serve --trace
   ```

### Port Already in Use

If port 4000 is already in use:

```bash
jekyll serve --port 4001
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
