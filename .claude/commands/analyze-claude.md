---
description: Analyze a claude.md file against best practices
argument-hint: <path-to-claude.md>
---

Read and analyze the claude.md file at: $ARGUMENTS

Evaluate it against the following best practices and provide specific, actionable feedback:

# The golden rule: ruthless brevity
A bloated, verbose file doesn't just increase costs—it introduces noise that degrades Claude's ability to follow instructions effectively. This fundamental principle should guide every line you write.

The most successful implementations typically range from 150-200 lines for standard projects, focusing exclusively on information Claude needs to perform its work. This means eliminating redundant explanations (if a folder is named "components," don't explain it contains components), avoiding narrative paragraphs in favor of crisp bullet points, and ruthlessly pruning nice-to-have commentary. Front-loading complete, concise context proves far more effective than forcing Claude to read files randomly, which can "poison" the context with irrelevant information.

# Writing style that Claude understands best
Use short, declarative bullet points rather than narrative paragraphs. Be specific and explicit, never vague or abstract. Front-load critical information in each section. Use technical terminology to prime Claude's knowledge effectively—this helps activate relevant training data. Include concrete code examples showing both correct patterns and anti-patterns.

Separate good versus bad examples clearly. Show what to do and what not to do with explicit labels. For instance:
"Component Pattern—Good Example: Use TypeScript interfaces with explicit prop types. Bad Example: const Button = (props) => {...} // Missing types, unclear contract."

Avoid hedging language like "might," "could potentially," or "perhaps" unless genuine uncertainty exists. Skip excessive explanation of obvious concepts. Your tone should be direct and imperative ("Use TypeScript for all new files") rather than suggestive ("you might want to consider TypeScript"). Be authoritative, not tentative—use "MUST" instead of "should consider" for non-negotiable requirements.

Analysis across community examples shows that code examples dramatically improve adherence. Rather than stating "follow React best practices," show the exact pattern you want. Include interface definitions, type annotations, function structures, and error handling approaches. The more concrete your examples, the more consistent Claude's output becomes.

# Understanding the instruction hierarchy
Claude Code treats instructions with strict priority: claude.md content functions as immutable system rules with superior adherence compared to user prompts, which are interpreted as flexible requests that must work within the established framework. This hierarchical approach ensures consistent behavior throughout sessions, preventing conversational drift where Claude gradually forgets critical constraints.

Claude.md files support an import system using @path/to/import syntax, allowing you to reference other documentation like README files or package.json for available commands. This enables modular organization while preventing token overflow, with a maximum import depth of 5 hops to maintain performance.

# Essential content: what every claude.md needs
Every effective claude.md file should establish your project's technical foundation immediately. Declare your tech stack with specific versions—Framework: Next.js 14, Language: TypeScript 5.2, Styling: Tailwind CSS 3.4—to prevent guesswork and version mismatches. Document your project structure concisely, outlining key directories and their roles without over-explaining obvious organization.

Development commands form the practical backbone of your configuration. List critical npm, bash, or shell scripts including build, test, lint, and deployment commands. This prevents Claude from guessing commands and failing. For example: "npm run dev: Start development server, npm test: Run all unit tests, npm run build: Production build."

Code style and conventions require explicit documentation. Specify formatting guidelines, naming conventions, and import/export syntax preferences. Rather than vague directives like "write good code," provide specific, actionable rules: "Use React.FC type annotations for ALL components, prefer ES modules (import/export) over CommonJS, destructure imports when possible."

File boundaries represent one of the most critical yet overlooked aspects. Explicitly define which files Claude can read and which are forbidden. This prevents context contamination from irrelevant or sensitive files. Example: "Safe to edit: /app/, /tests/, /components/ | Never touch: /venv/, /node_modules/, /.env, /dist/." Many developers skip this section and later struggle with Claude reading inappropriate files.

Repository etiquette and workflow procedures keep development organized. Document branch naming conventions (like "claude/feature-name"), commit message formats (such as Conventional Commits), merge versus rebase preferences, and step-by-step processes for common tasks. Include testing requirements and review procedures before code submission.

# Structural patterns that work
Analysis of 50+ real-world implementations reveals consistent organizational patterns. The most effective files follow a clear hierarchy using Markdown headings: H1 for the project title, H2 for major sections like Critical Rules and Project Context, H3 for subsections like Tech Stack and Architecture. This structure prevents "instruction bleeding" where guidance from one area unintentionally affects another.

Leading with critical rules proves highly effective. Position your most important constraints at the top in a dedicated section with clear emphasis: "CRITICAL RULES: NEVER use any types without explicit approval, MUST run lint and typecheck before completing tasks, NEVER commit directly to main branch." This front-loading ensures Claude sees your non-negotiable requirements first, before processing detailed implementation guidance.

Many sophisticated implementations use XML-style tags for structured sections when dealing with complex contexts. Tags like <system_context>, <paved_path>, <patterns>, and <workflow> create clear boundaries that help Claude parse different types of information. The "paved path" concept—documenting battle-tested canonical patterns that MUST be followed—appears frequently in mature configurations.

Modular organization across multiple files becomes necessary for larger projects. Root claude.md defines WHEN to use patterns, while subdirectory files define HOW to implement them. Teams working with monorepos benefit from hierarchical claude.md files at different levels: parent directories for shared conventions, subdirectories for component-specific rules. Advanced setups create specialized agent files in .claude/agents/ for tasks like memory synchronization, code searching, or design guidance.

# Common pitfalls that sabotage effectiveness
Verbosity and bloat rank as the most common mistake. Developers add extensive content without testing effectiveness, creating files that waste tokens, increase costs, and reduce adherence. The fix: keep concise, test iteratively, and remove what doesn't improve results. Each line should justify its token cost through improved outcomes.

Vague or generic instructions fail to guide Claude meaningfully. Directives like "follow best practices" or "write clean code" provide no actionable guidance. Instead, specify "Use async/await for all asynchronous operations, never use .then() chains" or "All database queries must use parameterized statements to prevent SQL injection."

Conflicting requirements confuse Claude and reduce reliability. When different sections contradict each other—for example, one section requiring TypeScript strict mode while another suggests any types are acceptable—Claude struggles to determine priority. Review your entire file for consistency before saving, ensuring all guidance aligns toward coherent standards.

Not iterating on effectiveness represents a strategic failure. Many developers create claude.md once and never refine it based on actual results. Treat your configuration like a frequently-used prompt that requires experimentation. Add instructions during sessions using the # keyboard shortcut, run prompts through improvers occasionally, and adjust based on Claude's adherence patterns. The best claude.md files evolve continuously with the project.

Missing anti-patterns documentation means documenting only what TO do without explicitly stating what NOT to do. Claude benefits tremendously from seeing wrong patterns with explanations of why they're problematic alongside correct alternatives. For example: "Anti-pattern: N+1 queries common in user dashboard (queries database for each item in loop). Correct: Use includes for related data to batch load."

Context overload from dumping entire documentation into claude.md crowds the context window and degrades performance. Instead, condense to essential information and use smart retrieval for extensive docs. Similarly, ignoring file boundaries and letting Claude read any file indiscriminately pollutes context with irrelevant information. Not using hierarchical structure creates a monolithic file when subdirectory-specific files would keep relevant information properly localized.

---

After analyzing the file, provide:

1. **Overall Assessment**: Line count, estimated token usage, general quality rating
2. **Strengths**: What's working well
3. **Critical Issues**: Problems that severely impact effectiveness (verbosity, vagueness, conflicts)
4. **Recommendations**: Specific, actionable improvements prioritized by impact
5. **Rewrite Suggestions**: For any sections that need significant improvement, show before/after examples

Be direct and specific. Focus on actionable feedback that will measurably improve Claude's adherence and reduce token waste.