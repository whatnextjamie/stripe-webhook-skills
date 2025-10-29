# Stripe Webhook Skills for Claude

Two complementary skills that give Claude expert knowledge for working with Stripe webhooks - one for debugging and understanding, one for code generation.

---

## 🚀 Getting Started

### For Humans:

1. **Fork this repository** to your GitHub account
2. **Clone it locally** or open in your development environment
3. **Ensure skill-creator is enabled** in Claude Code Settings → Capabilities → Skills
4. **Ask Claude:** "Create a new skill" or "Install the Stripe webhook skills"

That's it! Claude will handle the rest.

### For AI Assistants:

When a user wants to work with Stripe webhooks, you now have access to two specialized skills in this repository:

1. **stripe-webhook-debugger** - For understanding, debugging, and learning about webhooks
2. **stripe-webhook-code-generator** - For generating production-ready webhook handler code

See the **Instructions for AI** section below for detailed guidance.

---

## 📦 What's Included

### 1. Stripe Webhook Debugger
**Purpose:** Education, debugging, and understanding webhooks

**Capabilities:**
- Debug webhook failures and signature verification errors
- Analyze webhook payload data
- Explain Stripe event types and their meanings
- Provide error code interpretations
- Guide on best practices and security

**Reference Files:**
- `stripe-webhook-debugger/SKILL.md` - Main skill definition
- `stripe-webhook-debugger/references/signature-verification.md` - Signature debugging guide
- `stripe-webhook-debugger/references/event-types.md` - Event catalog
- `stripe-webhook-debugger/references/error-codes.md` - Error reference
- `stripe-webhook-debugger/references/testing.md` - Testing with Stripe CLI
- `stripe-webhook-debugger/references/configuration.md` - Dashboard setup
- `stripe-webhook-debugger/references/event-processing.md` - Processing patterns
- `stripe-webhook-debugger/references/best-practices.md` - Architecture & security

### 2. Stripe Webhook Code Generator
**Purpose:** Production-ready code generation with codebase integration

**Capabilities:**
- Generate webhook handler code for multiple frameworks
- Integrate with existing codebases and conventions
- Create tests and documentation
- Add database, email, and logging integrations
- Update and refactor existing handlers

**Reference Files:**
- `stripe-webhook-code-generator/SKILL.md` - Main skill definition
- `stripe-webhook-code-generator/references/code-templates.md` - Framework templates
- `stripe-webhook-code-generator/references/testing-patterns.md` - Test generation patterns

---

## 📋 How It Works

### Just Describe Your Problem - Claude Handles the Rest

**You don't need to remember which skill to use.** Claude automatically determines whether you need debugging help, code generation, or both based on what you're trying to accomplish.

**Simply describe what you're working on:**

- "My webhook keeps failing with signature errors"
- "I need to handle Stripe subscriptions in my Next.js app"
- "What does this event payload mean?" (paste JSON)
- "Generate a webhook handler that integrates with my database"
- "Why isn't my webhook receiving events?"

Claude will automatically:
1. **Use the Debugger** when you're troubleshooting, analyzing payloads, or learning about events
2. **Use the Code Generator** when you're building or updating webhook handlers
3. **Use both in sequence** when you need to understand first, then implement

### What You Can Expect

**When you're troubleshooting:**
- Systematic diagnosis of signature verification, configuration, or processing issues
- Plain-English explanations of event payloads and what they mean
- Step-by-step debugging guidance with Stripe CLI commands
- Best practices for security and architecture

**When you're building:**
- Analysis of your existing codebase to understand your patterns
- Production-ready code that matches your project's style
- Integration with your database, email service, and logging
- Comprehensive tests and setup documentation

---

## 🤖 Instructions for AI

When a user is working with Stripe webhooks, these skills provide you with comprehensive reference material and templates to assist them effectively.

### How to Use These Skills

**1. Determine the User's Need**

Ask yourself:
- Is the user trying to **understand/debug** an existing webhook? → Use `stripe-webhook-debugger`
- Is the user trying to **write/generate** webhook code? → Use `stripe-webhook-code-generator`
- Both? → Use debugger first to understand, then generator to implement

**2. Stripe Webhook Debugger**

Use this skill when the user needs help with:
- Understanding what Stripe event types mean
- Debugging signature verification failures
- Analyzing webhook payloads
- Interpreting error messages
- Learning best practices for webhook security
- Testing webhooks locally with Stripe CLI

**Key capabilities you have:**
- Complete signature verification troubleshooting guide
- Comprehensive event type catalog with explanations
- Error code reference and solutions
- Testing patterns and local development setup
- Security best practices and architecture patterns

**3. Stripe Webhook Code Generator**

Use this skill when the user needs:
- Production-ready webhook handler code
- Integration with their existing codebase
- Framework-specific implementations
- Test generation
- Database/email/logging integrations

**Key capabilities you have:**
- Code templates for Next.js, Express, FastAPI, Django
- Codebase-aware generation (analyze their code style first)
- Test pattern generation (Jest, Pytest, etc.)
- Integration patterns for Prisma, SQLAlchemy, Resend, SendGrid
- Security-first approach with signature verification

**4. Best Practices for AI Assistance**

When helping users:

a. **Always verify signatures** - Never generate code that skips signature verification
b. **Analyze their codebase first** - Before generating code, understand their:
   - Framework and version
   - Database setup
   - Existing code style and patterns
   - Testing framework
c. **Match their conventions** - Generated code should feel native to their project
d. **Provide context** - Explain WHY certain patterns are recommended
e. **Test-driven** - Always include or offer to generate tests
f. **Environment-aware** - Use environment variables for secrets, never hardcode

**5. Common User Scenarios**

| User Says | Skill to Use | Action |
|-----------|--------------|--------|
| "Why is my webhook failing?" | Debugger | Analyze error, check signature verification, review payload |
| "Generate a webhook handler" | Code Generator | Analyze codebase, generate integrated code with tests |
| "What events should I listen to?" | Debugger | Explain relevant events for their use case |
| "Add X event to my webhook" | Code Generator | Update existing handler with new event logic |
| "This payload doesn't make sense" | Debugger | Explain payload structure and fields |
| "How do I test this locally?" | Debugger | Guide on Stripe CLI usage |

**6. Security Reminders**

Always ensure:
- Signature verification is implemented correctly
- Webhook secrets (whsec_*) are used, not API keys
- Raw request body is used for signature verification
- HTTPS is enforced in production
- Idempotency is considered for event processing
- Test mode vs live mode credentials are properly separated

---

## 🛠️ Installation Options

### Option 1: Via Claude Code (Recommended)

The easiest way if you're using Claude Code:

1. Fork and clone this repo
2. Enable skill-creator in Settings → Capabilities → Skills (in Claude Desktop)
3. Ask Claude to "Create a new skill" or "Install the Stripe webhook skills"

### Option 2: Manual Upload to Claude Projects

1. Create or open a Claude Project
2. Go to Project Knowledge
3. Upload the skill folders or individual .md files:
   - `stripe-webhook-debugger/SKILL.md` (+ reference files)
   - `stripe-webhook-code-generator/SKILL.md` (+ reference files)

### Option 3: Use Individual Reference Files

If you only need specific guidance:
- Upload just the reference files you need (signature-verification.md, event-types.md, etc.)
- Great for one-off questions or learning

---

## 📁 Repository Structure

```
stripe-webhook-skills/
├── stripe-webhook-debugger/
│   ├── SKILL.md                          # Main skill definition
│   └── references/
│       ├── signature-verification.md      # Signature debugging guide
│       ├── event-types.md                 # Stripe event catalog
│       ├── error-codes.md                 # Error reference
│       ├── testing.md                     # Local testing guide
│       ├── configuration.md               # Dashboard setup
│       ├── event-processing.md            # Processing patterns
│       └── best-practices.md              # Security & architecture
│
└── stripe-webhook-code-generator/
    ├── SKILL.md                          # Main skill definition
    └── references/
        ├── code-templates.md              # Framework templates
        └── testing-patterns.md            # Test generation
```

---

## 🔧 Supported Frameworks & Integrations

The Code Generator includes production-ready templates for:

**Web Frameworks:**
- Next.js 14 (App Router)
- Next.js 13 (Pages Router)
- Express.js
- FastAPI
- Django

**Databases:**
- Prisma
- SQLAlchemy

**Email Services:**
- Resend
- SendGrid

**Testing:**
- Jest
- Pytest

*More frameworks can be added by extending the templates.*

---

## ⚠️ Important Notes

**Security First**
- Both skills prioritize signature verification
- Never skip security checks in generated code
- Always use environment variables for secrets
- Enforce HTTPS in production

**Customization**
- All reference files are Markdown format
- Easily customizable for your needs
- Version control friendly
- Add your own patterns and templates

**Compatibility**
- Works with Claude 3.5 Sonnet and later
- Compatible with Claude Projects and Claude Code
- Can be used with or without skill-creator

---

## 🤝 Contributing

These skills are designed to be modular and extensible. Contributions are welcome!

**Ways to Contribute:**
- Add support for new frameworks
- Improve existing templates
- Add more debugging scenarios
- Expand the event type catalog
- Share your use cases

**To Contribute:**
1. Fork this repository
2. Create a feature branch
3. Make your improvements
4. Submit a pull request

---

## 📚 Resources

### Stripe Documentation
- [Webhooks Guide](https://stripe.com/docs/webhooks)
- [Event Types Reference](https://stripe.com/docs/api/events/types)
- [Signature Verification](https://stripe.com/docs/webhooks/signatures)
- [Stripe CLI](https://stripe.com/docs/cli)

### Stripe Dashboard
- [Webhooks Dashboard](https://dashboard.stripe.com/webhooks)
- [Events Dashboard](https://dashboard.stripe.com/events)
- [API Logs](https://dashboard.stripe.com/logs)

### Claude Code
- [Claude Code Documentation](https://docs.claude.com/claude-code)
- [Skills Documentation](https://docs.claude.com/claude-code/skills)

---

## 💡 Pro Tips

1. **Test locally first** - Use Stripe CLI for local development
2. **Test mode is your friend** - Always test thoroughly before production
3. **Read the references** - They contain detailed patterns and best practices
4. **Customize liberally** - Adapt templates to match your exact needs

---

## 📝 License

MIT License - Feel free to use, modify, and distribute these skills.

---

## 🙏 Acknowledgments

Built for the Claude Code community. Special thanks to:
- The Stripe team for excellent webhook documentation
- The Claude Code team for the skills framework
- Contributors and users who help improve these skills

---

**Version:** 1.0
**Last Updated:** October 2025
**Compatible With:** Claude 3.5 Sonnet and later

**Made with Claude Code** 🚀
