# Agent Instructions for Website Project

## Build/Lint/Test Commands
- **Development**: `bun dev` - Start development server with hot reload
- **Build**: `bun run build` - Build static site for production
- **Preview**: `bun preview` - Preview production build locally
- **Deploy Production**: `bun run deploy` - Deploy to Cloudflare Pages production
- **Deploy Development**: `bun run deploy:dev` - Deploy to Cloudflare Pages development
- **No dedicated lint/test commands** - Use `bun run build` to validate TypeScript compilation

## Code Style Guidelines

### TypeScript & JavaScript
- Use strict TypeScript with `strictNullChecks: true`
- Prefer `const` over `let`, avoid `var`
- Use arrow functions for anonymous functions
- Single quotes for string literals
- Semicolons required
- 2-space indentation
- JSDoc comments for function descriptions

### Imports & Exports
- Use ES6 imports/exports
- Group imports: built-ins, external libraries, then local imports
- Use relative paths for local imports (`../`)
- Import types explicitly when needed

### Naming Conventions
- **Components**: PascalCase (e.g., `ThemeToggle.astro`)
- **Constants**: SCREAMING_SNAKE_CASE (e.g., `SITE_TITLE`)
- **CSS Classes**: kebab-case (e.g., `theme-toggle`)
- **Variables/Functions**: camelCase
- **Files**: PascalCase for components, camelCase for utilities

### Astro Components
- Use frontmatter (`---`) for imports and logic
- Keep component logic minimal and focused
- Use `<style>` blocks for component-scoped CSS
- Prefer server-side rendering over client-side JavaScript

### CSS & Styling
- Use CSS custom properties for theming (`var(--variable)`)
- Follow mobile-first responsive design
- Use semantic class names
- Include accessibility features (focus states, ARIA labels)

### Content Management
- Blog posts in `src/content/blog/` as Markdown/MDX
- Use defined schema in `content.config.ts` for type safety
- Include required frontmatter: `title`, `description`, `pubDate`

### Error Handling
- Use try-catch for async operations
- Validate user input and API responses
- Provide meaningful error messages to users

### Performance
- Minimize client-side JavaScript
- Use Astro's built-in image optimization
- Leverage static site generation for optimal performance