# Moqui AI Skill (moqui-ai-skill)
**AI-ready documentation skill for the Moqui Framework (model-agnostic)**

Moqui AI Skill is a **documentation-only component** that helps **any AI model or coding assistant** generate **framework-compliant Moqui artifacts** such as **entities, services, screens, REST resources, and tests** by providing structured, Moqui-specific guidance aligned with **Moqui 3.x XSD schemas** and common patterns used across **Mantle, SimpleScreens, and MarbleERP**.

## What is Moqui AI Skill?

Moqui Framework uses an XML-driven architecture and conventions that general-purpose AI tools often do not infer correctly. Moqui AI Skill provides a clear reference set that an AI assistant can follow to reliably produce valid outputs.

Use this skill to help an AI assistant:

- Generate valid **entity** and **view-entity** definitions
- Create correct **service** definitions and XML actions
- Build **screens** (form-single, form-list, transitions, subscreens)
- Configure **REST APIs** via `.rest.xml`
- Follow **Mantle UDM/USL** conventions and safe extension patterns
- Create **Spock tests** and end-to-end flows

## Model compatibility (AI vendor neutral)

This repository works with **any AI model** that can read Markdown documentation. Examples include Claude, OpenAI models, Google Gemini, Mistral, LLaMA, DeepSeek, Grok, Perplexity, and local LLMs (Ollama, LM Studio), plus enterprise or custom in-house models.

The key requirement is simple: **point your AI assistant to `SKILL.md` and the `references/` folder.**

## Why use moqui-ai-skill?

Without Moqui-specific context, AI assistants commonly produce:

- Invalid Moqui XML (wrong elements, missing required attributes, incorrect schema assumptions)
- Broken services (incorrect naming conventions, missing params, wrong transactions)
- Non-functional screens (invalid widgets, broken transitions, incorrect subscreens wiring)
- Mantle violations (editing core Mantle entities instead of extending safely)
- Missing framework integration points (SECA/EECA hooks, artifact auth, REST configuration)

Moqui AI Skill reduces these errors by giving the AI a structured, Moqui-native reference to follow.

## What’s included

| File | Purpose |
|------|---------|
| `SKILL.md` | Core instruction layer for AI usage and schema mapping |
| `references/ENTITIES.md` | Entity development reference (types, relationships, extend-entity, view-entity, EECA) |
| `references/SERVICES.md` | Service reference (params, actions, SECA, transactions, REST via `.rest.xml`) |
| `references/SCREENS.md` | Screen reference (forms, transitions, widgets, subscreens) |
| `references/MANTLE.md` | Mantle UDM/USL guidance (key entities, flows, conventions) |
| `references/MARBLE_ERP.md` | MarbleERP extension patterns and module structure |
| `references/TESTING.md` | Testing patterns (Spock tests for entities/services/screens/REST/flows) |

## Installation

### Using Gradle
```bash
cd moqui
./gradlew getComponent -Pcomponent=moqui-ai-skill
```

### Manual

```bash
cd moqui/runtime/component
git clone git@github.com:opdd-tech/moqui-ai-skill.git
```

## How to use with any AI assistant

### Step 1: Install the component

Add `moqui-ai-skill` to your Moqui runtime components.

### Step 2: Provide AI context

In your AI assistant or IDE tool, include or reference:

* `SKILL.md`
* `references/` directory (especially the file relevant to the task)

This works with:

* Chat-based assistants (paste or attach docs)
* IDE assistants (Cursor, Continue, Cody, Windsurf, etc.)
* RAG/MCP pipelines (index and retrieve from `SKILL.md` and `references/`)
* Local LLM workflows (Ollama/LM Studio + project docs)

### Step 3: Ask for Moqui artifacts

Example prompts:

* “Create an entity + service for order approval following Moqui 3.x patterns.”
* “Generate a form-list screen with filters and pagination for a custom entity.”
* “Add a `.rest.xml` resource for a service and include auth guidance.”
* “Write a Spock test for a service that creates an invoice.”

## Project structure

```text
moqui-ai-skill/
├── component.xml
├── SKILL.md
├── references/
│   ├── ENTITIES.md
│   ├── SERVICES.md
│   ├── SCREENS.md
│   ├── MANTLE.md
│   ├── MARBLE_ERP.md
│   └── TESTING.md
└── README.md
```

This component is **documentation-only**:

* No runtime dependencies
* No entities, services, or screens added to your application
* Safe to include in any Moqui project without affecting builds or deployments

## FAQ

### What problem does moqui-ai-skill solve?

It provides AI assistants with Moqui-specific conventions and schema-aligned guidance so they can generate valid Moqui XML and patterns without repeated trial and error.

### Does moqui-ai-skill add runtime overhead?

No. It contains Markdown docs and `component.xml` only. It does not change runtime behavior.

### Which Moqui versions are supported?

It targets **Moqui Framework 3.x** conventions and schema references (entity/service/screen XSD v3).

### Can I use it with tools other than Claude/OpenAI/Gemini?

Yes. It’s model-agnostic and works with any AI assistant that can read Markdown.

### Does it replace official Moqui documentation?

No. It complements the official docs and source code by providing an AI-optimized, structured reference.

## Contributing

Contributions are welcome. If you find a pattern where AI commonly generates incorrect Moqui code, please open an issue including:

1. The prompt you used
2. The incorrect output
3. The corrected expected output
4. Which reference file should be updated

Pull requests to improve accuracy, add missing patterns, or update schema references are appreciated.

## Related resources

* Moqui Framework: [https://www.moqui.org](https://www.moqui.org)
* Moqui docs: [https://www.moqui.org/docs](https://www.moqui.org/docs)
* moqui-framework (source): [https://github.com/moqui/moqui-framework](https://github.com/moqui/moqui-framework)
* mantle-udm: [https://github.com/moqui/mantle-udm](https://github.com/moqui/mantle-udm)
* mantle-usl: [https://github.com/moqui/mantle-usl](https://github.com/moqui/mantle-usl)
* SimpleScreens: [https://github.com/moqui/SimpleScreens](https://github.com/moqui/SimpleScreens)
* [Moqui Forum](https://forum.moqui.org)

## License

Released under **CC0 1.0 Universal** (public domain dedication).
[https://creativecommons.org/publicdomain/zero/1.0/](https://creativecommons.org/publicdomain/zero/1.0/)
