# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture Overview

This is a personal blog website built with Astro and TypeScript, optimized for static site generation and Cloudflare Pages deployment. The architecture follows modern web development practices:

- **Static Site Generation**: Astro pre-renders all pages at build time for optimal performance
- **TypeScript**: Full TypeScript support for type safety and better developer experience
- **Content Collections**: Astro's content collections provide type-safe blog post management
- **Responsive Design**: Mobile-first approach with modern CSS
- **Performance Optimized**: Zero JavaScript by default, with selective hydration when needed

## Development Workflow

### Essential Commands
- `pnpm dev` - Start development server with hot reload
- `pnpm build` - Build the static site for production
- `pnpm preview` - Preview the production build locally
- `pnpm deploy` - Deploy to Cloudflare Pages production (requires wrangler auth)
- `pnpm deploy:dev` - Deploy to Cloudflare Pages development (requires wrangler auth)

### Development Process
1. Always run `pnpm build` before deployment to ensure everything compiles correctly
2. Use TypeScript for all new code - the project has strict type checking enabled
3. Blog posts are written in Markdown and stored in `src/content/blog/`
4. The build process generates optimized static files in the `dist/` directory

## Code Organization

### Content Management
- **Blog Posts**: Located in `/src/content/blog/` as Markdown files
- **Content Schema**: Defined in `/src/content.config.ts` with TypeScript validation
- **Static Assets**: Images and other assets in `/src/assets/` and `/public/`

### Component Structure
- **Layouts**: Reusable page layouts in `/src/layouts/`
- **Components**: Astro components in `/src/components/`
- **Pages**: Route-based pages in `/src/pages/`
- **Styles**: Global CSS in `/src/styles/global.css`

### Blog Post Format
Blog posts should include frontmatter with:
```yaml
---
title: 'Post Title'
description: 'Post description for SEO'
pubDate: 2024-01-15
heroImage: '/blog-placeholder-1.jpg' # Optional
---
```

## Deployment Configuration

### Cloudflare Pages
- `wrangler.toml` configures deployment to Cloudflare Pages with multiple environments
- Production environment: `website`
- Development environment: `website-dev`
- Build command: `pnpm build`
- Output directory: `dist/`
- Production URL: https://website.pages.dev
- Development URL: https://website-dev.pages.dev

### GitHub Actions CI/CD
- `.github/workflows/deploy-production.yml` - Deploys to production on main branch pushes
- `.github/workflows/deploy-development.yml` - Deploys to development on any branch pushes (except main) and PRs
- Requires GitHub secret: `CLOUDFLARE_API_TOKEN`
- Manual deployments require confirmation for both environments

### Build Process
1. TypeScript compilation with strict type checking
2. Astro static site generation
3. Automatic optimization of images, CSS, and JavaScript
4. Generation of sitemap and RSS feed

## TypeScript Configuration

The project uses strict TypeScript settings:
- `strict: true` for maximum type safety
- Content collections provide type safety for blog post frontmatter
- All components and pages use TypeScript by default

## Content Guidelines

### Blog Posts
- Focus on web development, TypeScript, performance, and technology topics
- Use clear, technical writing with code examples
- Include hero images for visual appeal
- Add proper SEO descriptions in frontmatter

### Code Style
- Use TypeScript for all new code
- Follow Astro component conventions
- Keep components small and focused
- Use semantic HTML and accessible design patterns

## Performance Considerations

- Astro ships zero JavaScript by default for optimal performance
- Images are automatically optimized during build
- CSS is automatically bundled and minified
- Use the `client:*` directives only when interactivity is needed
