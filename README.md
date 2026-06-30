# Aqoon360--case-study
Aqoon360 — University & College Management and Attendance Platform
A production-grade, multi-tenant platform that replaces manual attendance sheets, scattered spreadsheets, and disconnected admin workflows with a structured digital university system — delivered as a React Native mobile app for lecturers and students and a Next.js web dashboard for institution administrators.

⚠️ Note on source code
This is a public case-study repository. The production source code for Aqoon360 is private because it is an active commercial product. This repo intentionally does not contain application source, Firebase configuration, API keys, security rules, customer data, or internal business logic. Its purpose is to document the product, architecture, and engineering decisions behind the platform, and to clarify my role in building it.

Table of Contents
Overview
Problem
Solution
Tech Stack
Core Features
Architecture
Data & Access Model
Attendance Workflow
Web Admin Dashboard
Security & Permissions
Offline & Reliability
Testing
Engineering Challenges Solved
My Role
Status
Screenshots
Overview
Aqoon360 is a university and college management platform that digitizes the academic operations of an institution — attendance, student records, timetables, staff management, exams and grades, and reporting — across two coordinated surfaces:

Mobile (React Native + Expo) — the operational layer where lecturers take attendance and students access their timetable, attendance, and grades.
Web (Next.js) — the administrative layer where institution admins manage departments, courses, classes, staff, students, and reports.
The platform is multi-tenant (each institution's data is isolated) and role-aware (admin, lecturer, student, and staff each see a tailored experience backed by enforced access rules).

Problem
Many universities and colleges still run core academic operations on paper and disconnected tools:

Paper attendance sheets are slow to collect, easy to lose, and impossible to analyze in real time.
Spreadsheets for student records, timetables, and grades drift out of sync and have no access control — anyone with the file sees everything.
Disconnected admin workflows mean staff re-enter the same data across systems, and leadership has no consolidated, trustworthy view of attendance or academic activity.
No mobile-first capture at the point where data is actually created — the classroom — so digitization happens late, manually, and with errors.
The result: lost time, unreliable records, weak reporting, and no single source of truth per institution.

Solution
Aqoon360 moves data capture to where it happens and centralizes it under a structured, permissioned model:

Lecturers capture attendance on mobile, in the classroom, in seconds — even on unreliable networks.
Students get a self-service portal for their own timetable, attendance, grades, and profile.
Admins manage their institution from the web dashboard — departments, faculties, courses, classes, staff, students, and reports.
Everything is real-time and permissioned: data updates propagate live, and each role can only see and act on what it is allowed to.
Reporting and analytics turn raw attendance and academic activity into views institutions can act on.
The design principle throughout: separate mobile operational workflows from web administrative workflows, while both read and write to a single, consistent, multi-tenant data model.

Tech Stack
Layer	Technology
Mobile app	React Native, Expo, TypeScript
Web admin	Next.js, TypeScript
Authentication	Firebase Authentication
Database	Cloud Firestore (real-time listeners)
Authorization	Firestore Security Rules + role-based access control
Backend logic	Firebase Cloud Functions (where server-side trust is required)
Offline support	Firestore offline persistence + offline-tolerant capture flows
State / data	Real-time subscriptions, optimistic UI on the mobile capture path
Testing	Jest, React Native Testing Library, Firebase Emulator Suite
Specific library choices, configuration, and security-rule implementations are part of the private production codebase and are intentionally omitted here.

Core Features
Institution administration (web)

Multi-tenant institution structure (each institution isolated)
Department / faculty / course / class management
Staff management and role assignment
Student records management
Timetable management
Reports and attendance/academic analytics
Lecturer (mobile)

Mobile-first attendance capture for assigned classes
Access scoped strictly to assigned courses and classes
Real-time attendance session updates
Offline-tolerant capture for unreliable networks
Student (mobile)

Personal timetable
Personal attendance history
Grades / exam marks
Profile
Cross-cutting

Role-based dashboards for admin, lecturer, student, and staff
Real-time data across web and mobile
Department → faculty → course → class → student/lecturer relationships
Exams, grades, and marks
Reporting layer over attendance and academic activity
Architecture
Aqoon360 uses a two-frontend, shared-backend architecture. The mobile app and the web dashboard are independent clients that share one authenticated, permissioned Firebase backend. There is no separate hand-rolled API server for core data flows — clients talk to Firestore directly under the protection of Security Rules, with Cloud Functions used where server-side trust is required.


flowchart TB
    subgraph Clients
        Mobile["📱 React Native + Expo<br/>Lecturer & Student app"]
        Web["💻 Next.js<br/>Institution Admin dashboard"]
    end

    subgraph Firebase["🔥 Firebase Backend (per-tenant data)"]
        Auth["Firebase Authentication<br/>identity + custom claims (role, tenant)"]
        Rules["Firestore Security Rules<br/>role + tenant enforcement"]
        DB[("Cloud Firestore<br/>institutions, departments, courses,<br/>classes, students, lecturers,<br/>attendance sessions, grades, reports")]
        Fns["Cloud Functions<br/>privileged / server-trusted logic"]
    end

    Mobile -->|sign in| Auth
    Web -->|sign in| Auth
    Auth -->|token w/ role + tenant| Mobile
    Auth -->|token w/ role + tenant| Web

    Mobile -->|read/write<br/>real-time listeners| Rules
    Web -->|read/write<br/>real-time listeners| Rules
    Rules --> DB
    Fns --> DB
    Mobile -.->|callable| Fns
    Web -.->|callable| Fns
Key architectural decisions

Separation of concerns by surface, not duplication of logic. Mobile owns operational capture (attendance, personal views); web owns administration (structure, records, reporting). Both share the same data model so there is one source of truth.
Authorization lives at the data layer. Access is enforced by Firestore Security Rules keyed on the authenticated user's role and tenant, not just hidden in the UI — so a client cannot read or write data it isn't entitled to, regardless of how it is built.
Real-time by default. Both clients subscribe to live listeners, so an attendance mark on mobile is reflected on the admin dashboard without manual refresh.
Cloud Functions only where trust must be server-side, keeping the common path simple and fast while protecting sensitive operations.
Data & Access Model
The data model mirrors how an institution is actually structured, and access is scoped along the same hierarchy.


flowchart TD
    Inst["🏛️ Institution (tenant)"]
    Inst --> Fac["Faculty"]
    Fac --> Dept["Department"]
    Dept --> Course["Course"]
    Course --> Class["Class / Section"]
    Class --> Session["Attendance Session"]
    Class --> Enroll["Enrolled Students"]
    Class --> Assign["Assigned Lecturer"]
    Session --> Record["Attendance Records"]
    Enroll --> Grade["Grades / Marks"]
Multi-tenancy. Every domain document is owned by exactly one institution (tenant). All reads and writes are scoped to the caller's tenant so no institution can ever see another's data.

Role-based scoping.

Role	Can access
Platform admin	Platform-level operations (kept fully separate from tenant experiences)
Institution admin	Only their own institution: departments, faculties, courses, classes, staff, students, reports
Lecturer	Only assigned courses/classes and the attendance sessions they own
Student	Only their own attendance, timetable, grades, and profile
Staff	Scoped to their assigned administrative responsibilities
This model is enforced both in the UI (what each role is shown) and at the data layer (what each role is permitted to read/write).

Attendance Workflow
Attendance is the highest-frequency, most network-sensitive flow in the product, so it gets a dedicated, mobile-first design.


sequenceDiagram
    participant L as Lecturer (mobile)
    participant App as RN App
    participant Cache as Local cache (offline persistence)
    participant DB as Firestore
    participant Admin as Admin dashboard (web)

    L->>App: Open assigned class
    App->>DB: Subscribe to class + roster
    DB-->>App: Roster (live)
    L->>App: Mark present / absent / late
    App->>Cache: Write immediately (optimistic)
    App-->>L: Instant UI confirmation
    Cache->>DB: Sync when network available
    DB-->>Admin: Live attendance update
    DB-->>App: Confirmed write
What makes it reliable

Optimistic, local-first writes — the lecturer's action is confirmed instantly; the network sync happens behind the scenes.
Offline-tolerant — attendance can be captured with no connectivity and syncs automatically when the device reconnects.
Real-time propagation — once synced, the admin dashboard reflects attendance live.
Scoped access — a lecturer can only open and record sessions for classes assigned to them.
Web Admin Dashboard
Built with Next.js + TypeScript, the admin dashboard is the control center for an institution:

Institutional structure — manage faculties, departments, courses, and classes and the relationships between them.
People — manage staff and student records, assign lecturers to classes, and control role assignment.
Timetables — define and maintain class schedules that flow through to student and lecturer views.
Reports & analytics — consolidated views of attendance and academic activity for decision-making.
Tenant isolation — an admin operates strictly within their own institution; platform-level concerns are kept entirely separate and never surfaced to tenant admins.
The dashboard consumes the same real-time Firestore data as the mobile clients, so administrative views stay in sync with on-the-ground activity without batch imports.

Security & Permissions
Security is treated as a data-layer responsibility, not a UI convenience.

Authentication via Firebase Authentication establishes identity for every client.
Role and tenant context travel with the authenticated session and drive both UI and data access.
Firestore Security Rules enforce, at the database boundary:
Tenant isolation — a user can only touch documents belonging to their own institution.
Lecturer scoping — read/write limited to assigned courses, classes, and owned attendance sessions.
Student scoping — read limited to their own attendance, timetable, grades, and profile; no access to other students' data.
Admin scoping — management limited to their own institution.
Cloud Functions handle operations that must be trusted server-side, so privileged logic never relies on the client.
Layer separation — platform-admin concepts and data are never exposed to institution tenants.
The actual rule definitions, claim structure, and Firebase configuration are part of the private codebase and are deliberately not published here.

Offline & Reliability
Aqoon360 is built for environments where connectivity is intermittent and unreliable, which is common in the target market.

Firestore offline persistence keeps recently accessed data (rosters, schedules) available without a connection.
Offline-tolerant attendance capture lets lecturers record a full session offline; writes queue locally and sync automatically on reconnect.
Optimistic UI on the capture path means the app never blocks the user waiting on the network.
Eventual consistency with real-time convergence — once a device reconnects, queued writes sync and all subscribed clients (including the admin dashboard) converge on the latest state.
This makes the classroom capture flow feel instant and dependable regardless of network quality.

Testing
The platform is validated with a layered testing approach:

Unit & component tests — Jest and React Native Testing Library for mobile UI and logic.
Rules & integration tests — Firebase Emulator Suite to exercise Firestore Security Rules and data-access behavior locally, verifying that role/tenant scoping holds (e.g., a lecturer cannot read another lecturer's class, a student cannot read another student's grades).
Workflow validation — emulated end-to-end checks of the critical attendance capture-and-sync path, including offline-then-reconnect scenarios.
Testing focuses on the highest-risk areas: authorization correctness and the offline attendance sync path.

Engineering Challenges Solved
Enforcing multi-tenant + multi-role access at the data layer — designing Security Rules and a data model so that tenant isolation and per-role scoping are guaranteed by Firestore itself, not merely hidden in the client.
Offline-tolerant attendance capture — building a capture flow that is instant and reliable on unreliable networks, with local-first writes and automatic background sync.
Coordinating two independent frontends over one shared backend — keeping a React Native app and a Next.js dashboard consistent through a single real-time data model without duplicating business logic.
Modeling a real institutional hierarchy — faculty → department → course → class → session/enrollment relationships that are flexible enough for real institutions and efficient to query in Firestore.
Real-time without overload — using scoped, targeted real-time listeners so live updates feel immediate without over-fetching or excessive reads.
Clean separation of platform vs. tenant concerns — ensuring platform-level administration never leaks into the experience or data of individual institutions.
My Role
I am the Senior Frontend / React Native developer behind Aqoon360. My responsibilities span:

Mobile app (React Native + Expo + TypeScript) — architecture, lecturer attendance flow, student portal, offline-tolerant capture, and real-time data integration.
Web admin dashboard (Next.js + TypeScript) — institution administration, structural management, and reporting views.
Firebase backend integration — Authentication, Firestore data modeling, real-time listeners, and Security Rules for role- and tenant-based access control.
Engineering decisions — the two-frontend/shared-backend architecture, the multi-tenant data model, the offline attendance strategy, and the separation between operational (mobile) and administrative (web) workflows.
I own the product's frontend architecture end-to-end across both surfaces and the Firebase data/access layer that backs them.

Status
Aqoon360 is an active, in-production commercial platform. This case-study repository is maintained as a public, source-free overview of the product and its engineering. Feature set and architecture described here reflect the current production system.

Screenshots
Visuals from the production app. Replace the placeholders below with real screenshots.

Lecturer mobile — attendance capture
[Add screenshot here]

Student portal — timetable & attendance
[Add screenshot here]

Admin dashboard — institution overview
[Add screenshot here]

Admin dashboard — reports & analytics
[Add screenshot here]

Data / access model diagram
[Add screenshot here]
