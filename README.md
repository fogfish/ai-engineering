# AI Engineering

Bootstraps and templates for speckit (Golang)

| Folder       | Governs                                                                                            | Architecture                                               |
| ------------ | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| [cli/](cli/) | Cobra-based command-line applications                                                              | hexagonal/onion, Cobra as the primary adapter              |
| [cap/](cap/) | large, complex libraries ("capabilities"): several processes, pluggable technologies, multi-module | hexagonal/onion, public core and ports, internal use-cases |
| [lib/](lib/) | small, focused libraries: one problem, one vocabulary, one module                                  | flat package, standard-library interfaces                  |

Each folder holds a `constitution.md` (copy to `.specify/memory/constitution.md`) and `adrs/` (copy to the project's `adrs/`). A `lib` project graduates to `cap` per `lib/adrs/001-system-architecture.md`; principle numbering is shared between the two.

All materials are written to be adopted, verbatim or with light edits, by any open-source library project. It contains no references to a specific product, company, or cloud account.



## Quick Start

Follow `https://github.com/github/spec-kit` to install speckit into your project. Then copy constitution to your project

```bash
# Clone templates
git clone https://github.com/fogfish/ai-engineering
   
# Copy constitution
cp ai-engineering/{fea}/constitution.md {your-project}/.specify/memory

# Copy adrs
cp -R ai-engineering/{fea}/adrs {your-project}/ 

# Adopt the consitution via spec-kit agent 
/speckit-constitution Adopt the new version across all project file, templates and agents. The constitution file .specify/memory/constitution.md was modified inline by user. It all principles the project MUST adhere. 
```

## License

[![See LICENSE](https://img.shields.io/github/license/fogfish/ai-engineering.svg?style=for-the-badge)](LICENSE)
