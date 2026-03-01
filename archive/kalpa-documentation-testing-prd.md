# PRD: Establishing Documentation & Testing Culture at Kalpa Inovasi Digital

**Author:** Alex  
**Date:** January 2025  
**Status:** Draft  
**Target Completion:** End of February 2025 (before Go Lite launch)

## 1. Executive Summary

### 1.1 Context
Kalpa's Go microservices codebase lacks comprehensive documentation and systematic testing patterns. As CEO and infrastructure lead, I want to contribute productively to the codebase while establishing documentation and testing culture for the team. Given team dynamics (hesitance to critique leadership) and my learning curve with the specific codebase, this requires careful guardrails and a structured approach.

### 1.2 Approach
1. Start with documentation (lower risk, high value, forces deep learning)
2. Use documentation work to discover and codify team conventions
3. Develop AI agents that follow our conventions
4. Contribute tests using established patterns
5. Eventually enable team-wide use of documentation/testing agents

### 1.3 Core Principle
**Every contribution must be net-positive for team productivity.** If my work creates more burden than value (debugging my code, diplomatic code reviews, refactoring my contributions), it fails.

## 2. Goals & Success Metrics

### 2.1 Primary Goals
1. **Establish living documentation system** that team actually uses and maintains
2. **Create testing patterns** that make it easier for team to write tests
3. **Build productive AI agents** that follow our conventions
4. **Make first code contributions** (tests) that team considers valuable
5. **Set cultural tone** around testing and documentation

### 2.2 Success Metrics

#### 2.2.1 Documentation Success
- [ ] Architecture docs updated and reviewed by team (100% accuracy)
- [ ] 3+ services documented with consistent pattern
- [ ] Team references docs when onboarding or debugging (observational)
- [ ] At least one team member uses documentation agent successfully

#### 2.2.2 Testing Success
- [ ] Gateway service has >70% unit test coverage
- [ ] Test patterns documented and used by team
- [ ] CI pipeline runs tests automatically
- [ ] My test PRs require ≤2 review cycles (shows quality/clarity)
- [ ] No tests I wrote get refactored within 1 month (shows good patterns)

#### 2.2.3 Cultural Success
- [ ] Team provides direct technical feedback on my PRs (not rubber-stamping)
- [ ] At least one PR gets significant critique that I implement
- [ ] Team initiates discussion about testing/docs without prompting

#### 2.2.4 Personal Learning
- [ ] Can trace request flow through gateway without assistance
- [ ] Understand data model well enough to design schema changes
- [ ] Can debug Go service issues using logs/metrics

## 3. Prerequisites & Setup

### 3.0 Phase 0: Foundation (Week 1, Days 1-2)

#### 3.0.1 Objective
Establish documentation infrastructure and baseline understanding

#### 3.0.2 Tasks
- [ ] **HUMAN:** Clone all repos locally, ensure dev environment works
- [ ] **HUMAN:** Read existing Google doc architecture overview completely
- [ ] **HUMAN:** Create `/docs` structure in main backend repo:
  ```
  /docs
    /architecture
      system-overview.md
      data-models.md
      integration-architecture.md
      /adrs
    /services
    /development
      setup.md
      conventions.md
      database-migrations.md
      testing-strategy.md
    /operations
  ```
- [ ] **HUMAN:** Create ADR template file (save as `/docs/architecture/adrs/000-template.md`)
- [ ] **HUMAN:** Create questions.md file in `/docs` root for tracking unknowns
- [ ] **AI:** Convert Google doc to markdown (faithful conversion, no updates yet)
  - Output: `/docs/architecture/system-overview.md`
  - Note all diagrams that need conversion
- [ ] **HUMAN:** Review converted markdown, add TODO comments for gaps/questions
- [ ] **AI:** Convert drawing.io diagrams to mermaid (preserve semantics, update syntax)
- [ ] **HUMAN:** Validate diagrams render correctly

#### 3.0.3 Success Criteria
- Documentation structure exists in repo
- Architecture doc converted and readable
- questions.md has 10+ specific questions ready for team

#### 3.0.4 Handoff
Share converted architecture doc with team for initial feedback before proceeding

## 4. Work Phases

### 4.1 Phase 1: Architecture Documentation & Validation (Week 1, Days 3-7)

#### 4.1.1 Objective
Update architecture docs with changes since August, validate understanding with team

#### 4.1.2 Tasks
- [ ] **HUMAN:** Review all commits/PRs from August to January (sample ~20%)
  - Note: major features, architectural changes, new services
  - Add notes to questions.md
- [ ] **HUMAN:** Interview team (async or quick calls):
  - "What changed architecturally since August?"
  - "What decisions did we make that should be documented?"
  - "What's missing from the architecture doc?"
- [ ] **HUMAN:** Create first ADRs for major changes identified:
  - [ ] ADR-001: [First major decision since August]
  - [ ] ADR-002: [Second major decision]
  - [ ] ADR-003: [Third major decision]
- [ ] **AI:** Update system-overview.md with identified changes
- [ ] **HUMAN:** Review and refine AI updates
- [ ] **HUMAN:** Create data-models.md with simple schema documentation:
  - Table → purpose mapping
  - Key relationships
  - Service ownership of tables
- [ ] **HUMAN:** Document database migration process in database-migrations.md:
  - Where migrations live (likely /backbone/migrations)
  - How to create new migration
  - How migrations run (auto on deploy? manual?)
  - Rollback strategy
- [ ] **HUMAN:** Create PR with all architecture documentation updates
  - PR Description: "Updated architecture docs with August-January changes. Reviewing for accuracy and completeness. Foundation for service-level documentation."

#### 4.1.3 Success Criteria
- Team reviews and approves architecture docs
- At least 3 ADRs created
- Questions.md items resolved or explicitly marked as "to investigate"
- Data model doc helps you understand schema relationships

#### 4.1.4 Handoff
Architecture docs approved → Begin service-level documentation

### 4.2 Phase 2: Gateway Service Deep Dive (Week 2)

#### 4.2.1 Objective
Deeply understand gateway service, document it, establish service documentation pattern

#### 4.2.2 Tasks - Understanding
- [ ] **HUMAN:** Trace single request through gateway (start with health check)
  - Document every function call, middleware, transformation
  - Note in questions.md anything unclear
- [ ] **HUMAN:** Map out gateway's:
  - Request flow (entry to exit)
  - Middleware chain (order and purpose)
  - Error handling patterns
  - Auth/authorization approach
  - Integration points (which services it calls)
  - External dependencies (SATU SEHAT, etc.)
- [ ] **HUMAN:** Understand the /services vs /services confusion:
  - Top-level /services = microservices
  - Inner /services = business logic layer (service layer pattern)
  - /handler = HTTP handlers (presentation layer)
  - Document this in architecture overview as common pattern

#### 4.2.3 Tasks - Convention Discovery
- [ ] **HUMAN:** Audit gateway codebase for conventions, document in `/docs/development/conventions.md`:
  - Error handling patterns (wrapping, custom types, logging)
  - Context usage (what's passed, timeout patterns)
  - Logging patterns (levels, structured logging)
  - Testing patterns (see existing tests if any)
  - Configuration patterns (env vars, config structs)
  - Database interaction patterns (repositories, transactions)
  - API contract patterns (validation, serialization)
- [ ] **HUMAN:** Share conventions.md with team:
  - "Here's what I'm observing. What's intentional vs accidental?"
  - "What am I missing?"
  - Refine based on feedback

#### 4.2.4 Tasks - Documentation Creation
- [ ] **HUMAN:** Write gateway README.md manually (first draft):
  - Service purpose
  - Request flow
  - Key components
  - Configuration
  - Local development setup
  - Known issues/gotchas
- [ ] **AI:** Generate documentation for 3-4 specific packages/functions in gateway
- [ ] **HUMAN:** Review AI-generated docs against your manual work:
  - Where does it match conventions?
  - Where does it miss the mark?
  - What prompt refinements needed?
- [ ] **HUMAN:** Iterate on AI prompts until output matches conventions
- [ ] **HUMAN:** Save refined prompts as documentation agent configuration

#### 4.2.5 Tasks - Team Validation
- [ ] **HUMAN:** Select ONE package/module within gateway for full documentation
- [ ] **AI:** Run documentation agent on selected module
- [ ] **HUMAN:** Review and refine output
- [ ] **HUMAN:** Create PR with:
  - Gateway README.md
  - One fully documented module
  - Updated conventions.md
  - PR Description: "Testing documentation pattern on gateway module. Does this level of detail/style work? Should I continue?"

#### 4.2.6 Success Criteria
- Can explain gateway request flow without looking at code
- Team approves documentation pattern and level of detail
- Documentation agent produces acceptable output with minimal human editing
- conventions.md reflects actual team practices

#### 4.2.7 Handoff
Team approves documentation approach → Scale to full gateway service

### 4.3 Phase 3: Scale Gateway Documentation & Develop Testing Patterns (Week 3)

#### 4.3.1 Objective
Complete gateway documentation, discover and document testing patterns

#### 4.3.2 Tasks - Complete Gateway Docs
- [ ] **AI:** Run documentation agent on remaining gateway packages
- [ ] **HUMAN:** Review all generated docs (spend 15-30min per package)
- [ ] **HUMAN:** Fix inaccuracies, refine unclear sections
- [ ] **HUMAN:** Ensure consistency across all gateway documentation
- [ ] **HUMAN:** Create PR with complete gateway documentation

#### 4.3.3 Tasks - Testing Pattern Discovery
- [ ] **HUMAN:** Audit existing tests across services:
  - What testing libraries are used? (testify, gomock, etc.)
  - How are tests organized? (same package, _test package?)
  - What's the naming convention? (Test_Function_Condition_Expected?)
  - Are there table-driven tests?
  - How are mocks created?
  - Are there test helpers/fixtures?
  - Integration test approach?
- [ ] **HUMAN:** Document findings in `/docs/development/testing-strategy.md`:
  - Overall testing philosophy
  - Unit vs integration test boundaries
  - Test organization
  - Naming conventions
  - Mock strategy
- [ ] **HUMAN:** Create service-specific `/docs/services/gateway/testing-patterns.md`:
  - Unit test examples from gateway
  - How to test handlers
  - How to test service layer
  - How to mock external dependencies
  - Common test fixtures for gateway

#### 4.3.4 Tasks - Write Example Tests
- [ ] **HUMAN:** Select 3 gateway functions that need tests
- [ ] **HUMAN:** Write unit tests manually for these functions:
  - Follow discovered patterns
  - Document your process/decisions
  - Note difficulties or questions
- [ ] **HUMAN:** Share test examples with team (informal review):
  - "Do these follow our patterns?"
  - "What would you do differently?"
  - "Are there test helpers I should use?"

#### 4.3.5 Success Criteria
- Gateway fully documented
- Testing patterns documented with real examples
- 3 example tests written and informally reviewed
- Clear understanding of team's testing preferences

#### 4.3.6 Handoff
Testing patterns documented and validated → Begin CI setup

### 4.4 Phase 4: CI/CD Infrastructure Setup (Week 3-4)

#### 4.4.1 Objective
Establish automated testing infrastructure, learn deployment process deeply

#### 4.4.2 Tasks - CI Setup
- [ ] **HUMAN:** Review existing deployment process:
  - How do changes reach production now?
  - What manual steps exist?
  - What could break?
- [ ] **HUMAN:** Design CI pipeline (document in `/docs/operations/ci-cd.md`):
  - Trigger conditions (all PRs? main branch only?)
  - Test execution (unit, integration, e2e?)
  - Linting/formatting checks
  - Build verification
  - Environment-specific configs
- [ ] **HUMAN:** Share CI design with team:
  - "Does this match our needs?"
  - "What am I missing?"
  - "Any concerns about build times/complexity?"
- [ ] **HUMAN:** Implement GitHub Actions workflow:
  - Start simple (just unit tests)
  - Add complexity incrementally
  - Document each addition in ci-cd.md
- [ ] **HUMAN:** Test CI with your example tests from Phase 3
- [ ] **HUMAN:** Add CI status badge to main README
- [ ] **HUMAN:** Document CI in `/docs/development/setup.md`:
  - How to run tests locally that CI runs
  - How to debug CI failures
  - How to skip CI (for urgent fixes)

#### 4.4.3 Tasks - Create ADRs for CI Decisions
- [ ] **HUMAN:** Document key CI decisions as ADRs:
  - [ ] ADR: Which CI platform and why
  - [ ] ADR: Test execution strategy
  - [ ] ADR: Branch protection rules
  - [ ] ADR: Deployment automation approach

#### 4.4.4 Decision Point: Documentation Agents in CI
- [ ] **HUMAN:** Discuss with team: Should documentation agents run in PR workflow?
  - Options:
    1. Manual only (devs run locally when needed)
    2. Automated suggestion (agent comments on PR with doc updates)
    3. Automated requirement (PR fails if docs outdated)
  - Document decision and reasoning
  - [ ] If automated: Implement documentation check in CI
  - [ ] If automated: Add "skip-docs-check" flag for urgent fixes

#### 4.4.5 Success Criteria
- CI pipeline running on all PRs
- Tests execute automatically
- Team understands how to use CI
- CI decisions documented as ADRs
- Clear policy on when/how docs are updated

#### 4.4.6 Handoff
CI working and documented → Begin systematic test contribution

### 4.5 Phase 5: Systematic Test Contribution to Gateway (Week 4-5)

#### 4.5.1 Objective
Increase gateway test coverage using established patterns, validate AI testing agent

#### 4.5.2 Tasks - Testing Strategy
- [ ] **HUMAN:** Analyze gateway test coverage:
  - Run coverage report (go test -cover)
  - Identify critical untested code
  - Prioritize: handlers > service layer > utilities
- [ ] **HUMAN:** Create testing roadmap in gateway/testing-patterns.md:
  - Current coverage: X%
  - Target coverage: 70%
  - Priority order for testing
  - Functions that are hard to test (document why)

#### 4.5.3 Tasks - Test Development Workflow
- [ ] **HUMAN:** For each testing session:
  - [ ] Select 3-5 related functions to test
  - [ ] Create feature branch: `test/gateway-[component-name]`
  - [ ] Write tests following documented patterns
  - [ ] Time-box at 4 hours maximum
  - [ ] If stuck >30min on one issue, document in questions.md and move on
  - [ ] Run tests locally, ensure they pass
  - [ ] Run coverage to see improvement
  - [ ] Create PR with:
    - Clear description of what's tested
    - Coverage improvement (before/after %)
    - Any questions/uncertainties
    - Request 2 approvals with requirement for substantive feedback

#### 4.5.4 Tasks - AI Testing Agent Development
- [ ] **AI:** Generate tests for 3-4 functions (parallel to your manual testing)
- [ ] **HUMAN:** Review AI-generated tests:
  - Do they follow patterns?
  - Do they actually test the right things?
  - Are they maintainable?
  - What's missing?
- [ ] **HUMAN:** Refine AI prompts based on review
- [ ] **HUMAN:** Document testing agent configuration
- [ ] **HUMAN:** Test agent on new functions, validate output quality
- [ ] **HUMAN:** Once confident, use agent to accelerate test writing:
  - Agent generates, you review/refine
  - Always run and validate tests locally
  - Never merge AI tests without understanding them

#### 4.5.5 Tasks - Iterative Improvement
- [ ] **HUMAN:** After each merged test PR:
  - Note what feedback you got
  - Update testing-patterns.md if new patterns emerge
  - Refine agent prompts if needed
  - Celebrate when team finds issues (shows they're reviewing seriously)
- [ ] **HUMAN:** Track metrics:
  - Review cycles per PR (target: ≤2)
  - Time from PR to merge (should decrease over time)
  - Coverage improvement per PR
  - Whether any tests get refactored later (flag for review)

#### 4.5.6 Milestone Targets
- [ ] 5 test PRs merged
- [ ] Gateway coverage >50%
- [ ] 10 test PRs merged  
- [ ] Gateway coverage >70%

#### 4.5.7 Success Criteria
- Gateway has ≥70% test coverage
- All critical paths tested
- Tests follow consistent patterns
- Team provides honest technical feedback
- Testing agent produces good tests with minimal human refinement
- You understand tested code deeply

#### 4.5.8 Handoff
Gateway well-tested → Expand to other services

### 4.6 Phase 6: Scale Documentation & Testing to Other Services (Week 5-6+)

#### 4.6.1 Objective
Apply proven patterns to additional services, enable team to use agents

#### 4.6.2 Tasks - Service Selection
- [ ] **HUMAN:** Prioritize services for documentation/testing:
  - Business criticality
  - Complexity
  - Frequency of changes
  - Team requests
- [ ] **HUMAN:** Document prioritization in `/docs/services/README.md`

#### 4.6.3 Tasks - Service Documentation
- [ ] For each service:
  - [ ] **HUMAN:** Understand service purpose and boundaries (1-2 hours)
  - [ ] **AI:** Generate initial service README using documentation agent
  - [ ] **HUMAN:** Review and refine (30-60min)
  - [ ] **AI:** Document service packages
  - [ ] **HUMAN:** Spot-check documentation (10-15min per package)
  - [ ] **HUMAN:** Create PR, get team review
  - [ ] **HUMAN:** If service has unique patterns, update conventions.md

#### 4.6.4 Tasks - Service Testing
- [ ] For each service:
  - [ ] **HUMAN:** Analyze current test coverage
  - [ ] **HUMAN:** Document testing patterns specific to this service
  - [ ] **AI + HUMAN:** Write tests using refined agent workflow
  - [ ] **HUMAN:** Target 60-70% coverage (not 100% - diminishing returns)
  - [ ] **HUMAN:** PRs follow same review process as gateway

#### 4.6.5 Tasks - Team Enablement
- [ ] **HUMAN:** Create agent usage guide in `/docs/development/using-ai-agents.md`:
  - When to use documentation agent
  - When to use testing agent
  - How to review agent output
  - Common pitfalls
  - How to refine prompts
- [ ] **HUMAN:** Pair with team member to use agents:
  - Walk through documentation agent
  - Walk through testing agent
  - Get feedback on usability
  - Refine based on feedback
- [ ] **HUMAN:** Share agent configurations in repo (`.claude/` or similar)
- [ ] **HUMAN:** Monitor team agent usage:
  - Who's using them?
  - What problems do they encounter?
  - What improvements needed?

#### 4.6.6 Tasks - Backbone/Migrations (Last)
- [ ] **HUMAN:** Study database migrations thoroughly:
  - Read all migration files
  - Understand schema evolution
  - Document migration patterns
- [ ] **HUMAN:** Document in database-migrations.md:
  - Current schema state
  - Migration history/rationale
  - How to add migrations safely
  - Rollback procedures
  - Common migration gotchas
- [ ] **HUMAN:** Create data dictionary if not done yet:
  - Update data-models.md with complete schema
  - Document constraints, indexes, triggers
  - Note service ownership boundaries

#### 4.6.7 Success Criteria
- 3+ services documented and tested beyond gateway
- Team actively uses agents
- Documentation stays current (check 1 month after completing)
- Test coverage across services averages >60%
- Agents are part of normal workflow

#### 4.6.8 Handoff
Infrastructure mature → Ongoing maintenance mode

## 5. Guardrails & Validation

### 5.1 Personal Guardrails

#### 5.1.1 Before creating any PR
- [ ] Have I time-boxed this work? (Stuck >4hrs = ask for help)
- [ ] Can I explain this code/decision to my team?
- [ ] Does this follow documented conventions?
- [ ] Have I tested this locally?
- [ ] Is my PR description clear about what I want feedback on?

#### 5.1.2 During PR review
- [ ] Am I getting substantive feedback or rubber-stamps?
- [ ] If rubber-stamped: Explicitly ask for specific critique
- [ ] When criticized: Thank them, implement their suggestion
- [ ] Track: Are review cycles decreasing over time?

#### 5.1.3 After PR merge
- [ ] Does merged code get refactored within 1 month?
- [ ] If yes: What pattern did I miss? Update documentation.
- [ ] Did this PR help or burden the team?
- [ ] What did I learn?

### 5.2 Team Protection Mechanisms

#### 5.2.1 Establish upfront
- [ ] Designate one senior dev as "code mentor" - their job is to critique
- [ ] Require 2 approvals on all PRs with substantive feedback required
- [ ] Set expectation: "Review my PRs as harshly as you would a junior dev"
- [ ] Create escape hatch: PRs sitting >2 days can be closed with explanation

#### 5.2.2 Cultural signals
- [ ] Celebrate when team finds bugs in my code
- [ ] Always implement suggested changes, even if I disagree (learn their way first)
- [ ] Never defend my code - ask questions to understand their perspective
- [ ] Share failures openly: "I got stuck on X, learned Y"

### 5.3 Quality Checks

#### 5.3.1 Documentation quality
- [ ] Is this accurate? (Validate with team)
- [ ] Is this useful? (Would this help someone onboarding?)
- [ ] Is this maintainable? (Can team keep it updated?)
- [ ] Is this the right level of detail? (Not too sparse, not too verbose)

#### 5.3.2 Test quality
- [ ] Do tests actually test what they claim?
- [ ] Are tests readable and maintainable?
- [ ] Do tests follow team patterns?
- [ ] Would these tests catch real bugs?
- [ ] Am I testing implementation or behavior?

#### 5.3.3 AI output quality
- [ ] Does agent output require heavy editing? (If yes: refine prompts)
- [ ] Does output follow conventions? (If no: update agent config)
- [ ] Can team use agents successfully? (If no: improve documentation)

## 6. Open Questions & Decision Points

### 6.1 Questions to resolve during execution

#### 6.1.1 Technical
- [ ] What's the actual test database setup for integration tests?
- [ ] How do we mock SATU SEHAT in tests?
- [ ] What performance test thresholds should exist?
- [ ] How do frontend and BFF interact with gateway? (Document boundaries)

#### 6.1.2 Process
- [ ] Should branch protection rules require passing tests before merge?
- [ ] Who reviews documentation PRs? (Same reviewers as code?)
- [ ] How often should docs be audited for staleness?
- [ ] Should ADRs require team consensus or just notification?

#### 6.1.3 Cultural
- [ ] How do we prevent docs from rotting after I establish them?
- [ ] Should code PRs that affect architecture require doc updates?
- [ ] How do we know if team actually finds docs useful?
- [ ] What's the right level of involvement for me long-term?

#### 6.1.4 Tooling
- [ ] Which testing libraries should be standard?
- [ ] Should we use test coverage minimums as gates?
- [ ] What CI/CD platform features do we need?
- [ ] How should agents be distributed to team? (Repo files? Shared configs?)

### 6.2 Decision Log
Track decisions made during execution:

| Date | Decision | Context | Owner |
|------|----------|---------|-------|
| | | | |

## 7. Appendix: Templates & Examples

### 7.1 ADR Template
```markdown
# ADR-XXX: [Title]

**Date:** YYYY-MM-DD  
**Status:** Proposed | Accepted | Superseded | Deprecated  
**Deciders:** [Names]

## Context
What's the issue we're trying to solve?
What's the context around this decision?

## Decision
What did we decide to do?
Be specific and concrete.

## Consequences

### Positive
What becomes easier or better?

### Negative
What becomes harder or worse?

### Neutral
What changes without clear positive/negative?

## Alternatives Considered
What else did we evaluate?
Why did we reject those options?

## References
Links to relevant discussions, docs, or code.
```

### 7.2 Service README Template
```markdown
# [Service Name]

## Purpose
What does this service do?
What business problem does it solve?

## Architecture
How does this fit in the overall system?
What services does it depend on?
What services depend on it?

## Key Components
- `/handlers` - HTTP request handlers
- `/services` - Business logic layer
- `/repository` - Data access layer
- `/models` - Data structures

## Configuration
Required environment variables:
- `ENV_VAR_1` - Description
- `ENV_VAR_2` - Description

## Local Development

### Setup
```bash
# Commands to set up locally
```

### Running Tests
```bash
# Commands to run tests
```

### Common Issues
- Problem X → Solution Y
- Problem A → Solution B

## API Endpoints
Document key endpoints or link to API docs.

## Monitoring & Operations
- Logs: Where to find them
- Metrics: What to watch
- Alerts: What fires when things break

## Related Documentation
- [Architecture Overview](../architecture/system-overview.md)
- [Testing Patterns](./testing-patterns.md)
```

### 7.3 PR Description Template for Learning PRs
```markdown
## What This PR Does
[Clear description of changes]

## Context
[Why I'm making these changes, what I'm learning]

## What I'm Uncertain About
- [ ] Question 1
- [ ] Question 2

## Review Focus
Please pay special attention to:
- [ ] Does this follow our conventions?
- [ ] Is this the right approach?
- [ ] What would you do differently?

## Testing
- [ ] Tests added/updated
- [ ] Tests pass locally
- [ ] Coverage before: X%
- [ ] Coverage after: Y%

## Checklist
- [ ] Follows conventions.md
- [ ] Documentation updated
- [ ] No sensitive data committed
```

## 8. Notes for Ralphloop Integration

### 8.1 Iteration Structure
Each phase above can be broken into ralphloop iterations:

#### 8.1.1 Iteration components
1. **Input:** Previous phase outputs + current task list
2. **Execution:** AI agent performs tasks marked AI, outputs results
3. **Validation:** Human reviews outputs against success criteria
4. **Refinement:** Human provides feedback, agent refines
5. **Checkpoint:** Save state, update questions.md, proceed or pause

### 8.2 Error/Question Logging
For ralphloop automation, maintain:
- `/docs/ralphloop-errors.md` - Issues encountered during automation
- `/docs/ralphloop-questions.md` - Questions needing human decision
- `/docs/ralphloop-refinements.md` - Prompt improvements over time

### 8.3 Handoff Protocol
Between phases, ralphloop should:
1. Summarize phase completion
2. List success criteria met/unmet
3. Flag open questions requiring human input
4. Propose next phase or pause for human review

---

**END OF PRD**
