# AI Development Pattern Library

> **Demo patterns and examples for AI-powered software development**

⚠️ **DEMO REPOSITORY** - This is a demonstration and learning resource showcasing architectural patterns. These are examples and starting points, **not production-ready code**. Use these patterns as inspiration and adapt them for your specific production requirements.

A curated collection of architectural patterns, best practices, and implementation guides designed to work seamlessly with AI coding assistants like Claude, ChatGPT, and GitHub Copilot.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## 📚 What's Inside

This pattern library contains **11 comprehensive patterns** organized into three categories:

### 🚀 Getting Started
- **[Pattern Index](getting-started/PATTERNS_INDEX.md)** - AI navigation hub and quick start guide
- **[Overview](getting-started/README.md)** - What are these patterns and how to use them

### ⚙️ Backend Patterns (6 patterns)
- **[Authentication Strategy](backend/AUTHENTICATION_STRATEGY.md)** - JWT with RS256, token management, auth guards
- **[Messaging Strategy](backend/MESSAGING_STRATEGY.md)** - RabbitMQ event-driven architecture, outbox/inbox patterns
- **[Universal Patterns](backend/UNIVERSAL_PATTERNS.md)** - Standard service structure, configuration, health checks
- **[Processing Patterns](backend/PROCESSING_PATTERNS.md)** - FFmpeg integration, event-driven processing, retry logic
- **[Storage Strategy](backend/STORAGE_STRATEGY.md)** - S3/MinIO object storage, pre-signed URLs
- **[Observability](backend/OBSERVABILITY.md)** - Structured logging, metrics, correlation IDs

### 🎨 Frontend Patterns (3 patterns)
- **[Frontend Patterns](frontend/FRONTEND_PATTERNS.md)** - Vue 3 + TypeScript architecture, component organization
- **[API Integration](frontend/API_INTEGRATION.md)** - Axios setup, auth interceptors, error handling
- **[Component Patterns](frontend/COMPONENT_PATTERNS.md)** - TypeScript components, accessibility, best practices

---

## 🎯 Why These Patterns?

### For Developers
- **Accelerate learning** on common architectural decisions
- **Practical examples** - demonstrating real-world patterns (not production-ready)
- **AI-optimized** - designed to work with AI coding assistants
- **Real-world inspired** - derived from production services as educational examples

### For AI Assistants
- Clear navigation structure (start with `getting-started/PATTERNS_INDEX.md`)
- Consistent formatting for reliable code generation
- Cross-referenced patterns for context awareness
- Implementation checklists to ensure completeness

### For Teams
- Consistent architecture across all services
- Faster onboarding for new developers
- Reduced code review time
- Shared vocabulary for technical discussions

---

## 🚀 Quick Start

### 1. For Manual Use
Browse the patterns by category:
- Start with [Pattern Index](getting-started/PATTERNS_INDEX.md) for an overview
- Pick a pattern based on your needs
- Follow the implementation guide
- Copy and adapt code examples

### 2. For AI Assistants
Include this instruction in your prompts:

```
Please use the patterns from this repository as guidelines.
Start by reading getting-started/PATTERNS_INDEX.md to understand
available patterns, then reference specific patterns as needed.
```

### 3. Three Common Scenarios

#### Scenario 1: Build a Registration Service
**AI reads:**
1. `PATTERNS_INDEX.md` → Discovers `AUTHENTICATION_STRATEGY.md`
2. `AUTHENTICATION_STRATEGY.md` → JWT patterns, RS256, token storage
3. `MESSAGING_STRATEGY.md` → Event publishing for "user.registered"
4. `UNIVERSAL_PATTERNS.md` → Standard service structure

**Result:** Demo NestJS service structure with JWT auth examples, RabbitMQ event patterns, health checks

#### Scenario 2: Create a Video Processing Service
**AI reads:**
1. `PATTERNS_INDEX.md` → Discovers `PROCESSING_PATTERNS.md`
2. `PROCESSING_PATTERNS.md` → FFmpeg integration, queue-based processing
3. `STORAGE_STRATEGY.md` → S3 storage, signed URLs
4. `OBSERVABILITY.md` → Metrics, logging, progress tracking

**Result:** Example event-driven video processor pattern with FFmpeg, S3, observability concepts

#### Scenario 3: Add User Dashboard with Analytics
**AI reads:**
1. `PATTERNS_INDEX.md` → Discovers `FRONTEND_PATTERNS.md`
2. `FRONTEND_PATTERNS.md` → Vue 3 + TypeScript architecture
3. `API_INTEGRATION.md` → REST client, auth token handling
4. `COMPONENT_PATTERNS.md` → Chart components, responsive layouts

**Result:** Example TypeScript dashboard pattern with API integration, charts, responsive layout concepts

---

## 📖 Pattern Structure

Each pattern includes:
- ✅ **Overview** - What problem it solves
- ✅ **When to Use** - Applicable scenarios
- ✅ **Core Concepts** - Key architectural decisions
- ✅ **Implementation Guide** - Step-by-step instructions
- ✅ **Code Examples** - Copy-paste ready code
- ✅ **Best Practices** - Security, performance, maintainability
- ✅ **Integration Checklist** - Ensure nothing is missed
- ✅ **Related Patterns** - Cross-references

---

## 🛠️ Technology Stack

### Backend
- **Framework:** NestJS (TypeScript)
- **Message Queue:** RabbitMQ
- **Database:** PostgreSQL with TypeORM
- **Object Storage:** S3 / MinIO
- **Authentication:** JWT with RS256
- **Observability:** Winston, Prometheus

### Frontend
- **Framework:** Vue 3 with Composition API
- **Language:** TypeScript (strict mode)
- **State Management:** Pinia
- **HTTP Client:** Axios
- **Build Tool:** Vite
- **Styling:** CSS variables + PrimeVue

---

## 🎓 Who Is This For?

### ✅ Perfect For
- Developers learning microservices architectures
- Teams exploring AI coding assistant workflows
- Engineers studying event-driven architecture patterns
- Anyone learning full-stack TypeScript development
- Developers seeking architectural inspiration and examples

### ⚠️ Not Ideal For
- Direct production deployment (these are demos/examples only)
- Simple CRUD applications (these patterns may be overkill)
- Non-TypeScript projects (patterns are TS-first)
- Monolithic architectures (designed for microservices)

---

## 📦 What You Get

| Category | Files | Lines of Code | Value |
|----------|-------|---------------|-------|
| Getting Started | 2 | ~950 | Navigation & overview |
| Backend Patterns | 6 | ~2,400 | Demo service examples |
| Frontend Patterns | 3 | ~1,350 | UI pattern examples |
| **Total** | **11** | **~4,700** | **Demo architecture patterns** |

---

## 🤝 Contributing

While this is a curated pattern library, we welcome:
- **Bug reports** - Found an issue? Let us know
- **Real-world feedback** - How did these patterns work for you?
- **Suggestions** - Missing a critical pattern?

Please open an issue to discuss before submitting PRs.

---

## 📄 License

MIT License - See [LICENSE](LICENSE) for details.

**TL;DR:** Use these patterns freely in your projects, commercial or otherwise. Attribution appreciated but not required.

---

## 🌟 Support This Project

If you find these demo patterns helpful for learning:
- ⭐ **Star this repository** on GitHub
- 🔗 **Share** with your team and network
- 📝 **Write about** your experience using these patterns
- 💬 **Provide feedback** to help improve the examples

---

## 🔗 Links

- **Live Pattern Viewer:** [Patterns](https://learn.gilltrick.com/patterns)
- **Website:** [AI Aligned Development ](https://learn.gilltrick.com)
- **Blog:** [Blog](https://learn.gilltrick.com/blog)
- **Journey:** [Journey-Blog](https://blog.gilltrick.de/blog/awakaning)
- **Workshops:** [Contact for details](info@gilltrick.de)

---

## 📊 Stats

- **Patterns:** 11
- **Code Examples:** 50+
- **Implementation Checklists:** 10
- **Lines of Demo Code:** ~4,700
- **Learning Resource:** Educational patterns and examples

---

## 🙏 Acknowledgments

These demo patterns are inspired by real production systems:
- Registration service (authentication, messaging)
- Mail service (event processing, queueing)
- Upload service (storage, pre-signed URLs)
- Video processing service (FFmpeg, S3, events)
- Video streaming service (CDN, optimization)
- Frontend applications (Vue 3, TypeScript, API integration)

Distilled from **16,500+ lines** of production code into **4,700 lines** of focused demo examples and learning materials.

---

**Built with ❤️ for developers learning AI-powered development patterns**

*Last updated: 2025-11-11*
