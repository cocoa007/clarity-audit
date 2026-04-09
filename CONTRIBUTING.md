# Contributing to Clarity Audit

Thank you for your interest in contributing to the Clarity Audit project! This document provides guidelines for contributing audit findings, patterns, and improvements.

## How to Contribute

### Reporting Issues

If you find errors in existing audits or have suggestions for improvements:

1. Check existing issues to avoid duplicates
2. Open a new issue with:
   - Clear description of the problem or suggestion
   - Reference to the specific contract or pattern
   - Supporting evidence (contract code, transaction IDs, etc.)

### Adding New Audits

To add a new contract audit:

1. **Analyze the contract** using the [Clarity REPL](https://docs.stacks.co/docs/clarity/) or [Clarinet](https://docs.hiro.so/clarinet)
2. **Document findings** following the format in `audits/template.md`
3. **Generate HTML** using the build script (if applicable)
4. **Submit a PR** with:
   - Contract name and deployer address
   - Summary of findings (critical/high/medium/low/info counts)
   - Detailed analysis with code references
   - Recommendations for fixes

### Adding New Patterns

To document a reusable pattern:

1. **Identify the pattern** from production contracts
2. **Document** in `patterns/` directory following the existing format
3. **Include**:
   - Pattern name and description
   - Use cases and examples
   - Security considerations
   - Code snippets from real contracts

### Improving Existing Content

- Fix typos or unclear explanations
- Add missing context or references
- Update audit findings if new information becomes available
- Improve HTML styling or accessibility

## Audit Standards

### Severity Levels

- **Critical**: Immediate risk of fund loss or contract bricking
- **High**: Significant risk under specific conditions
- **Medium**: Moderate risk or important best practice violation
- **Low**: Minor issue or gas optimization
- **Info**: Educational note or suggestion

### Documentation Format

Each audit should include:

```markdown
# Contract Name

**Deployer**: `SP...`  
**Contract ID**: `SP....contract-name`  
**Source**: [GitHub/explorer link]

## Summary
- Critical: X
- High: X
- Medium: X
- Low: X
- Info: X

## Findings

### [FINDING-001] Title
**Severity**: Critical/High/Medium/Low/Info

Description of the issue...

**Code Reference**:
```clarity
Relevant code snippet
```

**Recommendation**:
Suggested fix...

**Status**: Open/Fixed/Acknowledged
```

## Development Setup

```bash
# Clone the repository
git clone https://github.com/cocoa007/clarity-audit.git
cd clarity-audit

# Install dependencies (if any build tools are added)
npm install

# Build HTML files (if build script exists)
npm run build
```

## Style Guidelines

- Use clear, concise language
- Include code references with line numbers when possible
- Provide actionable recommendations
- Cite sources for patterns (original contract, documentation)
- Use American English spelling

## Review Process

1. All contributions require review before merging
2. Audits will be checked for accuracy and completeness
3. Patterns should be verified against production usage
4. HTML generation (if applicable) must not break existing pages

## Questions?

- Open an issue for discussion
- Reach out to the maintainer: cocoa007

## License

By contributing, you agree that your contributions will be licensed under the same license as the project.

---

**Thank you for helping make Stacks/Clarity contracts more secure!**
