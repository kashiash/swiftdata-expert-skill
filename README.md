# SwiftData Expert Skill

A comprehensive AI agent skill for SwiftData development on iOS 17+, macOS 14+, and other Apple platforms. This skill provides concise, code-focused guidance for building production-ready SwiftData applications with SwiftUI.

## Installation

### Using npx (Recommended)

```bash
npx skills add https://github.com/kashiash/swiftdata-expert-skill --skill swiftdata-expert
```

### Manual Installation

1. Clone or download this repository
2. Copy the `swiftdata-expert` folder to your skills directory:
   - User-level: `~/.kiro/skills/`
   - Workspace-level: `.kiro/skills/`

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

## Credits and Sources

This skill is based on knowledge from multiple authoritative sources:

### Primary Sources

- **SwiftData by Example** by Paul Hudson (@twostraws)
  - Comprehensive book covering all SwiftData features
  - Website: [hackingwithswift.com](https://www.hackingwithswift.com)
  - Twitter: [@twostraws](https://twitter.com/twostraws)

- **Apple SwiftData Documentation**
  - Official Apple documentation and WWDC sessions
  - WWDC23 Session 10187: Meet SwiftData
  - WWDC23 Session 10195: Build an app with SwiftData


## Contributing

Contributions are welcome! Please:

1. Ensure accuracy of technical content
2. Provide code examples that compile and run
3. Credit original sources appropriately
4. Follow the existing format and style
5. Test examples with latest Xcode/iOS versions

## Feedback

Found an issue or have a suggestion? Please open an issue on GitHub or contact the maintainers.

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





---

**Note**: This is a community-maintained skill for AI coding assistants. It is not officially affiliated with Apple Inc. or any of the source authors, though it respectfully builds upon their excellent work.
