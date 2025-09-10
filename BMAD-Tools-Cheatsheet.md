# BMad Method: Tools Cheatsheet by Phase

## 🎯 **PHASE 1: PLANNING (Web UI)**

### **Primary Tools**
- **Gemini Web** (1M+ tokens) - Best for large document generation
- **ChatGPT** - Alternative for comprehensive planning
- **Claude** - Good for analytical tasks

### **Agent Commands**
```bash
@analyst
*brainstorm {topic}              # Structured brainstorming
*create-project-brief           # Project discovery document
*perform-market-research        # Market analysis

@pm
*create-prd                     # Product Requirements Document
*correct-course                 # Adjust PRD if needed

@ux-expert
*create-frontend-spec           # UI/UX specification
*generate-ai-frontend-prompt    # For v0/Lovable generation

@architect
*create-architecture            # Technical architecture
*technical-research             # Deep tech research

@po
*validate-artifacts             # Document validation
*shard-doc {document}          # Create sharded versions
```

### **External Tools**
- **v0.dev** - AI UI generation (React/Next.js)
- **Lovable** - Full-stack app generation
- **Figma** - Manual UI design
- **Miro/Mural** - Brainstorming sessions

---

## 🚀 **PHASE 2: DEVELOPMENT (IDE)**

### **Primary Tools**
- **Cursor** - Native AI integration (recommended)
- **VS Code + GitHub Copilot** - Alternative IDE setup
- **Claude Code** - Anthropic's IDE
- **Windsurf** - Built-in AI capabilities

### **Agent Commands**
```bash
@sm
*draft                         # Create next story
*story-checklist              # Validate story completeness

@dev
*implement-story              # Code implementation
*update-file-list            # Track changes

@qa
*review-story                # Code review & testing
*create-test-plan           # Comprehensive testing
```

### **Development Tools**
- **Git** - Version control
- **Docker** - Containerization
- **ESLint/Prettier** - Code quality
- **Jest/Cypress** - Testing frameworks
- **Postman/Insomnia** - API testing

---

## 🔄 **PHASE 3: ITERATION (IDE/Web UI)**

### **Primary Tools**
- **IDE** - For code changes and reviews
- **Web UI** - For retrospectives and planning

### **Agent Commands**
```bash
@po
*epic-retrospective           # Review completed epic
*plan-next-epic              # Plan next iteration
```

### **Project Management Tools**
- **GitHub Issues** - Story tracking
- **Linear** - Project management
- **Notion** - Documentation
- **Slack/Discord** - Team communication

---

## 🛠️ **PROJECT TYPE TOOL STACKS**

### **Full-Stack Applications**
```
Planning: Gemini → v0.dev → Figma
Development: Cursor → Docker → AWS/Vercel
Testing: Jest → Cypress → Postman
```

### **UI/Frontend Only**
```
Planning: Gemini → Figma → Component libraries
Development: Cursor → Vite/Next.js → Vercel/Netlify
Testing: Jest → Storybook → Lighthouse
```

### **Backend/API Services**
```
Planning: Gemini → API design tools
Development: Cursor → Docker → AWS/GCP
Testing: Jest → Postman → Load testing tools
```

### **Brownfield Projects**
```
Analysis: Gemini → Code analysis tools
Development: Cursor → Git → CI/CD
Testing: Existing test suites → New tests
```

---

## ⚡ **QUICK COMMAND REFERENCE**

### **Start New Project**
```bash
@analyst → *create-project-brief
@pm → *create-prd
@ux-expert → *create-frontend-spec (if UI)
@architect → *create-architecture
@po → *validate-artifacts → *shard-doc
```

### **Development Cycle**
```bash
@sm → *draft
@dev → *implement-story
@qa → *review-story (optional)
```

### **Project Completion**
```bash
@po → *epic-retrospective
```

---

## 🎛️ **TOOL SELECTION MATRIX**

| Phase | Primary Tool | Secondary | When to Use |
|-------|-------------|-----------|-------------|
| **Planning** | Gemini Web | ChatGPT/Claude | Large documents, brainstorming |
| **UI Design** | v0.dev | Figma/Lovable | AI generation vs manual design |
| **Development** | Cursor | VS Code + Copilot | Native AI vs extension-based |
| **Testing** | Jest | Cypress/Playwright | Unit vs E2E testing |
| **Deployment** | Vercel | AWS/Docker | Simple vs complex infrastructure |

---

## 🚨 **CRITICAL TOOL TIPS**

1. **Web UI for Planning:** Use Gemini's 1M+ token context for comprehensive documents
2. **IDE for Development:** Cursor provides best AI integration for real-time coding
3. **External AI Tools:** v0.dev for rapid UI prototyping
4. **Version Control:** Always use Git for tracking changes
5. **Testing:** Start with Jest, add E2E testing as needed

---

## 📱 **MOBILE/QUICK ACCESS**

### **Essential Commands**
- `@analyst` → `*create-project-brief`
- `@pm` → `*create-prd`
- `@sm` → `*draft`
- `@dev` → `*implement-story`

### **Emergency Tools**
- **Git** - Always available for version control
- **Docker** - Consistent development environments
- **Postman** - Quick API testing
- **Browser DevTools** - Frontend debugging

This cheatsheet provides quick access to the right tools for each phase of your BMad project.
