# School Buddy

A responsive, database-backed school workspace for student records, classes, enrollment, daily attendance, fees, examinations, report cards, PTMs, staff, parent communication, reports and school settings.

## What runs

- Persistent CRUD with validated forms, search, filters, pagination, archive actions, and audit logs.
- A dashboard computed from database records, including fee collection, enrollment and attendance charts.
- Student profiles with guardian links, class/session history, attendance, fees, academic reports and private uploads.
- Class/section management, capacity checks, enrollment transfers and academic sessions.
- Fee structures, student concessions, scholarships, generated period invoices, partial/full payments, signed adjustments/refunds, printable receipts, and CSV financial reports.
- Class attendance register with audited changes and duplicate prevention.
- Examination subjects, marks, configured grades, pass/fail calculations and immutable published report cards.
- Timetable creation with class/teacher/room conflict detection.
- PTM events, invitations, confirmations/attendance statuses and private notes.
- Reusable communication templates, class/individual recipient preview, consent, test logs, scheduled queues and configurable provider adapters.
- Secure server-side roles for Super Admin, School Administrator, Teacher, Accountant, and linked Parent/Guardian.
- Light/dark modes, desktop/mobile navigation, downloadable PDFs, print views and CSV exports compatible with Excel.

## Architecture decision

The active deployment uses React/TypeScript + Vinext, Cloudflare D1 (SQLite), R2 private storage, and Sites sign-in. It does **not** use Supabase Auth or PostgreSQL at runtime. See [architecture](docs/ARCHITECTURE.md) for the reason, server permission design and limits. Optional PostgreSQL/Supabase migration scripts are included under `supabase/migrations`; they are not deployed or verified against PostgreSQL.

## Local setup

Requires Node 22.13+ and pnpm 11.25.0 (or a compatible pinned toolchain). Use an official Node/pnpm installation available on PATH.

```sh
pnpm install
pnpm build
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0000_ambiguous_catseye.sql
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0001_integrity.sql
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0002_tenant_guards.sql
pnpm dev
```

Apply each migration only once, in order. Keep `.wrangler/state` for local persistence. Production and local databases are separate. The local URL is `http://127.0.0.1:5173`. Portable development uses an explicitly simulated `seedy@sites.test` sign-in; this mock authentication is absent from production builds.

Sign in, create your school workspace, then add an academic session, classes, sections, students, guardians and enrollments. You can load the optional development dataset only in an empty school. The demo flag is displayed throughout the interface; demo contacts use `example.test` and live sending is blocked. Production never silently seeds records.

## Daily workflows

1. **Student:** Add student → open profile → add/link guardian → enroll in a section → upload photos/documents.
2. **Class:** Create class and academic session → create section with capacity → assign teacher and subjects → enroll students. Transferring an existing student closes their prior active enrollment.
3. **Fees:** Configure class charges → add individual concessions/scholarships → Generate invoices for a billing period → record full/partial payments → print/download receipt. Corrections use Adjustments & refunds.
4. **Attendance:** Select section and date → mark Present/Absent/Late/Leave → Save attendance. History and changes remain audited.
5. **Academics:** Create examination → define every assessed subject and maximum/passing marks → enter marks (or mark absent) → Publish report card → view/download/print.
6. **Parent access:** Create user with their sign-in email → assign Parent/Guardian role → select that user on the guardian record → link guardian to student. Teachers similarly require class-teacher/section assignments.
7. **Communications:** Preview recipients and text, record a test or live queued message, then process the due queue. See [provider setup](docs/INTEGRATIONS.md).

## Authentication and permissions

Hosted sign-in, session management, credential security and account recovery are handled by Sites/its identity provider. School Buddy does not store passwords or hardcode production credentials. Every API checks authentication and active membership. School IDs supplied in request data cannot change the caller's tenant. Teacher/student and parent/child scopes are enforced on the server, including exports, PDFs and document downloads.

New Sites are owner-private. Additional school users need both hosting access and application membership. Widen hosting access deliberately using Sites controls; do not publish school records publicly.

## Configuration

`.env.example` documents messaging secret names without secrets. Keep actual local settings in ignored files or runtime environment variables. Use native Sites runtime secret configuration for production. Never enter API tokens into the school settings UI.

## Testing

```sh
pnpm test
python tests/database_test.py
pnpm exec tsc --noEmit
pnpm build
```

The Node tests cover financial calculations, grades, attendance-related result rules, role permissions, template rendering and timetable conflicts. Python tests apply the real migrations to an isolated in-memory SQLite database and test overpayment, append-only history, unique admission numbers, foreign keys, invoice immutability and duplicate attendance.

`node scripts/http-smoke.mjs` is an optional **mutating local-demo-only** workflow test; it creates and archives a QA student, records a payment and a traceable reversal, publishes a report and saves a test message. It refuses production URLs or non-demo schools. This test was not run during delivery because permission was not granted.

## Deployment

`.openai/hosting.json` preserves the Site ID and declares `DB` and `BUCKET`. The Sites build plugin emits a deployable Worker. Follow the Sites source workflow to push the exact checked source, package the matching build, and deploy it privately. Sites provisions D1/R2 and applies the migrations. Reuse the existing Site ID; do not register a duplicate Site.

For standalone Cloudflare hosting, provision D1/R2, adapt trusted authentication, configure bindings, apply migrations, then deploy the generated Worker. Do not forward browser-supplied identity headers as trusted identity on a standalone public server.

## Known limits

This is a working initial application, **not a claim that every requested enterprise feature is production-complete**. Autonomous scheduled reminders and signed delivery webhooks, fully implemented outbound WhatsApp templates, server-side pagination/aggregate reporting for large schools, native Excel exports and multilingual embedded PDF fonts need further work. Late fees can be assessed from configured charges and recorded as auditable adjustments. Class-wide promotion/transfer is available. Direct PDF downloads use Latin standard fonts; use browser print for other scripts. The optional Supabase port/RLS migrations require PostgreSQL verification before use. Real provider delivery and multiple real user accounts cannot be verified without credentials and hosting access.
