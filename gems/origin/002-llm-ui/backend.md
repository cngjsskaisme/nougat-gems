# json-render-based Backend UI Generation Layer Design Document

> Reference: Catalog Prompt, Inline Mode, SpecStream, Mixed Stream Processing, Spec Validation models from the official json-render documentation  
> Scope: The layer where the LLM generates UI Specs constrained by the Catalog and delivers them to the Frontend

# 1. Design Purpose

The purpose of this design is to define a Backend UI generation layer that enables the LLM to generate safe and predictable UI Specs based on user requests and conversation Context.

The standard flow of the Backend UI generation layer is as follows.

```text
Catalog
→ Catalog Prompt
→ LLM
→ Inline Mixed Stream
→ Separate Text and SpecStream
→ Spec Compile
→ Catalog Validation
→ Return Text Part and Spec Part
```

The LLM does not generate HTML, Vue Components, or executable code.

The LLM generates json-render Specs using only the Components, Props, and Actions permitted by the Catalog.

# 2. Application Modes

# 2.1 Inline Mode

json-render's Inline Mode is used for conversational UI generation.

The Backend uses `catalog.prompt({ mode: "inline" })` to generate the LLM System Prompt.

In Inline Mode, the LLM can return the following.

- Plain text
- Plain text and JSONL Patch
- JSONL Patch-centric responses

The Backend distinguishes between text and JSONL Patches and converts them into order-preserving Message Parts.

# 2.2 Standalone Mode Excluded

Standalone Mode is suitable for generation tools where the entire result is a single UI Spec.

This design targets the Backend layer that generates UI within a conversation, so Standalone Mode is not included in the default scope.

# 3. Responsibilities of the Backend UI Generation Layer

The Backend UI generation layer is responsible for the following.

- Catalog management
- Catalog Prompt generation
- Combining conversation Context with UI generation rules
- LLM invocation
- Inline Mixed Stream processing
- Separating Text and SpecStream
- SpecStream Compile
- Final Spec structure validation
- Catalog-based Component·Props·Action validation
- UI generation policy validation
- Composing Text Part and Spec Part
- SpecStream error handling
- Observability related to UI generation

The Backend UI generation layer is not responsible for the following.

- Vue Component implementation
- Frontend Rendering
- Actual management of UI State
- User Event handling
- Business Workflow execution
- External system calls

# 4. Key Components

# 4.1 Catalog

The Catalog defines the vocabulary of UI that the AI can generate.

The Catalog includes the following items.

- Component definitions
- Component Props Schema
- Slot definitions
- Component Description
- Action definitions
- Action Parameter Schema
- Action Description
- Custom Function definitions

The Catalog simultaneously provides generation constraints and validation contracts.

# 4.2 Catalog Prompt Generator

The Catalog Prompt Generator transforms the Catalog into a System Prompt for the LLM.

In Inline Mode, the following is used.

```text
catalog.prompt({ mode: "inline" })
```

The generated Prompt includes the following.

- Available Components
- Component Props
- Slots
- Available Actions
- Action Parameters
- SpecStream writing format
- Inline Text writing rules

# 4.3 UI Generation Rules

Custom Rules suited to the UI generation purpose are added to the Catalog Prompt.

Custom Rules can define the following.

- Conditions under which UI should be generated
- Conditions under which only text should be returned
- Recommended Components
- Restricted Components
- Maximum UI complexity
- Layout composition principles
- Scope of information that can be included in the UI
- Action exposure conditions

Custom Rules are guidelines that steer the direction of UI generation.

They do not replace the Catalog and Validation.

# 4.4 LLM Gateway

The LLM Gateway separates the UI generation layer from the LLM Provider.

The Gateway receives the following common inputs.

- System Prompt
- Conversation
- Generation Mode
- Output limits
- Streaming flag

The Gateway returns a Text Stream.

The Text Stream in Inline Mode may contain both plain text and JSONL Patches.

# 4.5 Mixed Stream Processor

The Mixed Stream Processor distinguishes plain text from SpecStream Patches in Inline Mode output.

It uses the official json-render Utility.

The available official processing methods are as follows.

- `pipeJsonRender`
- `createJsonRenderTransform`
- `createMixedStreamParser`

The Mixed Stream Processor uses official Parsing rules rather than creating its own regex-based parser.

# 4.6 SpecStream Compiler

The SpecStream Compiler applies JSONL Patches in order to construct the current Spec.

The official processing methods are as follows.

- `compileSpecStream`
- `createSpecStreamCompiler`

The Compiler manages the following state.

- Current Spec
- Applied Patches
- Last Patch
- Compile errors
- Completion status

# 4.7 Spec Validator

The Spec Validator checks whether the compiled Spec conforms to the Schema and Catalog.

The official features are as follows.

- `validateSpec()`
- `catalog.validate()`
- `catalog.zodSchema()`
- `catalog.jsonSchema()`

The Spec Validator performs structural validation and Catalog validation separately.

# 4.8 Response Part Composer

The Response Part Composer composes Inline Mode results into Parts that the Frontend can render.

The logical Parts are as follows.

- Text Part
- Spec Part

The order of Parts preserves the LLM output order.

Only Specs that pass validation are included in the Spec Part.

# 5. Catalog Design

# 5.1 Catalog and Schema

The Schema defines the structure and grammar of the Spec.

The Catalog defines the list of Components, Actions, and Functions that can be used within that structure.

```text
Schema
= Grammar of the Spec

Catalog
= Vocabulary of generatable UI
```

The Backend defines the Catalog with a Schema compatible with the target Frontend Renderer.

# 5.2 Component Definition

A Catalog Component has the following information.

- Props Schema
- Optional Slot
- Description

The Props Schema constrains the format and allowed values of Properties that the AI can generate.

The Description helps the LLM judge the purpose of the Component and when it is appropriate to use it.

# 5.3 Action Definition

A Catalog Action has the following information.

- Action name
- Parameter Schema
- Description

The AI can only include defined Action names and Parameters in the Spec.

Action execution implementation is not included in the Catalog.

# 5.4 Function Definition

A Catalog Function declares named Functions that can be used in Validation or Transformation.

The LLM can only reference Functions registered in the Catalog.

The actual Function implementation is provided by the Runtime.

# 5.5 Catalog Scale

The Catalog should be limited to the Components and Actions necessary for UI generation.

An excessive Catalog creates the following problems.

- Increased Prompt size
- Ambiguity in Component selection
- Degraded generation result consistency
- Expanded validation scope
- Increased Spec complexity

The Catalog should reflect the product's UI Design System and generation purpose.

# 6. Prompt Composition

The System Prompt for UI generation is composed in the following order.

```text
json-render Inline Mode guidelines
→ Catalog Component definitions
→ Catalog Action definitions
→ Catalog Function definitions
→ UI generation Custom Rules
→ Conversation Context
```

The Prompt clearly separates user requests from Catalog rules.

The Catalog Prompt is used as the generation baseline so that the LLM does not generate Components or Actions outside the Catalog.

# 7. Inline Mixed Stream Processing

# 7.1 Text Line

Content that is not interpreted as a JSON Patch is passed to the Text Part.

# 7.2 Patch Line

Valid JSONL Patch Lines are passed to the SpecStream.

# 7.3 Order Preservation

Since text and Patches may be output alternately, the Mixed Stream Processor preserves Part boundaries.

For example, if generated in the order text, Spec, text, the Response Part maintains the same order.

# 7.4 Incomplete Chunks

In Streaming Transport, a single JSONL Line may be split across multiple Chunks.

The Parser determines Patches based on completed Lines, not Chunks.

# 8. SpecStream Design

# 8.1 Patch Format

SpecStream uses RFC 6902 JSON Patch Operations.

The supported Operations are as follows.

- add
- remove
- replace
- move
- copy

# 8.2 Patch Application Order

Patches are applied in the order output by the LLM.

The order is not changed, and parallel application is not performed.

# 8.3 Incremental Spec

The current Spec can be updated each time a Patch is applied.

The incremental Spec can be passed to Frontend Streaming Rendering or used for internal final Spec generation in the Backend.

# 8.4 Final Spec

When the Stream completes successfully, the last current Spec is confirmed as the final Spec.

If incomplete Lines or failed Patches remain, it is not treated as a final success.

# 9. Spec Validation

# 9.1 Structural Validation

Structural validation checks whether the Spec follows the grammar of the selected Schema.

Validation items:

- Spec Object format
- Root existence
- Elements structure
- Element Type
- Props structure
- Children references
- Event Binding structure
- Dynamic Value format
- State format

# 9.2 Catalog Validation

Catalog validation checks whether the Spec uses only the permitted UI vocabulary.

Validation items:

- Component registration status
- Component Props Schema
- Slot usage
- Action registration status
- Action Parameter Schema
- Function registration status

# 9.3 UI Generation Policy Validation

UI generation policy validation checks generation quality rules that are difficult to express with the Catalog Schema.

Validation items:

- Maximum number of Elements
- Maximum nesting depth
- Maximum repeat count
- Allowed Layout scope
- Excessive repetition of the same Component
- Unnecessary UI generation
- Excessive Spec generation for requests where Text Only is appropriate
- Information that cannot be included in the UI

# 9.4 Validation Failure

Specs that fail validation are not passed to the Frontend.

The handling method is one of the following.

- Return only valid Text Parts
- Return a UI generation failure Part
- Attempt limited regeneration
- Treat the response as failed

The original invalid Spec is not used as a rendering target.

# 10. Response Design

# 10.1 Streaming Response

In Streaming Response, Text Parts and Spec Patch Parts are delivered sequentially.

The Frontend Compiles Spec Patches and renders progressively.

# 10.2 Final Spec Response

In Non-streaming Response, the Backend Compiles and validates all SpecStreams and then returns the final Spec Part.

# 10.3 Response Part Principles

- Text and Spec are expressed as separate Parts.
- Part order is preserved.
- Only validated Specs are included in the Spec Part.
- Text Only responses are allowed.
- UI Only responses are allowed.
- Responses containing both Text and UI are allowed.

# 11. Error Handling

The error types of the UI generation layer are as follows.

- Catalog Prompt generation errors
- LLM generation errors
- Mixed Stream Parse errors
- JSONL Line errors
- Patch application errors
- Spec Compile errors
- Structural validation errors
- Catalog validation errors
- UI generation policy errors

When an error occurs, the full server internal information is not returned; only the success status of the UI generation result and a limited error classification are provided.

# 12. Version Management

The Backend UI generation layer manages the following versions.

- Catalog Version
- Schema Version
- SpecStream Format Version
- Response Part Contract Version
- UI Generation Rule Version

Only Specs matching the Catalog and Schema Versions supported by the Frontend are delivered.

When the Catalog changes, the impact on Prompt, Validation, and Frontend Registry is reviewed together.

# 13. Observability

The UI generation layer records the following information.

- Generation Mode
- Catalog Version
- Schema Version
- LLM Provider and Model
- Text Part count
- Patch count
- Final Element count
- Spec Compile result
- Catalog Validation result
- UI generation policy result
- Generation time
- Streaming completion or interruption
- Error type

Observability information is used to evaluate the quality and stability of the UI generation process.

# 14. Performance Principles

- Limit the Catalog Prompt size.
- Limit the conversation Context size.
- Limit the maximum number of Patches.
- Limit the maximum Spec size.
- Limit the maximum number of Elements.
- Limit the maximum State size.
- Do not unnecessarily convert Text Only responses into Specs.
- Do not perform unlimited regeneration on Validation failure.
- Manage the validation cost of incomplete Specs during Streaming.

# 15. Test Strategy

# 15.1 Catalog Test

- Catalog Prompt generation
- Component Props Schema
- Action Parameter Schema
- Function definitions
- Catalog Version

# 15.2 Generation Test

- Text Only
- Spec Only
- Text and Spec mixed
- Multiple Text Blocks and Spec Parts
- Catalog Component selection accuracy
- Action generation accuracy

# 15.3 Streaming Test

- Normal JSONL Patches
- Patches split across multiple Chunks
- Incomplete Lines
- Invalid Patch Operations
- Patch order errors
- Stream interruption
- Final Spec confirmation

# 15.4 Validation Test

- Normal Spec
- Unregistered Component
- Invalid Props
- Unregistered Action
- Invalid Action Parameters
- Unregistered Function
- Invalid Children references
- UI complexity limit exceeded

# 16. Completion Criteria

The Backend UI generation layer must satisfy the following conditions.

- Generate LLM Prompts based on `catalog.prompt({ mode: "inline" })`.
- Separate Text and JSONL Patches from Inline Output using official Utilities.
- Compile SpecStream into a final or incremental Spec.
- Structurally validate the final Spec.
- Validate the final Spec with `catalog.validate()`.
- Remove or reject Components, Actions, and Functions outside the Catalog.
- Deliver only validated Specs to the Frontend.
- Support Text Only, Spec Only, and Text and Spec mixed responses.
- Process Streaming and Final Spec Response with the same Spec model.
- Maintain compatibility between Frontend Registry and Catalog Version.

# 17. Official Documentation Reference

- Introduction: https://json-render.dev/docs
- Catalog: https://json-render.dev/docs/catalog
- Generation Modes: https://json-render.dev/docs/generation-modes
- Specs: https://json-render.dev/docs/specs
- Streaming: https://json-render.dev/docs/streaming
- Registry: https://json-render.dev/docs/registry
- Validation: https://json-render.dev/docs/validation
- Core API: https://json-render.dev/docs/api/core
- AI SDK Integration: https://json-render.dev/docs/ai-sdk
