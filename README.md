# Resiliency Skills

Production-readiness skills for AI agents. Build secure, observable, and performant applications.

## Installation

Install via [skills.sh](https://skills.sh):

```bash
npx skills add emilio/resiliency_skills
```

Or install individual skills:

```bash
npx skills add emilio/resiliency_skills/skills/auth
npx skills add emilio/resiliency_skills/skills/observability
npx skills add emilio/resiliency_skills/skills/security
npx skills add emilio/resiliency_skills/skills/logging
npx skills add emilio/resiliency_skills/skills/performance
```

## Skills

| Skill | Description |
|-------|-------------|
| **auth** | Authentication with NextAuth, Passport.js, golang-jwt, Supabase |
| **observability** | Tracing & metrics with OTel, Datadog, New Relic, Dynatrace |
| **security** | Dependency audits, secure coding, API protection |
| **logging** | Structured JSON logging with Pino, Winston, slog |
| **performance** | Load testing, benchmarking, profiling with k6, pprof |
| **production-ready** | Orchestrator that coordinates all skills (optional) |

## Framework Support

| Framework | Auth | Observability | Security | Logging | Performance |
|-----------|------|---------------|----------|---------|-------------|
| Next.js | NextAuth | ✓ | ✓ | Pino | k6 |
| Express | Passport | ✓ | Helmet | Pino/Winston | k6 |
| Go | golang-jwt | ✓ | ✓ | slog | pprof |
| React SPA | Supabase | ✓ | ✓ | - | k6 |

## Usage

Each skill activates automatically when relevant. For example:

```
User: Build me a todo API with authentication

Agent: [auth skill activates]
       Setting up NextAuth with protected routes...
```

Or explicitly request production-readiness:

```
User: Build me a production-ready e-commerce API

Agent: [production-ready skill activates]
       I'll set up authentication, security, observability, logging, and performance testing.
       Which observability provider would you prefer?
       - OpenTelemetry (recommended)
       - Datadog
       - New Relic
       - Dynatrace
```

## Structure

```
resiliency_skills/
├── skills/
│   ├── auth/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── nextjs.md
│   │       ├── express.md
│   │       ├── go.md
│   │       └── supabase.md
│   ├── observability/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── providers/
│   │       └── frameworks/
│   ├── security/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── logging/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── performance/
│   │   ├── SKILL.md
│   │   └── references/
│   └── production-ready/
│       └── SKILL.md
└── README.md
```

## Specification

These skills follow the [Agent Skills format](https://agentskills.io/specification).

## License

Apache 2.0 License
