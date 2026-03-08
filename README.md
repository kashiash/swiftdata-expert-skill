# SwiftData Expert Skill

A comprehensive AI agent skill for SwiftData development on iOS 17+, macOS 14+, and other Apple platforms. This skill provides concise, code-focused guidance for building production-ready SwiftData applications with SwiftUI.

**Author:** [@kashiash](https://github.com/kashiash)  
**Version:** 1.0.0  
**License:** MIT

## Quick Start

### Install to all detected agents at once

```bash
npx skills add https://github.com/kashiash/swiftdata-expert-skill --skill swiftdata-expert --agent '*' -g -y
```

### Install to a specific agent

```bash
# Claude Code
npx skills add https://github.com/kashiash/swiftdata-expert-skill --skill swiftdata-expert -g -a claude-code

# Cursor
npx skills add https://github.com/kashiash/swiftdata-expert-skill --skill swiftdata-expert -g -a cursor

# Kiro CLI
npx skills add https://github.com/kashiash/swiftdata-expert-skill --skill swiftdata-expert -g -a kiro-cli

# GitHub Copilot
npx skills add https://github.com/kashiash/swiftdata-expert-skill --skill swiftdata-expert -g -a github-copilot

# Windsurf
npx skills add https://github.com/kashiash/swiftdata-expert-skill --skill swiftdata-expert -g -a windsurf
```

### Install to multiple specific agents

```bash
npx skills add https://github.com/kashiash/swiftdata-expert-skill --skill swiftdata-expert -g -a claude-code -a cursor -a kiro-cli
```

### Manual Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/kashiash/swiftdata-expert-skill.git
   ```

2. Copy the `swiftdata-expert` folder to the skills directory for your agent:

| Agent | Global path | Project path |
|-------|------------|--------------|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Cursor | `~/.cursor/skills/` | `.agents/skills/` |
| Kiro CLI | `~/.kiro/skills/` | `.kiro/skills/` |
| GitHub Copilot | `~/.copilot/skills/` | `.agents/skills/` |
| Windsurf | `~/.codeium/windsurf/skills/` | `.windsurf/skills/` |
| Continue | `~/.continue/skills/` | `.continue/skills/` |
| OpenHands | `~/.openhands/skills/` | `.openhands/skills/` |
| Cline | `~/.agents/skills/` | `.agents/skills/` |
| Gemini CLI | `~/.gemini/skills/` | `.agents/skills/` |
| Universal | `~/.config/agents/skills/` | `.agents/skills/` |

## What's Included

This skill covers all essential SwiftData topics:

- **Models & Attributes** - @Model macro, property types, attributes, transformers
- **Relationships** - One-to-one, one-to-many, many-to-many, delete rules
- **Queries & Predicates** - @Query, #Predicate, FetchDescriptor, filtering, sorting
- **Context & Saves** - ModelContext, ModelContainer, autosave, transactions
- **SwiftUI Integration** - @Query, @Bindable, dynamic sorting/filtering
- **Performance** - Batch operations, faulting, prefetching, optimization
- **CloudKit Sync** - iCloud setup, requirements, troubleshooting
- **Migrations** - Schema changes, migration plans, versioning
- **Testing & Previews** - In-memory containers, unit tests, preview data

## Usage

Once installed, AI coding assistants can automatically reference this skill when working on SwiftData projects. The skill provides:

- **60-second triage** for quick problem diagnosis
- **Routing map** to find relevant solutions fast
- **Code examples** for every common scenario
- **Common mistakes** and how to avoid them
- **Best practices** from production experience
- **Troubleshooting guides** for silent failures

## Target Audience

- iOS/macOS developers using SwiftData
- Teams migrating from Core Data to SwiftData
- Developers building multi-device sync apps
- Anyone learning SwiftData for the first time

## Requirements

- iOS 17.0+ / macOS 14.0+ / tvOS 17.0+ / watchOS 10.0+ / visionOS 1.0+
- Xcode 15.0+
- Swift 5.9+
- SwiftUI

## Knowledge Sources

This skill compilation is based on publicly available knowledge from the following sources:

### Primary Reference Materials

- **SwiftData by Example** by Paul Hudson
  - Comprehensive book covering all SwiftData features
  - Website: [hackingwithswift.com](https://www.hackingwithswift.com)
  - Used as primary reference for patterns and examples

- **Apple SwiftData Documentation**
  - Official Apple documentation and API references
  - WWDC23 Session 10187: Meet SwiftData
  - WWDC23 Session 10195: Build an app with SwiftData
  - Developer documentation at [developer.apple.com](https://developer.apple.com/documentation/swiftdata)

### Additional Reference Articles

The following publicly available articles were consulted as reference materials:

**Architecture and Patterns:**
- "The Art of SwiftData in 2025: From Scattered Pieces to a Masterpiece" by Mathis Gaignet
- "SwiftData Architecture: Patterns & Practices" by Mohammad Azam
- "Deep dive into dynamic SwiftData queries" by Mathis Gaignet

**Queries and Predicates:**
- "How to Dynamically Construct Complex Predicates for SwiftData" by fatbobman
- "How to Handle Optional Values in SwiftData Predicates"
- "Predicate Secret in Swift: Why One Line Changes Everything"
- "SwiftData Query Macro: Efficient Search in iOS17 & SwiftUI5"

**Performance and Best Practices:**
- "Best Practices: SwiftData Performance, Predicates, Filtering & Search"
- "Auto-save Records in SwiftData & SwiftUI"
- "Sorting Options in SwiftData & SwiftUI"

**Practical Guides:**
- "SwiftData: Praktyczny przewodnik - Views, Lists, Queries"
- "My First Foray into SwiftData: 5 Things That Surprised Me"
- "Data Persistence in iOS: Core Data, Realm & SwiftData Compared"

**Migration and Troubleshooting:**
- "Rozwiązywanie problemów migracji CoreData → SwiftData: Obowiązkowe atrybuty i walidacja"

**SwiftData by Example Series (Polish translations):**
- Parts 1-10, 19: Comprehensive tutorial series covering all aspects of SwiftData

### Related Medium Articles

For Polish-language articles and tutorials on SwiftData, visit:
- [Medium - SwiftData Tag](https://medium.com/tag/swiftdata)
- Author's Medium profile: [@kashiash](https://medium.com/@kashiash)

### Skill Format

- Structure inspired by the [Agent Skills open format](https://github.com/AvdLee/Core-Data-Agent-Skill)
- Format based on Antoine van der Lee's Core Data Agent Skill

**Note:** This is an independent compilation created by [@kashiash](https://github.com/kashiash). The authors of the referenced materials are not affiliated with or endorsing this project.

## License

MIT License - Copyright (c) 2024 [@kashiash](https://github.com/kashiash)

See [LICENSE](LICENSE) file for full license text.

**Important:** This skill is an independent educational compilation. All referenced source materials (books, articles, documentation) retain their original copyrights and licenses. This project does not claim ownership of the referenced materials and is not affiliated with or endorsed by the original authors.

## Feedback and Contributing

### Found an Issue?

Open an issue at: https://github.com/kashiash/swiftdata-expert-skill/issues

### Want to Contribute?

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit your changes: `git commit -am 'Add new pattern'`
4. Push to the branch: `git push origin feature/your-feature`
5. Open a Pull Request

**Contribution Guidelines:**
- Ensure code examples compile with latest Xcode
- Follow existing format and style
- Credit original sources appropriately
- Test examples before submitting
- Update relevant documentation

## Publishing This Skill (For Maintainers)

### Initial Setup

```bash
cd skills/swiftdata-expert
git init
git add .
git commit -m "Initial release: SwiftData Expert Skill v1.0.0"
git branch -M main
git remote add origin https://github.com/kashiash/swiftdata-expert-skill.git
git push -u origin main
```

### Creating a Release

1. Go to: https://github.com/kashiash/swiftdata-expert-skill/releases
2. Click "Create a new release"
3. Tag: `v1.0.0`
4. Title: "SwiftData Expert Skill v1.0.0"
5. Description: Copy from "What's Included" section
6. Publish release

### Updating the Skill

```bash
# Make changes to files
git add .
git commit -m "Update: description of changes"
git push

# Create new release on GitHub with incremented version
```

## Version History

- **1.0.0** (2024) - Initial release
  - Complete coverage of SwiftData fundamentals
  - 9 detailed section files
  - Code examples for all common scenarios
  - Troubleshooting guides
  - Performance optimization patterns
  - CloudKit sync guidance
  - Migration strategies
  - Testing and preview patterns

## Related Skills

- Core Data Agent Skill (for Core Data projects)
- SwiftUI Expert Skill (for SwiftUI patterns)
- iOS Architecture Skill (for app architecture patterns)

## Support

### Questions About SwiftData?
- Check [Apple's SwiftData Documentation](https://developer.apple.com/documentation/swiftdata)
- Read [SwiftData by Example](https://www.hackingwithswift.com) by Paul Hudson
- Watch WWDC23 Sessions: [10187](https://developer.apple.com/videos/play/wwdc2023/10187/), [10195](https://developer.apple.com/videos/play/wwdc2023/10195/)

### Questions About This Skill?
- Open an issue: https://github.com/kashiash/swiftdata-expert-skill/issues
- Check existing issues for solutions
- Review the SUMMARY.md for detailed information

### Questions About AI Coding Assistants?
- Kiro: Check Kiro documentation
- Cursor: Check Cursor documentation
- Other assistants: Refer to their respective docs

## Links

- **GitHub Repository**: https://github.com/kashiash/swiftdata-expert-skill
- **Issues**: https://github.com/kashiash/swiftdata-expert-skill/issues
- **Releases**: https://github.com/kashiash/swiftdata-expert-skill/releases
- **Author**: [@kashiash](https://github.com/kashiash)
- **Medium Articles**: [@kashiash on Medium](https://medium.com/@kashiash)

---

**Note**: This is a community-maintained skill for AI coding assistants. It is not officially affiliated with Apple Inc. or any of the source authors, though it respectfully builds upon their excellent work.
