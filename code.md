# Code Writing Style

This document describes the coding conventions to follow when adding or changing code in Suchale. It is based on the existing React + TypeScript frontend and Express + TypeScript + Mongoose backend, while standardizing inconsistencies that already exist in the codebase.

## Core principles

- Prefer readable, explicit code over clever abstractions.
- Keep changes focused. Follow the existing architecture instead of introducing a new pattern for a small feature.
- Reuse existing utilities, types, validators, models, and service functions before creating new ones.
- Keep business rules easy to find and test.
- Handle expected failures deliberately; do not silently swallow errors.
- Avoid unrelated refactors when implementing a feature or fixing a bug.

## TypeScript

- Use TypeScript for application code and define types for function inputs, outputs, API payloads, and shared state where practical.
- Use `interface` for extendable object contracts and model shapes; use `type` for unions, aliases, and parameter objects.
- Prefer descriptive names in camelCase for variables and functions, PascalCase for React components, interfaces, and model constructors, and UPPER_SNAKE_CASE for true constants.
- Avoid `any`. Use a specific type, a generic, `unknown` with narrowing, or a small dedicated interface instead.
- Do not add type assertions merely to silence the compiler. Validate or narrow values at the boundary.
- Keep types close to the code that owns them; move them into shared types only when multiple modules genuinely need them.
- Use explicit return types for exported functions when they improve clarity or prevent unstable public contracts.

## Imports and formatting

- Group imports by purpose: external packages, internal aliases or modules, then relative imports. Separate groups with a blank line when it improves readability.
- Use the project's existing path aliases (such as `@/`) in the frontend where configured. Use valid relative imports in backend modules.
- Use double quotes in TypeScript/TSX, consistent with the dominant style in the codebase.
- Use two-space indentation.
- Keep one statement per line. Avoid compressed expressions and multiple statements on one line.
- Use trailing commas in multiline objects, arrays, and argument lists.
- Remove unused imports, dead code, and excessive blank lines.
- Follow the formatter and lint rules already configured in the relevant package rather than introducing a competing formatting convention.

## Frontend: React and TypeScript

### Components and state

- Use functional React components and hooks.
- Name components in PascalCase and keep files named consistently with the component or feature they contain.
- Keep component state local unless it must be shared. Use the existing contexts for genuinely shared concerns such as user, chat, theme, and socket state.
- Give state variables meaningful names and use the standard `[value, setValue]` convention.
- Define props with a dedicated type or interface when the contract is non-trivial.
- Extract repeated or complex behavior into a custom hook or focused component when doing so makes ownership clearer.
- Avoid turning a component into a general-purpose framework. Extract only reusable or independently understandable logic.
- Keep hooks unconditional and at the top level of a component. Do not return before hooks have run.
- For effects, specify dependencies correctly and clean up subscriptions, timers, event listeners, and sockets when the effect ends.

### Data fetching and UI states

- Use the shared Axios instance from `@/utils/axiosConfig` for API calls instead of creating ad hoc clients in individual components.
- Keep API base URLs and credentials configuration centralized.
- Handle loading, empty, success, and error states explicitly where relevant.
- Use `try/catch/finally` for async UI operations when needed so loading state is reset even if a request fails.
- Do not duplicate server data in multiple state variables without a clear reason.
- Keep JSX focused on rendering. Move complicated data transformations and event logic into named functions or hooks.
- Use semantic HTML and the existing Tailwind utility conventions. Reuse existing UI components and design tokens before adding one-off styles.
- Avoid unnecessary effects for values that can be derived during render.

### Example

```tsx
import { useEffect, useState } from "react";
import api from "@/utils/axiosConfig";
import type { Chat } from "@/types";

const ChatList = () => {
  const [chats, setChats] = useState<Chat[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const loadChats = async () => {
      setLoading(true);
      setError(null);

      try {
        const response = await api.get<Chat[]>("/user/userList");
        setChats(response.data);
      } catch {
        setError("Could not load chats.");
      } finally {
        setLoading(false);
      }
    };

    void loadChats();
  }, []);

  if (loading) return <p>Loading chats...</p>;
  if (error) return <p role="alert">{error}</p>;

  return (
    <ul>
      {chats.map((chat) => (
        <li key={chat.id}>{chat.name}</li>
      ))}
    </ul>
  );
};

export default ChatList;
```

Treat the example as a pattern, not as a claim about the exact response shape of every Suchale endpoint. Match the actual API contract and existing types.

## Backend: Express and TypeScript

### Module responsibilities

Keep responsibilities separated using the existing project structure:

- **Routers** declare HTTP methods, paths, middleware, and controller handlers. They should not contain business logic.
- **Controllers** handle request/response concerns: authentication context, input validation, status codes, and response shape.
- **Services** contain reusable business logic and coordinate model operations when a workflow is more than trivial.
- **Models** define Mongoose schemas, persistence constraints, and indexes.
- **Validators** validate untrusted request input at the API boundary.
- **Middleware** handles cross-cutting concerns such as authentication and authorization.

Do not move every simple database operation into a new service automatically. Use a service when it improves reuse, testability, or separation of responsibilities.

### Request handling

- Type Express request and response parameters.
- Treat request bodies, query parameters, route parameters, and external provider payloads as untrusted.
- Validate input with the project's existing Zod validators (for example, `safeParse`) before using it.
- Validate MongoDB ObjectIds before querying by ID when malformed IDs could otherwise cause errors.
- Check authentication and authorization before reading or mutating protected resources. Authentication alone does not prove the user may access a specific record.
- Return consistent HTTP status codes and response shapes. Use 400 for invalid input, 401 for unauthenticated requests, 403 for authenticated users lacking permission, 404 for missing resources, and 500 for unexpected server failures.
- Do not expose stack traces, secrets, tokens, raw database errors, or internal implementation details in client responses.
- Log useful server-side context without logging credentials or sensitive user data.
- Avoid returning from some branches and continuing execution in others; make each response path unambiguous.

### Async work and errors

- Use `async/await` for asynchronous flows.
- Catch errors at a boundary that can respond meaningfully. Avoid repetitive catch blocks that only hide the original problem.
- Use `finally` for cleanup that must happen regardless of success or failure.
- For intentional fire-and-forget work, make the intent explicit and ensure failures are handled by the called function or a dedicated error handler. Do not leave rejected promises unobserved.
- Avoid blocking the request on non-critical side effects when the product does not require them to finish first.
- Never use a success response to mask a failed critical operation.

### Example

```ts
import type { Request, Response } from "express";
import { createPostValidator } from "../validators/post-validator";
import { createPostService } from "../services/postService";

export const createPost = async (req: Request, res: Response) => {
  if (!req.id || !req.username) {
    return res.status(401).json({ message: "Unauthorized" });
  }

  const parsed = createPostValidator.safeParse(req.body);

  if (!parsed.success) {
    return res.status(400).json({
      message: parsed.error.issues[0]?.message ?? "Invalid request body",
    });
  }

  try {
    const post = await createPostService({
      userId: req.id,
      actorUsername: req.username,
      data: parsed.data,
    });

    return res.status(201).json({
      message: "Post created successfully",
      post,
    });
  } catch (error) {
    console.error("Failed to create post", error);
    return res.status(500).json({ message: "Internal server error" });
  }
};
```

## MongoDB and Mongoose

- Keep schemas and TypeScript model contracts aligned.
- Define required fields, trimming/normalization rules, defaults, and length constraints in the schema when they are persistence invariants.
- Add indexes to support actual query patterns. Avoid adding indexes speculatively; indexes add write and storage costs.
- Use `.select()` to fetch only needed fields and `.lean()` for read-only queries when a hydrated Mongoose document is not required.
- Populate only the relations needed for the response.
- Use atomic updates or conditional filters for operations that must not race, such as toggles, counters, and ownership checks.
- Avoid loading an entire collection when pagination or a bounded query is appropriate.
- Keep database access parameterized through Mongoose APIs; never build raw query expressions from unchecked user input.
- Normalize API response objects deliberately. Do not leak internal fields or rely on accidental differences between populated documents and plain objects.

## API and security

- Keep authentication middleware on protected routes.
- Enforce resource ownership and role checks in the backend, not only in the UI.
- Keep secrets in environment variables and validate required configuration at startup or before use.
- Use the shared API client on the frontend so credentials and base URL behavior stay consistent.
- Do not add tokens to local storage or expose secrets to frontend code. Only intentionally public configuration belongs in frontend environment variables.
- Validate upload metadata and constrain storage paths to the authenticated user where applicable.
- Add rate limits and appropriate security headers to sensitive endpoints.
- Keep response payloads minimal and stable; avoid returning full database documents by default.

## Naming and comments

- Choose names that explain intent: `createPostService`, `formatPostForResponse`, and `userChatList` are clearer than generic names such as `handleData` or `temp`.
- Use verbs for functions that perform actions and nouns for values and types.
- Comments should explain why a non-obvious decision exists, not narrate the next line of code.
- Preserve useful issue references or security rationale in comments.
- Remove stale comments, commented-out code, and decorative section banners when they add noise.
- Prefer small named helpers over deeply nested conditional expressions when they improve readability.

## Testing and verification

- Add or update tests for behavior changes, especially authentication, authorization, validation, and data mutation.
- Test failure paths as well as the happy path.
- Prefer tests that assert observable behavior and security boundaries instead of implementation details.
- Run the relevant package's tests, type checks, and lint commands before considering a change complete.
- Inspect the final diff for unrelated edits, duplicated logic, unsafe casts, missing cleanup, and inconsistent response contracts.
- Do not claim tests passed unless they were actually run.

## Patterns to avoid

- `any` spreading through service and response types.
- Controllers that contain large, reusable business workflows.
- Routes that mix path declarations with database logic.
- Missing `finally` blocks that can leave loading indicators stuck.
- Async side effects with unhandled promise rejections.
- Inconsistent quote style, indentation, spacing, and semicolon usage within the same file.
- Unvalidated request data or trusting client-supplied ownership IDs.
- Broad database reads when a bounded, indexed query is possible.
- Catch blocks that discard errors or return success after a critical failure.
- Introducing a new dependency or architectural layer without a concrete benefit.

## When changing existing code

1. Inspect the nearest implementation of the same feature and follow its established contracts.
2. Check the relevant types, validators, model, route, and frontend caller before changing an API.
3. Make the smallest complete change that solves the problem.
4. Preserve backward compatibility unless a breaking change is intentional and coordinated.
5. Add tests for the behavior and edge cases.
6. Run relevant checks and review the diff before finishing.

The goal is not to copy every inconsistency in the current repository. Preserve Suchale's straightforward, feature-oriented structure while making new code more consistent, typed, secure, and maintainable.
