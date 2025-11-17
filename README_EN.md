<div align="center">

# 🚀 Node.js Backend Development Notes

[![GitHub stars](https://img.shields.io/github/stars/cyrus0315/nodejs-backend-notes?style=social)](https://github.com/cyrus0315/nodejs-backend-notes/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/cyrus0315/nodejs-backend-notes?style=social)](https://github.com/cyrus0315/nodejs-backend-notes/network/members)
[![GitHub issues](https://img.shields.io/github/issues/cyrus0315/nodejs-backend-notes)](https://github.com/cyrus0315/nodejs-backend-notes/issues)
[![GitHub license](https://img.shields.io/github/license/cyrus0315/nodejs-backend-notes)](https://github.com/cyrus0315/nodejs-backend-notes/blob/main/LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/cyrus0315/nodejs-backend-notes/pulls)

**A comprehensive and practical guide to Node.js backend development** 📚

Covering Node.js Core, TypeScript, Databases, Message Queues, Frameworks, ORMs, API Design, Security, Testing, AWS, DevOps, System Design, and more

English | [简体中文](./README.md)

</div>

---

## ✨ Introduction

After working in backend development for several years, I realized that my knowledge was fragmented and never systematically organized. This repository is my journey to reorganize the entire Node.js backend tech stack, documenting what I've used, learned, and struggled with in real projects.

### 🎯 Who Is This For

- 🌱 Beginners who want to learn Node.js backend development systematically
- 💼 Working developers who need quick reference to technical concepts
- 📖 Job seekers preparing for interviews and reviewing knowledge
- 🚀 Full-stack engineers looking to improve their backend skills

### 🌟 Features

- ✅ **Comprehensive**: Covers all aspects of Node.js backend development
- ✅ **Practice-Oriented**: Based on real project experience with plenty of examples
- ✅ **Continuously Updated**: New content added based on work practice
- ✅ **Chinese-Friendly**: Original content in Chinese with English version
- ✅ **Ready-to-Use**: Code examples can be run directly

## 📑 Table of Contents

- [✨ Introduction](#-introduction)
- [📚 Knowledge Base](#-knowledge-base)
  - [Language and Core](#language-and-core)
  - [Data Storage](#data-storage)
  - [Frameworks and Tools](#frameworks-and-tools)
  - [API Development](#api-development)
  - [Cloud Services](#cloud-services)
  - [Fundamentals](#fundamentals)
  - [Architecture](#architecture)
- [📝 How to Use](#-how-to-use)
- [💭 Learning Tips](#-learning-tips)
- [🤝 Contributing](#-contributing)
- [📄 License](#-license)
- [⭐ Star History](#-star-history)

## 📚 Knowledge Base

### Language and Core

- **[01-nodejs-core](./01-nodejs-core)** - Node.js Core Mechanisms
  - Event Loop, Streams, Buffers, Process & Threads, Performance Optimization
  - Fundamental concepts frequently asked in interviews
  
- **[02-typescript](./02-typescript)** - TypeScript Deep Dive
  - Type System, Generics, Decorators, Utility Types
  - Essential for modern Node.js development

### Data Storage

- **[03-database](./03-database)** - Databases
  - PostgreSQL, MySQL, MongoDB, Redis, DynamoDB
  - Real-world database usage and common pitfalls

- **[04-message-queue](./04-message-queue)** - Message Queues
  - Kafka, RabbitMQ, AWS SQS/SNS
  - Essential for async processing and decoupling

### Frameworks and Tools

- **[05-frameworks](./05-frameworks)** - Framework Comparison
  - Express, Koa, NestJS, Fastify, Monorepo
  - Pros and cons of different frameworks and when to use them

- **[06-orm-odm](./06-orm-odm)** - ORM/ODM
  - Prisma, TypeORM, Sequelize, Mongoose
  - Database operation tools and efficiency comparison

### API Development

- **[07-api-design](./07-api-design)** - API Design
  - RESTful, GraphQL, API Standards
  - Good API design saves a lot of trouble

- **[08-authentication-security](./08-authentication-security)** - Security
  - JWT, OAuth, Common Security Issues
  - Security cannot be ignored

- **[09-testing](./09-testing)** - Testing
  - Jest, Unit Tests, Integration Tests
  - Writing tests really reduces bugs

### Cloud Services

- **[10-aws](./10-aws)** - AWS Services
  - Lambda, API Gateway, SQS, S3, DynamoDB
  - Common AWS services for backend development

- **[11-devops](./11-devops)** - DevOps
  - Docker, CI/CD, GitHub Actions
  - Deployment and operations best practices

- **[12-monitoring-logging](./12-monitoring-logging)** - Monitoring & Logging
  - Log Management, Performance Monitoring, Error Tracking
  - Essential for production troubleshooting

### Fundamentals

- **[13-network-protocols](./13-network-protocols)** - Network Protocols
  - HTTP/HTTPS, WebSocket, TCP/UDP
  - Fundamental but crucial

- **[14-algorithms-data-structures](./14-algorithms-data-structures)** - Algorithms
  - Common Algorithms and Data Structures
  - May not write often but need to understand

### Architecture

- **[15-system-design](./15-system-design)** - System Design
  - High Availability, High Concurrency, Distributed Systems
  - Increasingly important as projects scale

- **[16-architecture](./16-architecture)** - Architecture Patterns
  - Microservices, DDD, Event-Driven Architecture
  - Trade-offs of different architectures

### Other

- **[17-tools-skills](./17-tools-skills)** - Tools & Skills
  - Git, Linux, Code Standards
  - Essential daily work tools

## 📝 How to Use

### Using This Repository

Each directory contains detailed notes and code examples. I usually:

1. Check README for overall structure
2. Focus on unfamiliar topics
3. Run the code examples
4. Research deeper when confused
5. Add new insights as I learn

### Checkbox Purpose

Each module has `[ ]` checkboxes to mark your progress:
- `[ ]` Haven't studied or not familiar yet
- `[x]` Already understood and mastered

No need to check everything—mark according to your needs.

## 💭 Learning Tips

### About Learning Methods

1. **Don't be greedy**: Focus on one topic at a time, master it before moving on
2. **Practice**: Reading without coding is useless—write the code yourself
3. **Take notes**: Write down what you understand for faster future reference
4. **Build things**: Implementing things yourself leads to deeper understanding
5. **Read source code**: When in doubt, read the source code, don't guess

### Recommended Resources

**Books** (that I found helpful):
- "Understanding Node.js: Core Concepts" by Pu Ling (朴灵) - Classic
- "Node.js Design Patterns" - Many practical patterns
- "Designing Data-Intensive Applications" - Theoretical but very useful

**Websites**:
- Node.js Official Documentation - Most authoritative resource
- MDN - Everything about web development
- GitHub - Read code from excellent projects

**Practice**:
- Build your own small projects
- Contribute to open source
- Refactor old code

## 🔄 Continuous Updates

This repository is continuously updated with:
- New technologies learned
- Problems encountered and solutions
- Performance optimization practices
- Architecture design insights

If you're also learning Node.js backend development, feel free to exchange ideas.

## 📂 Directory Structure

```
.
├── 01-nodejs-core/          # Node.js Core
├── 02-typescript/           # TypeScript
├── 03-database/             # Databases
├── 04-message-queue/        # Message Queues
├── 05-frameworks/           # Frameworks
├── 06-orm-odm/              # ORM/ODM
├── 07-api-design/           # API Design
├── 08-authentication-security/  # Security
├── 09-testing/              # Testing
├── 10-aws/                  # AWS
├── 11-devops/               # DevOps
├── 12-monitoring-logging/   # Monitoring & Logging
├── 13-network-protocols/    # Network Protocols
├── 14-algorithms-data-structures/  # Algorithms
├── 15-system-design/        # System Design
├── 16-architecture/         # Architecture
└── 17-tools-skills/         # Tools & Skills
```

---

## 🤝 Contributing

Contributions are welcome! If you have suggestions or find issues:

1. 🐛 [Submit an Issue](https://github.com/cyrus0315/nodejs-backend-notes/issues/new) to report problems or suggestions
2. 🔧 [Submit a Pull Request](https://github.com/cyrus0315/nodejs-backend-notes/pulls) to contribute code or content
3. ⭐ Give the project a Star to support
4. 🔀 Fork the project and add your own notes

See [CONTRIBUTING.md](./CONTRIBUTING.md) for detailed contribution guidelines.

## 💬 Discussions

If you have questions, suggestions, or ideas:
- 💡 [GitHub Discussions](https://github.com/cyrus0315/nodejs-backend-notes/discussions) - Discussion area
- 🐛 [GitHub Issues](https://github.com/cyrus0315/nodejs-backend-notes/issues) - Bug reports
- ⭐ Star the project to share with others

## 📄 License

This project is licensed under the [MIT License](./LICENSE).

## ⭐ Star History

If this project helps you, please give it a Star ⭐️!

[![Star History Chart](https://api.star-history.com/svg?repos=cyrus0315/nodejs-backend-notes&type=Date)](https://star-history.com/#cyrus0315/nodejs-backend-notes&Date)

---

<div align="center">

**Keep learning, keep growing.** 💪

Made with ❤️ by [cyrus0315](https://github.com/cyrus0315)

If this project helps you, please ⭐️ Star it!

</div>

