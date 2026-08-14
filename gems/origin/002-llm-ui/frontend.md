# json-render-based Frontend UI Generation Layer Design Document

> Reference: Catalog, Vue Renderer, Inline Mode, SpecStream, State, Action, Visibility, Validation models from the official json-render documentation  
> Scope: The layer that composes and renders AI-generated UI Specs into Vue UI

# 1. Design Purpose

The purpose of this design is to define a Frontend UI generation layer that renders AI-generated UI in a safe and consistent manner within a Vue application.

The existing response type-based branching structure is replaced with the following official json-render flow.

```text
AI-generated Spec
→ Check compatibility with Catalog
→ Vue Registry lookup
→ Interpret state and actions via Providers
→ Renderer
→ Vue Component Tree
```

The AI does not generate Vue code or HTML. The AI generates JSON Specs using only the Components, Props, and Actions defined in the Catalog.

# 2. Application Modes

# 2.1 Inline Mode

json-render's Inline Mode is used for conversational UI.

In Inline Mode, a single AI response can take the following forms.

- Text only
- UI Spec only
- Text and UI Spec together

The Frontend distinguishes between Text Parts and Spec Parts within a Message and renders them.

# 2.2 Standalone Mode Excluded

Standalone Mode is suitable for screen generators, dashboard generators, form builders, and other tools where the entire result is a single generative UI.

This design is a structure that inserts generative UI into a conversation, so Standalone Mode is not included in the default scope.

# 3. Responsibilities of the Frontend UI Generation Layer

The Frontend UI generation layer is responsible for the following.

- Distinguishing received Message Parts
- Extracting Spec Parts
- Checking the basic structure of the Spec
- Connecting Catalog and Vue Registry
- Rendering the Vue Component Tree
- Providing Generated UI State
- Interpreting State Binding
- Evaluating Visibility conditions
- Executing Validation
- Connecting Action Events with Handlers
- Incremental application of SpecStream
- Fallback handling for unknown Components

The Frontend UI generation layer is not responsible for the following.

- LLM invocation
- Catalog Prompt generation
- AI output generation
- Business Workflow determination
- External system execution
- Server result determination

# 4. Key Components

# 4.1 Message Part Resolver

The Message Part Resolver distinguishes text from UI Specs in Inline Mode responses.

The logical Message Parts are as follows.

- Text Part
- Spec Part

The Text Part is passed to the existing text or Markdown Renderer.

The Spec Part is passed to the json-render Vue Renderer.

The Message Part Resolver must preserve the order of Parts. Even when text and UI are generated alternately, they are displayed in the original generation order.

# 4.2 Spec Boundary Validator

The Spec Boundary Validator checks whether the Spec is in a processable format before entering the Renderer.

The validation scope is as follows.

- Spec existence
- Spec Object format
- Root identifier existence
- Elements structure existence
- State format
- Element reference format
- Catalog Version compatibility
- Maximum Element count
- Maximum State size

Even if detailed per-Component validation of the Catalog is performed on the Backend, basic validation is performed again at the Frontend boundary.

# 4.3 Vue Registry

The Vue Registry connects Component names defined in the Catalog to actual Vue Component implementations.

It uses the official `defineRegistry` from `@json-render/vue`.

The Registry provides the following.

- Catalog-based type safety
- Component implementation linking
- Action Handler linking
- Component Event delivery
- Props delivery
- Children delivery
- Loading state delivery
- Binding path delivery

The Registry is the single entry point for Component selection.

Strings included in the Spec are not directly interpreted as Vue Component names or Import paths.

# 4.4 Renderer

The official `Renderer` takes a validated Spec and Registry as input and generates a Vue Component Tree.

The Renderer operates in the following order.

```text
Check Spec Root
→ Look up Elements
→ Look up Component Type
→ Select Registry Component
→ Resolve Props Dynamic Values
→ Recursively render Children
→ Connect Event Bindings
```

When the Renderer encounters an unregistered Component, it does not make arbitrary guesses and uses a Fallback Component.

# 5. Provider Design

# 5.1 StateProvider

The StateProvider provides the State Model used by the Generated UI.

The StateProvider supports the following two operating modes.

## Uncontrolled Mode

The internal Store of json-render manages the UI State.

Suitable states:

- Internal input values within the UI
- Transient selection states
- Toggle states
- Conditional display values
- Data used only within the generated UI

## Controlled Mode

An external StateStore is injected into the StateProvider.

Suitable situations:

- When the state of the generative UI needs to be shared with the application Store
- When the UI state needs to be referenced from other screens
- When the same Spec needs to be reused or restored

The basic principle of the UI generation layer is to keep the owner of the same state as a single entity.

# 5.2 VisibilityProvider

The VisibilityProvider determines whether to display an Element based on conditions declared in the Spec.

Visibility conditions reference State paths.

Main roles:

- Conditional Component display
- Additional UI display based on input values
- Section changes based on selection state
- Internal Loading or Error representation within the UI

Visibility is responsible only for screen representation. It does not determine business permissions or server execution feasibility.

# 5.3 ValidationProvider

The ValidationProvider is responsible for input validation of the Generated UI.

It can use the official Built-in Validator and Custom Validation Functions registered in the Catalog.

Validation scope:

- Required values
- Email format
- Minimum and maximum length
- Numeric range
- Regular expressions
- Comparison with other Fields
- Conditional required values

The validation timing is configured as one of the following.

- change
- blur
- submit

Validation results are reflected in the Generated UI State and Component representation.

# 5.4 ActionProvider

The ActionProvider connects Action names declared in the Spec to Frontend Handlers.

Roles of the ActionProvider:

- Action name lookup
- Parameter delivery
- Asynchronous Action execution
- Providing Loading state during Action execution
- Linking follow-up Actions on Success or Error
- Navigation Handler linking

The AI only declares Action names and Parameters. The actual Handler implementation is provided by the Frontend.

# 6. Component Design Principles

# 6.1 Catalog-based Components

All generatable Components must first be defined in the Catalog.

Each Component definition has the following information.

- Component name
- Props Schema
- Optional Slot
- Description for the AI to understand the Component's purpose

The Frontend registers Vue Component implementations that match the Catalog definitions in the Registry.

# 6.2 Component Responsibilities

Registry Components are responsible only for the following.

- Props representation
- Children representation
- Emitting user Events
- Displaying and changing bound values
- Loading state display
- Validation result display

Registry Components do not select other Components or directly modify the Spec.

# 6.3 Container Components

Components that contain Children declare Slots in the Catalog.

The Renderer transforms Children references in the Spec into actual Vue Child Trees and passes them.

# 7. Dynamic Value and Data Binding

The Dynamic Value model of json-render is used to connect Props and State.

The supported concepts are as follows.

- `$state`: State value lookup
- `$bindState`: Two-way State value Binding
- `$item`: Current Item lookup in repeat scope
- `$bindItem`: Repeat Item value Binding
- `$index`: Repeat Index lookup
- `$cond`: Value selection based on conditions
- `$template`: String generation including State values
- `$computed`: Computation based on Catalog Functions

The Frontend does not evaluate Dynamic Values as arbitrary expressions.

Only the Directives and Resolvers defined by the json-render Runtime are used.

# 8. Spec Structure

The UI Spec is composed according to the selected Schema.

The basic Flat Tree structure used by the Vue Renderer has the following concepts.

- root
- elements
- state

# 8.1 root

The identifier of the top-level Element from which rendering begins.

# 8.2 elements

An Element Map keyed by Element ID.

Each Element can include the following information.

- type
- props
- children
- on
- visible
- repeat

# 8.3 state

The initial State Model used by the Generated UI.

# 9. Action Binding

An Element's Event can be linked to one or more Actions.

The processing flow is as follows.

```text
User Event
→ Registry Component emit
→ Look up Element Event Binding
→ ActionProvider
→ Execute Action Handler
→ Reflect in UI State or rendering result
```

When multiple Actions are linked, the order declared in the Spec is preserved.

Built-in Actions for State changes and Custom Actions defined by the application are distinguished.

# 10. Streaming Design

# 10.1 SpecStream

json-render uses a JSONL-based SpecStream.

Each Line is an RFC 6902 JSON Patch Operation.

Patches are applied sequentially, and the current Spec is progressively completed.

# 10.2 Vue Streaming Runtime

The Frontend uses the Streaming Composable from `@json-render/vue` or the SpecStream Compiler from `@json-render/core`.

The Streaming state includes the following.

- Current Spec
- Streaming in progress
- Last Patch
- Errors
- Abort state
- Completion state

# 10.3 Incremental Rendering

After applying a Patch, the Renderer is updated within the valid range of the current Spec.

It must be able to handle temporary incomplete states caused by Elements that have not yet been generated.

When Streaming completes, the final Spec is confirmed.

# 11. Fallback Handling

Fallback UI is used in the following cases.

- Spec format errors
- Missing Root
- Unregistered Component
- Invalid Element references
- Dynamic Value resolution failure
- Missing Action Handler
- Streaming Patch application failure
- Renderer errors

Fallback is displayed only within the scope of the relevant Spec Part without interrupting the entire Chat UI.

# 12. Version Management

The Frontend UI generation layer checks the following versions.

- Catalog Version
- Spec Schema Version
- Message Part Contract Version
- Registry Version

Specs with unsupported Versions are not rendered.

When the Catalog changes, the completeness of Registry Components and Action Handlers is verified together.

# 13. Observability

The UI generation layer records the following events.

- Spec received
- Spec Validation success·failure
- Registry Component lookup failure
- Renderer start·completion·failure
- Action execution start·completion·failure
- Streaming start·completion·interruption
- Fallback occurrence
- Version mismatch

Observability data is recorded focusing on Spec structure and processing results, and the original user input is excluded from the default recording target.

# 14. Performance Principles

- Do not create a Renderer for Text Only responses.
- Manage Specs at the Message level.
- Do not unnecessarily re-Compile the same Spec.
- Update focusing on changed Elements when applying Patches.
- Limit the maximum number of Elements and repeat count.
- Registry Components use stable Keys.
- Limit the Catalog scale to suit the UI purpose.

# 15. Accessibility Principles

Accessibility is ensured by the Registry Component, not the AI.

Registry Components must provide the following by default.

- Meaningful Labels
- Keyboard operation
- Focus handling
- Form Error association
- Loading state notification
- Dynamic Content change announcement
- Appropriate Semantic Elements

# 16. Completion Criteria

The Frontend UI generation layer must satisfy the following conditions.

- Be able to distinguish Text Parts and Spec Parts in Inline Mode.
- Use the Registry and Renderer from `@json-render/vue`.
- Configure StateProvider, VisibilityProvider, ValidationProvider, and ActionProvider according to their purposes.
- Do not render Components not in the Catalog.
- Do not execute Actions not in the Catalog.
- Interpret Dynamic Values and Data Binding with the json-render Runtime.
- Be able to compose SpecStream into an incremental Spec.
- Handle Rendering errors with Spec Part-level Fallback.
- Check compatibility between Catalog and Registry Versions.

# 17. Official Documentation Reference

- Introduction: https://json-render.dev/docs
- Catalog: https://json-render.dev/docs/catalog
- Generation Modes: https://json-render.dev/docs/generation-modes
- Specs: https://json-render.dev/docs/specs
- Streaming: https://json-render.dev/docs/streaming
- Registry: https://json-render.dev/docs/registry
- Validation: https://json-render.dev/docs/validation
- Vue API: https://json-render.dev/docs/api/vue
