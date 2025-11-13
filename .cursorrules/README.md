---
description: Development rules and best practices
globs: []
alwaysApply: true
---

# Development Rules and Best Practices

This directory contains development guidelines and best practices in the modern Cursor Rules format (MDC). These rules are automatically applied based on file patterns and provide context-aware guidance to AI assistants.

## Rules Overview

### Core Guidelines
- **architecture.mdc** - Project overview, technology stack, and infrastructure setup (Always Applied)
- **README.md** - This file - overview of all development rules (Always Applied)

### Code Quality
- **typescript-guidelines.mdc** - TypeScript best practices and conventions (Auto-attached to `.ts`/`.tsx` files)
- **code-style.mdc** - General coding standards and style guide (Auto-attached to code files)
- **file-structure.mdc** - File and directory organization patterns (Auto-attached to config files)

### Framework Development
- **react-guidelines.mdc** - React development principles and patterns (Auto-attached to `.tsx`/`.jsx` files)

### Testing & Quality
- **testing-guidelines.mdc** - Testing strategies and best practices (Auto-attached to test files)

## How Rules Work

### Automatic Attachment
Rules are automatically included in your AI context based on file patterns (globs). When you work on TypeScript files, the TypeScript guidelines are automatically loaded.

### Manual Reference
You can manually reference any rule using the `@ruleName` syntax:
- `@architecture` - Include architecture overview and tech stack
- `@typescript-guidelines` - Load TypeScript best practices
- `@react-guidelines` - Get React development patterns
- `@testing-guidelines` - Include testing recommendations
- `@code-style` - Reference coding standards
- `@file-structure` - Get file organization guidance

### Rule Types Used
- **Always Applied** - Loaded in every context (architecture.mdc, README.md)  
- **Auto Attached** - Loaded when matching file patterns are referenced
- **Agent Requested** - Available for AI to include when relevant
- **Manual** - Only included when explicitly mentioned

## Development Workflow

### Getting Started
1. Clone the repository
2. Install dependencies: `npm install` (or `yarn install` / `pnpm install`)
3. Copy `.env.example` to `.env` and configure environment variables
4. Start development server: `npm run dev`

### Common Commands
```bash
# Development
npm run dev              # Start development server
npm run build            # Build for production
npm run preview          # Preview production build

# Code Quality
npm run lint             # Run linter
npm run lint:fix         # Fix linting issues
npm run typecheck        # TypeScript type checking
npm run format           # Format code with Prettier

# Testing
npm test                 # Run tests
npm run test:watch       # Run tests in watch mode
npm run test:coverage    # Generate coverage report
npm run test:e2e         # Run end-to-end tests

# Utilities
npm run clean            # Clean build artifacts
npm run validate         # Run all checks (lint, typecheck, test)
```

## Project Structure

```
project-root/
├── .cursorrules/          # Cursor AI development rules
├── .github/               # GitHub workflows and templates
├── src/                   # Source code
│   ├── components/        # React components
│   ├── pages/            # Page components
│   ├── hooks/            # Custom React hooks
│   ├── services/         # Business logic services
│   ├── utils/            # Utility functions
│   ├── types/            # TypeScript types
│   ├── styles/           # Global styles
│   └── config/           # App configuration
├── tests/                 # Test files
├── public/                # Static assets
├── docs/                  # Documentation
├── scripts/               # Build and utility scripts
├── .env.example           # Environment variables template
├── package.json           # Package configuration
├── tsconfig.json          # TypeScript configuration
├── vite.config.ts         # Vite configuration
└── README.md              # Project overview
```

## Code Standards

### Naming Conventions
- **Variables & Functions**: `camelCase`
- **Classes & Interfaces**: `PascalCase`
- **Constants**: `UPPER_SNAKE_CASE`
- **Files & Directories**: `kebab-case`
- **Components**: `PascalCase.tsx`

### TypeScript
- Enable strict mode
- Avoid `any` type - use `unknown` or proper types
- Always type function parameters and return types
- Use interfaces for object shapes
- Use type for unions and intersections

### React
- Use functional components with hooks
- Keep components focused and single-purpose
- Use TypeScript for props
- Implement proper error boundaries
- Optimize with `memo`, `useMemo`, `useCallback` when needed

### Testing
- Write tests for all business logic
- Test user behavior, not implementation
- Use React Testing Library for components
- Maintain 80%+ code coverage
- Keep tests fast and reliable

## Git Workflow

### Branch Strategy
- `main` - Production-ready code (protected)
- `develop` - Integration branch
- `feature/*` - New features
- `fix/*` - Bug fixes
- `hotfix/*` - Production hotfixes

### Commit Conventions
Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add user authentication
fix: resolve login redirect issue
docs: update API documentation
refactor: simplify user service logic
test: add tests for payment flow
chore: update dependencies
```

### Pull Request Process
1. Create feature branch from `develop`
2. Implement changes with tests
3. Run all checks: `npm run validate`
4. Create PR with clear description
5. Address review feedback
6. Merge after approval and passing CI

## Code Review Guidelines

### For Authors
- Keep PRs focused and reasonably sized
- Write clear PR descriptions
- Add tests for new functionality
- Update documentation as needed
- Respond to feedback promptly

### For Reviewers
- Review within 24 hours when possible
- Focus on logic, design, and maintainability
- Be constructive and respectful
- Ask questions to understand intent
- Approve when changes meet standards

## Usage Guidelines

### For Developers
- Rules are automatically applied based on file context
- Check rule descriptions to understand when they're activated
- Use manual references (`@ruleName`) for additional context
- Keep rules updated as the codebase evolves
- Follow the established patterns consistently

### For AI Assistants
- Rules provide consistent guidance across conversations
- Use rule context to maintain coding standards
- Reference specific rules when making recommendations
- Apply rule principles in code suggestions and reviews
- Adapt recommendations to project-specific needs

## Contributing to Rules

### Adding New Rules
1. Create a new `.mdc` file in this directory
2. Include proper metadata headers with description and globs
3. Write clear, actionable guidelines with examples
4. Test the rule with relevant file patterns
5. Update this README with the new rule
6. Create a PR for review

### Updating Existing Rules
1. Modify the rule content while preserving metadata
2. Test changes with affected file patterns
3. Ensure consistency with other rules
4. Update examples and best practices as needed
5. Document changes in PR description

### Rule File Template
```markdown
---
description: Brief description of the rule's purpose
globs: ["**/*.ts", "**/*.tsx"]  # File patterns for auto-attachment
alwaysApply: false              # Whether to always include this rule
---

# Rule Title

## Overview
Brief explanation of what this rule covers

## Guidelines

### Subsection
- Guideline point
- Another guideline

### Examples
\`\`\`typescript
// Good example
const example = () => {};

// Bad example  
const bad = () => {};
\`\`\`

## Best Practices
1. Practice one
2. Practice two
```

## Rule Format Reference

Each rule file uses the MDC format with metadata:

```markdown
---
description: Brief description of the rule's purpose
globs: ["**/*.ts", "**/*.tsx"]  # File patterns for auto-attachment
alwaysApply: false              # Whether to always include this rule
---

# Rule Title

Rule content in Markdown format...
```

### Metadata Fields

#### description (required)
- Brief description of what the rule covers
- Shown in rule listings and selections
- Should be clear and concise (one sentence)

#### globs (required)
- Array of glob patterns for automatic attachment
- Patterns match against file paths
- Use `[]` for no automatic attachment
- Common patterns:
  - `**/*.ts` - All TypeScript files
  - `**/*.tsx` - All React TypeScript files
  - `**/*.test.ts` - All test files
  - `**/package.json` - Package configuration files

#### alwaysApply (required)
- Boolean indicating if rule should always be included
- Set to `true` for core rules (architecture, README)
- Set to `false` for context-specific rules
- Use sparingly to avoid context overload

## Best Practices for Rules

### Writing Effective Rules
1. **Be specific** - Provide concrete examples
2. **Be consistent** - Align with other rules
3. **Be practical** - Focus on real-world scenarios
4. **Be concise** - Keep explanations clear and brief
5. **Be current** - Update rules as practices evolve

### Rule Organization
1. **Group related content** - Use logical sections
2. **Prioritize important info** - Put key points first
3. **Use examples** - Show good and bad patterns
4. **Link related rules** - Reference other relevant rules
5. **Include rationale** - Explain why, not just what

### Maintaining Rules
1. **Regular reviews** - Audit rules quarterly
2. **Team feedback** - Gather input from developers
3. **Version with code** - Keep rules in sync with codebase
4. **Document changes** - Track rule evolution
5. **Remove outdated content** - Keep rules current

## Troubleshooting

### Rules Not Loading
- Check glob patterns match your file paths
- Verify metadata syntax is correct
- Ensure `.cursorrules` directory is in workspace root
- Try reloading the Cursor window

### Conflicting Guidelines
- Follow the most specific rule for your context
- Consult the team for clarification
- Propose rule updates if conflicts persist
- Document exceptions in code comments

### Performance Issues
- Limit number of "alwaysApply" rules
- Keep rule files reasonably sized
- Use specific glob patterns
- Remove unused rules

## Additional Resources

### Documentation
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
- [React Documentation](https://react.dev/)
- [Testing Library Docs](https://testing-library.com/)
- [Conventional Commits](https://www.conventionalcommits.org/)

### Tools
- [ESLint](https://eslint.org/) - Code linting
- [Prettier](https://prettier.io/) - Code formatting
- [Vitest](https://vitest.dev/) - Testing framework
- [TypeScript](https://www.typescriptlang.org/) - Type checking

### Internal Resources
- Architecture Decision Records (ADRs) - `docs/architecture/adr/`
- API Documentation - `docs/api/`
- Setup Guide - `docs/guides/setup.md`
- Deployment Guide - `docs/guides/deployment.md`

## Feedback and Support

### Getting Help
- Check existing documentation first
- Ask in team chat or meetings
- Review related code examples
- Consult with senior developers

### Providing Feedback
- Suggest rule improvements via PR
- Report issues with existing rules
- Share successful patterns
- Contribute examples and clarifications

### Contact
- **Tech Lead**: [Name/Email]
- **Team Channel**: [Slack/Discord channel]
- **Documentation**: [Wiki/Confluence link]
- **Repository**: [GitHub repository URL]

---

## Summary

These development rules provide a consistent foundation for building high-quality software. By following these guidelines, we ensure:

- **Consistency** - Unified coding standards across the team
- **Quality** - High standards for code, tests, and documentation
- **Efficiency** - Faster development with clear patterns
- **Maintainability** - Code that's easy to understand and modify
- **Collaboration** - Smooth teamwork with shared expectations

Remember: Rules are guidelines, not rigid constraints. Use good judgment, and don't hesitate to propose improvements when you identify better approaches.

For the most up-to-date version of these guidelines, always refer to the files in this directory.

**Last Updated**: 2025-11-13
