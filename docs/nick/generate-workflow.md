---
name: generate-workflow
description: Analyzes the codebase for a given feature using parallel explorer agents
  and generates a NEW workflow test document consumed by /test-flow
allowed-tools: Read, Bash, Grep, Glob, LS, Task, Write
argument-hint: [feature description or path hint]
---

You are generating a NEW workflow test document by deeply analyzing the codebase.
The feature to analyze: $ARGUMENTS

IMPORTANT: This creates a fresh workflow from scratch. If a workflow already exists
for this feature, warn the user and suggest using /update-workflow instead to
preserve any manual tweaks. Check workflows/ directory first.

## Phase 1: Check for existing workflow

Look in the workflows/ directory for a file that might already cover this feature.
If one exists, tell the user:
"Found existing workflow: workflows/[name].md
Use /update-workflow workflows/[name].md to update it while preserving your manual edits.
Or confirm you want to regenerate from scratch (this will overwrite)."

If the user already confirmed or no existing file was found, proceed.

## Phase 2: Fan out explorers

Spawn ALL 5 explorer sub-agents IN PARALLEL using the Task tool. Each one focuses
on a different aspect of the feature. Give them all the feature context: "$ARGUMENTS"

If the user gave a path hint, include it in every explorer prompt so they start there.
Otherwise, tell them to search src/app/ and src/components/ for the feature.

### Explorer 1: Routes & Pages
"Find all Next.js route segments (page.tsx, layout.tsx) related to: $ARGUMENTS
For each route:
- Read the full page.tsx
- List every component it imports and renders
- Note the URL path from the file structure
- Identify if it's a list page, detail page, or form page
- Check for dynamic route segments ([id], [...slug])
Output: list of routes with their purpose and top-level components."

### Explorer 2: Forms & Validation Schemas
"Find all form components and their validation schemas related to: $ARGUMENTS
- Grep for useForm, FormProvider, react-hook-form, <form>, zodResolver, z.object
- Read every form component fully
- Read every validation schema fully
- For EACH field in the schema extract: key, type, required/optional, ALL constraints
  (min, max, minLength, maxLength, email, regex, refine), defaults, transforms
- For EACH field in the JSX extract: UI label (from Label/FormLabel/placeholder),
  input component type (Input, Textarea, Select, Combobox, Command, DatePicker,
  Checkbox, Switch, RadioGroup, multi-select)
Output: complete table of every form field with schema key, UI label, type,
component, required, constraints, default value."

### Explorer 3: Dropdowns & Data Sources
"Find all dropdown, select, combobox, and searchable input components related to: $ARGUMENTS
- Grep for Combobox, Command, Select, Popover, cmdk, react-select, Listbox,
  options, items, onSearch, onInputChange
- For EACH dropdown found, Read the component and trace where its options come from:
  - Static: hardcoded array — list all options
  - API on mount: useQuery/useEffect fetch — note the endpoint URL
  - Async search: search param + debounce — note endpoint URL AND debounce ms
  - Dependent: value filtered by another field — note the dependency
- Note the interaction pattern: does it have a search input? is it a popover?
  does it support multi-select? how does selection display (text, chip, tag)?
Output: detailed list of every dropdown with data source, async behavior, and
interaction pattern."

### Explorer 4: API Endpoints & Server Actions
"Find all API routes and server actions related to: $ARGUMENTS
- Glob for src/app/api/**/route.ts and Grep for 'use server' in the feature area
- Read each relevant endpoint fully
- Document: HTTP method, URL path, request body shape (what fields),
  response shape (what it returns), status codes, error formats
- Check for side effects: does it create related records? trigger revalidation?
  send notifications? update caches?
- Also check for middleware, auth checks, rate limiting on these endpoints
Output: complete API map with request/response shapes and side effects."

### Explorer 5: Data Flow & UI Behavior
"Trace what happens after form submission related to: $ARGUMENTS
- Grep for: router.push, redirect, revalidatePath, revalidateTag, toast, sonner,
  mutate, invalidateQueries, setOpen, onClose, onSuccess, optimistic
- Document the full post-submit sequence: API call → toast → modal close → redirect
- Read each destination page (where user lands after submit)
- Map which form fields appear on which destination pages and in what format
  (raw '2500' might display as '2.500 kg')
- Also find: Suspense boundaries, loading/skeleton states, error boundaries,
  confirmation dialogs, debounce/throttle timers, toast durations, pagination type
Output: complete data flow chain and all UI timing/behavioral patterns."

Wait for ALL 5 explorers to return.

## Phase 3: Aggregate

Combine all 5 explorer outputs into a single structured analysis.

## Phase 4: Compose workflow

Pass the COMPLETE aggregated analysis to the workflow-composer agent with:
- The full combined analysis
- The original feature description: "$ARGUMENTS"
- Instruction: "Generate the complete workflow test document."

## Phase 5: Save and report

Derive a kebab-case filename from the feature.
Save to: workflows/[filename].md

Report:
1. Files analyzed
2. What was found: form fields count, dropdown count, API endpoints, pages involved
3. Saved location
4. Run command: /test-flow workflows/[filename].md
5. Remind: review the workflow, then future updates via /update-workflow