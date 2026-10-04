# Scout and Alma

**Live URL**: https://www.scoutandalma.com

## Project Overview

Scout and Alma is a dual-sided higher education discovery and student recruitment platform engineered for universities, apprenticeship employers, and prospective candidates. The platform delivers two distinct, role-segregated applications from a shared workspace environment:

* **Scout (fit-shortlist)** serves prospective students as an agentic AI admissions advisor. Scout engages students through structured conversational intake, parses uploaded resumes and academic transcripts, creates structured search briefs, and recommends degree programs and apprenticeships categorized into personalized shortlist tiers.


* **Alma (prospect-radar)** serves university admissions departments and enterprise apprenticeship recruiters. Alma provides candidate radar filters, prospect discovery tools, interaction tracking, and direct application invitations for students who make their profiles visible.


* **Workspace Extensions** provide supplementary capabilities including an application task tracker (tasks-db), research surveys for market discovery (student-survey and university-survey), survey response dashboards (survey-responses), and administrative reporting panels (student-analytics and alma-data-panel).



## Architecture Explanation

The system is organized around a client-side desktop environment that interfaces with workspace proxy backends, an isolated data plane, and multi-model artificial intelligence engines.

### Desktop Workspace Shell and Windowing Runtime

The primary container is the SpaceDesktop component, which implements an operating system model inside the web browser.

* **Responsive Viewport Modes**: The shell detects viewport dimensions using JavaScript matchMedia listeners to prevent hydration mismatches and double-mounting. Desktop displays present a floating application dock, an application launcher grid, and side-by-side split panels for simultaneous sub-application use and assistant chat. Mobile viewports collapse into a single stacked view with bottom dock navigation, accompanied by body scroll locks and animation frame repaints to prevent layout distortion during mobile keyboard entry.


* **Window Orchestration and State Synchronization**: The workspace orchestrator manages transitions between active sub-applications, the AI chat assistant, a persistent memory file browser, and system settings. Window state updates synchronize with the browser history stack and URL hash parameters using pushState and popState listeners. A versioned cache-busting mechanism forces page reloads when browsers attempt to restore stale pages from back-forward cache.


* **Fault Isolation**: Every mounted application component is isolated within an AppErrorBoundary boundary. Component crashes in an individual sub-app render a localized failure notification and retry handler, preventing workspace shell crashes.



### Role-Based Access Control and Deep Linking

Access control is enforced at the shell layer to isolate student and institution workspaces:

* **Role Isolation Matrix**: User roles are student, university, and company. Students can only view and access the fit-shortlist application, while universities and companies can only access prospect-radar.


* **URL and Hash Gating**: When a user arrives through an external link or modifies the URL hash, the navigation router evaluates the target against the role permission matrix. Unpermitted route requests are blocked and redirected to the authorized default application.


* **Public Survey Bypass Pipeline**: Public entry points, such as student-survey and university-survey, are marked with public bypass flags. These bypass routes skip authentication barriers, provision random ephemeral session tokens, and present full-viewport views with all desktop controls hidden.



### Session Management and Authentication Lifecycle

User identity and workspace access are governed by multi-tier session state managers:

* **EmailGate Entry**: Unauthenticated customer visits land on the EmailGate view. Users choose their identity type and submit their email address, which is validated using one-time password verification or session token recovery before private data access is granted.


* **Cross-Component Event Bus**: When identity is established or cleared, the runtime broadcasts custom browser events including audos:session-established and audos:session-cleared. These events synchronize session identifiers across disparate modules without component unmounting.


* **Cross-Domain Visitor Telemetry**: The client writes a persistent tracking cookie named audos_vid. The script inspects the host domain against a list of second-level country code domains and cloud platform suffixes, ensuring cookies are scoped across relevant subdomains without triggering browser tracking blocks.


* **Post-Checkout Auto-Bootstrap**: When users return from Stripe billing checkout with payment indicators, the runtime intercepts the session query parameters, queries payment status endpoints, registers an authenticated customer session automatically, and launches the desktop environment.



### Data Plane and Persistence Engine

Data operations are handled by the WorkspaceDB client library, which interacts with dedicated PostgreSQL database tables per workspace:

* **Authorization and Session Headers**: The SDK auto-injects an authorization token using the X-Workspace-DB-Token header and includes the active user session inside the X-Session-Id header.


* **Fluent Query Builder**: The SDK offers a chainable builder supporting column selection, equality and range operators, ordering, limits, offsets, bulk inserts, updates, record counts, and mathematical aggregations.


* **Private and Shared Multi-Tenancy**: Private rows are automatically tagged with the current user session ID so students can only inspect their own records. Queries marked with the shared flag access shared workspace tables, allowing institutions to inspect shared applicant directories.


* **Reactive React Integration (useWorkspaceDB)**: A specialized data hook binds component rendering to database queries. The hook manages request epoch counters to discard out-of-order asynchronous responses, invalidates superseded completions before screen paints, and isolates cached row projections during rapid session handoffs.


* **Schema Design**: Dedicated tables manage domain state, including scout_programs for academic degree offerings, scout_program_events for program state change logs, scout_messages for conversational records, scout_documents for uploaded student files, scout_state_snapshots for full session recovery, and scout_application_requests for institutional outreach.



### AI Advisory and Processing Pipeline

Scout functions through an agentic conversational engine that extracts profile facts, constructs search profiles, and dispatches UI commands:

* **Conversational Intake Flow**: Scout moves candidates through an ordered intake checklist covering target subject areas, undergraduate or postgraduate levels, higher education motivations, location preferences, tuition budget, test results, citizenship residency status, desired outcomes, program priorities, and recruiter visibility.


* **Search Brief Fit Bucketing**: Unstructured conversational answers are parsed into a multi-factor search brief covering majors, rankings, locations, budgets, and post-study career roles. Each factor organizes preferences into four buckets: excellent, good, borderline, and not a fit. Verbal answers are condensed into concise title-case keyword chips while ignoring conversational filler.


* **Multi-Modal Document Parsing**: Uploaded student resumes and transcripts are sent to a server-side document analysis pipeline powered by vision and document processing endpoints. Extracted details populate structured profile fields for previous institutions, degrees, grades, employment histories, publications, technical skills, and extracurriculars, triggering automated bio generation.


* **Action Execution Engine**: Scout generates responses as structured JSON objects containing message text and executable actions. The client-side runtime executes actions for saving intake answers, modifying search brief chips, updating candidate profile objects, searching programs, and moving programs between shortlist columns. Shortlist modifications are committed to the database before confirmation messages are presented.



### Live Program Search and Sourcing Engine

Once candidate preferences are established, the recommendation pipeline finds and validates matching offerings:

* **Targeted Domain Sourcing**: The search service prioritizes verified university hosts (.edu, .ac.uk, and accredited institutional websites), discarding aggregator portals and outdated course directories.


* **Fact Extraction and Sourced Evidence**: Scout fetches and inspects official program pages directly to capture published tuition numbers, admission deadlines, standardized test criteria, and degree lengths. Every fact is bound strictly to evidence discovered in the search snippets.


* **Dynamic Preference Re-Evaluation**: When candidates change their target degree level, geographical parameters, or annual tuition thresholds, the active program list is re-evaluated. Previously recommended courses that violate the new constraints are automatically updated with explanation reasons and transferred into the skipped column.



## Technical Stack

* **Frontend Framework**: React 18 using functional components, lazy-loaded sub-applications, and custom context providers.


* **Programming Language**: TypeScript providing type contracts for workspace configurations, theme tokens, and data models.


* **Styling and Layout**: Tailwind CSS paired with dynamic CSS variables, theme recipe tokens, and Chakra Petch typography.


* **Animation and Graphics**: Framer Motion for desktop panel transitions, Rive Canvas for vector animation playback, and Lucide React for icon sets.


* **Data Visualization and Document Generation**: Recharts for analytical graphics and jsPDF for client-side document exports.


* **Database and SDK**: WorkspaceDB client SDK querying partitioned PostgreSQL tables via REST endpoints.


* **AI and Natural Language Processing**: Anthropic Claude (claude-sonnet-5) streaming messages via platform proxies, OpenAI GPT models for structured schema extractions, and Google Gemini for document parsing and vision tasks.


* **External Integrations**: Stripe for checkout workflows, Google Cloud Storage for asset persistence, and Mailgun for email notifications.


* **Telemetry and Pixels**: Google Tag Manager, custom funnel tracking hooks, Meta Pixel, and Reddit Pixel conversion dispatchers.
