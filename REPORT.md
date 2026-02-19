
# Documentation–Implementation Mismatches and LLM / AI Risk

**Repository analyzed:** `libmodal/modal-js`

---
## Executive summary

- **12 findings are included below as verified documentation–implementation contradictions**
- The dominant patterns are:
  - **async vs sync contract drift** (missing `await` risk)
  - **return type drift** (`void` vs `Promise<T>`, `string` vs `Promise<object>`, “dictionary/object” vs `Promise<Secret[]>`)
  - **overly vague typing** (`unknown` documented vs concrete client instance)
- These issues may not break runtime behavior today, but they **materially increase LLM hallucination risk** by misrepresenting runtime contracts.

---
## How LLM and AI risk arises

LLMs rely heavily on comments, JSDoc, and local documentation as high-confidence signals.  
When documentation contradicts implementation, models must infer intent and guess behavior.

In practice:
- comments outweigh code during inference,
- contradictions force reconciliation,
- reconciliation optimizes for plausibility, not correctness.

This produces plausible-but-wrong code:
- missing `await`,
- invalid assumptions about returned fields,
- incorrect chaining on unresolved promises,
- incorrect usage patterns (e.g., treating objects as strings, treating factories as constructors).

Each failure increases retries, corrective turns, and token usage—directly increasing cost in AI-assisted workflows.

---
## Fully verified contradictions (high LLM / AI risk)

---

### 1. `ControlPlaneInvocation`

- **Location:** `src/invocation.ts:34`
- **Severity:** HIGH
- **Documented:** Returns `void`
- **Runtime:** Returns `Promise<unknown>` (static `async create`)
- **Risk:** Callers may treat invocation creation as synchronous and omit `await`, leading to unresolved promises and incorrect invocation sequencing.

---

### 2. `ControlPlaneInvocation`

- **Location:** `src/invocation.ts:34`
- **Severity:** MEDIUM
- **Documented:** Synchronous function
- **Runtime:** Asynchronous (`static async create`)
- **Risk:** Incorrect assumptions about execution order can introduce subtle race conditions, especially in chained invocation flows.

---

### 3. `InputPlaneInvocation`

- **Location:** `src/invocation.ts:131`
- **Severity:** HIGH
- **Documented:** Returns `void`
- **Runtime:** Returns `Promise<unknown>`
- **Risk:** Consumers may skip `await`, causing downstream logic to run before invocation setup completes.

---

### 4. `InputPlaneInvocation`

- **Location:** `src/invocation.ts:131`
- **Severity:** MEDIUM
- **Documented:** Synchronous function
- **Runtime:** Asynchronous
- **Risk:** Leads to incorrect mental models of control flow, particularly when composing multiple invocations.

---

### 5. `FunctionService.fromName`

- **Location:** `src/function.ts:44`
- **Severity:** HIGH
- **Documented:** Returns `Function`
- **Runtime:** Returns `Promise<Function_>`
- **Risk:** Callers may attempt to invoke methods on the returned value immediately, resulting in runtime errors from unresolved promises.

---

### 6. `Proxy.fromName`

- **Location:** `src/proxy.ts:47`
- **Severity:** HIGH
- **Documented:** Returns `string`
- **Runtime:** Returns `Promise<Proxy>`
- **Risk:** The comment misrepresents both return type and async behavior, leading to severe misuse (string operations on unresolved promises or proxy objects).

---

### 7. `Function_`

- **Location:** `src/function.ts:108`
- **Severity:** HIGH
- **Documented:** Returns `Function`
- **Runtime:** Returns `Promise<Function_>`
- **Risk:** AI-generated code is especially likely to omit `await`, producing high-confidence but broken usage and chaining.

---

### 8. `Cls.lookup`

- **Location:** `src/cls.ts:129`
- **Severity:** HIGH
- **Documented:** Returns `Cls`
- **Runtime:** Returns `Promise<unknown>` (static async lookup)
- **Risk:** Callers may treat the result as a ready-to-use class rather than a promise, breaking instantiation and method access patterns.

---

### 9. `Secret.fromName`

- **Location:** `src/secret.ts:134`
- **Severity:** HIGH
- **Documented:** Returns dictionary
- **Runtime:** Returns `Promise<Secret>`
- **Risk:** Consumers may misunderstand the abstraction and attempt dictionary-style access, or operate on unresolved promises.

---

### 10. `mergeEnvIntoSecrets`

- **Location:** `src/secret.ts:165`
- **Severity:** HIGH
- **Documented:** Returns object
- **Runtime:** Returns `Promise<Secret[]>`
- **Risk:** Incorrect assumptions about structure and timing can produce broken environment configuration logic (treating an array as an object and omitting `await`).

---

### 11. `buildSandboxCreateRequestProto`

- **Location:** `src/sandbox.ts:158`
- **Severity:** HIGH
- **Documented:** Returns object
- **Runtime:** Returns `Promise<SandboxCreateRequest>`
- **Risk:** Missing `await` can cause malformed or incomplete sandbox creation requests, especially when composed into higher-level create flows.

---

### 12. `getDefaultClient`

- **Location:** `src/client.ts:449`
- **Severity:** HIGH
- **Documented:** Returns `unknown`
- **Runtime:** Returns `ModalClient`
- **Risk:** Overly vague typing guidance increases guessing, encourages unnecessary defensive code, and reduces reliability of AI-assisted code generation around client usage.

---

## Research attribution

- Ji et al., 2023 — *A Survey of Hallucination in Large Language Models*  
  https://arxiv.org/abs/2311.05232

- Liu et al., 2024 — *CodeMirage: Hallucinations in Code Generation*  
  https://arxiv.org/abs/2408.08333
