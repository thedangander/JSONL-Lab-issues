# JSONL-Lab-issues
Issue only repo for my JSONL-Lab VSCode extension

Report any bugs and request features here.
I created this extension only really to fill an empty weekend with an experiment, but if you find it useful and find something doesn't work like you think it should, or if you've got an idea for an improvement or cool feature, let me know and I'll look at it on my next empty weekend.

## Feature Implementation Guides

For contributors looking to implement requested features, we've created detailed technical documentation:

### Column Resize Feature (Issue #[pending])
- **[Feature Specification](./FEATURE_SPEC_COLUMN_RESIZE.md)** - Complete technical specification with requirements, data structures, and design patterns
- **[Implementation Guide](./IMPLEMENTATION_GUIDE.md)** - Step-by-step instructions for implementing the feature
- **[Quick Reference](./QUICK_REFERENCE.md)** - TL;DR version with code snippets and common patterns

These documents provide everything needed to implement drag-resizable columns with smart defaults:
- Timestamp and Level columns auto-fit to content
- Message column expands to fill available space  
- User can drag column borders to resize
- Column widths persist across sessions
- Optional column settings UI for configuration
