# Prompt Chaining: Advanced AI Prompting Technique

## What is Prompt Chaining?

**Prompt chaining** is a technique where you break down a complex task into smaller, sequential steps, with each step's output feeding into the next step as input. Instead of asking an AI to do everything in one large prompt, you create a series of connected prompts that build on each other.

### Simple Analogy
Think of it like an assembly line in a factory:
- **Single Prompt**: One worker does everything (design → build → test → package)
- **Prompt Chaining**: Each worker specializes in one step, passing the result to the next

---

## Why Use Prompt Chaining?

### Benefits

1. **Higher Quality Results**
   - Each step focuses on one specific task
   - AI can give full attention to each part
   - Reduces errors and hallucinations

2. **Better Control**
   - Validate output at each step
   - Adjust direction based on intermediate results
   - Easier to debug when something goes wrong

3. **Handles Complexity**
   - Break down overwhelming tasks
   - Manage token limits effectively
   - Maintain context across long workflows

4. **Reusability**
   - Create modular prompts
   - Reuse chains for similar tasks
   - Build prompt libraries

5. **Transparency**
   - See how the AI arrives at conclusions
   - Audit each step's reasoning
   - Understand the decision-making process

---

## When to Use Prompt Chaining

### ✅ Use Prompt Chaining When:

1. **Task is Multi-Step**
   - Requires distinct phases (analyze → design → implement → test)
   - Each phase has different requirements
   - Output from one phase informs the next

2. **Task is Complex**
   - Single prompt would be too long or confusing
   - Requires multiple types of expertise
   - Needs validation at intermediate steps

3. **Need for Quality Control**
   - Want to verify each step before proceeding
   - May need human review between steps
   - Error in one step shouldn't affect others

4. **Working with Large Codebases**
   - Need to analyze different parts separately
   - Refactoring requires careful planning
   - Multiple files need coordinated changes

5. **Research or Analysis Tasks**
   - Gather information → Analyze → Synthesize → Report
   - Each step requires different cognitive approach

### ❌ Don't Use Prompt Chaining When:

1. **Task is Simple**
   - Can be completed in one straightforward prompt
   - No intermediate validation needed
   - Example: "Format this JSON" or "Fix this typo"

2. **Steps are Tightly Coupled**
   - All information needed simultaneously
   - Breaking it up would lose context
   - Example: "Explain this specific code block"

3. **Time is Critical**
   - Need immediate answer
   - Adding complexity isn't worth the benefit

---

## How to Use Prompt Chaining

### Basic Structure

```
Step 1: [Specific Task]
   ↓ (Output becomes input)
Step 2: [Use Step 1 output for next task]
   ↓ (Output becomes input)
Step 3: [Use Step 2 output for final task]
   ↓
Final Result
```

### Types of Prompt Chains

#### 1. Sequential Chain (Linear)
Each step depends on the previous step's output.

```
Prompt 1: Analyze → Output A
Prompt 2: Use Output A to Design → Output B
Prompt 3: Use Output B to Implement → Output C
```

#### 2. Parallel Chain (Branching)
Multiple independent steps that later merge.

```
Prompt 1a: Analyze Frontend →┐
Prompt 1b: Analyze Backend  →├→ Prompt 2: Integrate Results
Prompt 1c: Analyze Database →┘
```

#### 3. Iterative Chain (Loop)
Repeat steps until a condition is met.

```
Prompt 1: Generate Code
Prompt 2: Test Code
If tests fail → Return to Prompt 1 with feedback
If tests pass → Continue to Prompt 3
```

#### 4. Conditional Chain (Decision Tree)
Next step depends on output evaluation.

```
Prompt 1: Analyze Problem
↓
If simple → Prompt 2a: Direct Solution
If complex → Prompt 2b: Break Down Further
```

---

## Practical Examples

### Example 1: Building a Complete Feature (Java Spring Boot)

#### ❌ Without Chaining (Poor Approach)
```
Create a complete User management feature with entity, repository, service, 
controller, DTOs, exception handling, validation, security, and tests.
```
*Problem*: Too much at once, likely to miss details or make mistakes.

#### ✅ With Chaining (Better Approach)

**Chain 1: Planning Phase**
```
Prompt 1: Analyze Requirements
"Analyze the requirements for a User management feature. 
What entities, relationships, and operations are needed?"

Output: List of entities, fields, relationships, operations
```

**Chain 2: Design Phase**
```
Prompt 2: Design Data Model
"Based on the analysis: [paste Prompt 1 output]
Design the User entity with JPA annotations, validation, and relationships."

Output: User entity design
```

**Chain 3: Implementation Phase**
```
Prompt 3: Create Repository
"For this User entity: [paste entity]
Use .github/prompts/create-repository.md to generate UserRepository
with methods for: findByUsername, findByEmail, search by keyword."

Output: UserRepository interface

Prompt 4: Create DTOs
"For this User entity: [paste entity]
Use .github/prompts/create-dto.md to create:
- UserRequestDto (for create/update)
- UserResponseDto (for API responses)
- UserSummaryDto (for list views)"

Output: DTO classes

Prompt 5: Create Service
"For this User entity and repository: [paste context]
Use .github/prompts/create-service.md to create UserService
with CRUD operations and business logic."

Output: UserService class

Prompt 6: Create Controller
"For this UserService: [paste service]
Use .github/prompts/create-controller.md to create REST endpoints
following RESTful conventions."

Output: UserController class
```

**Chain 4: Testing Phase**
```
Prompt 7: Generate Tests
"For this UserService: [paste service]
Generate unit tests using JUnit 5 and Mockito.
Test all CRUD operations and edge cases."

Output: UserServiceTest class
```

---

### Example 2: Code Refactoring

#### Chain Structure:

```
Prompt 1: Code Analysis
"Analyze this code for potential issues:
- Code smells
- Performance problems
- Security vulnerabilities
- Maintainability issues
[paste code]"

↓ Output: List of issues with severity

Prompt 2: Prioritization
"From this analysis: [paste issues]
Prioritize these issues by:
1. Security impact
2. Performance impact
3. Maintainability
Suggest which to fix first."

↓ Output: Prioritized list

Prompt 3: Solution Design
"For the top 3 priority issues: [paste items]
Design refactoring solutions that:
- Fix the issues
- Don't break existing functionality
- Follow Spring Boot best practices"

↓ Output: Refactoring plan

Prompt 4: Implementation
"Implement the refactoring for issue #1: [paste details]
Show the before and after code with explanations."

↓ Output: Refactored code

Prompt 5: Validation
"Review this refactored code: [paste code]
- Does it solve the original issue?
- Are there any new issues introduced?
- Suggest unit tests to verify the fix."

↓ Output: Validation results and test suggestions
```

---

### Example 3: Database Schema Design

```
Prompt 1: Requirements Extraction
"From this user story: [paste story]
Extract all entities, attributes, and relationships needed."

↓ Output: Entity list

Prompt 2: Normalization
"For these entities: [paste list]
Apply database normalization (3NF).
Identify primary keys, foreign keys, and indexes."

↓ Output: Normalized schema

Prompt 3: SQL Generation
"From this schema: [paste schema]
Generate:
1. Flyway migration scripts
2. JPA entity classes
3. Repository interfaces"

↓ Output: Migration scripts and Java code

Prompt 4: Validation
"Review this database design: [paste design]
Check for:
- Missing indexes
- N+1 query potential
- Performance bottlenecks
- Security concerns"

↓ Output: Issues and recommendations
```

---

### Example 4: API Design

```
Prompt 1: Domain Analysis
"For a User Management API, identify:
- Core resources
- Sub-resources
- Actions/operations
- Relationships"

↓

Prompt 2: Endpoint Design
"Design RESTful endpoints for: [paste resources]
Include: HTTP methods, URL patterns, status codes"

↓

Prompt 3: Request/Response Design
"For endpoint POST /api/v1/users: [paste details]
Design request/response DTOs with validation rules"

↓

Prompt 4: OpenAPI Specification
"Generate OpenAPI 3.0 spec for: [paste API design]
Include all endpoints, schemas, and examples"

↓

Prompt 5: Implementation
"Implement this API endpoint: [paste spec]
Using Spring Boot with proper validation and error handling"
```

---

## Best Practices for Prompt Chaining

### 1. Start with Planning
```
Always begin with an analysis or planning prompt:
"Break down this task into logical steps: [task description]"
```

### 2. Be Explicit About Context
```
Good: "Using the User entity from Step 2: [paste entity], create..."
Bad:  "Now create the service" (AI might forget context)
```

### 3. Validate Each Step
```
After each prompt, review the output:
- Is it correct?
- Does it meet requirements?
- Should I adjust the next prompt?
```

### 4. Save Intermediate Results
```
Keep outputs from each step in a document or notes.
You'll need to reference them in later prompts.
```

### 5. Use Prompt Templates
```
Create reusable templates for common chains.
Example: "Feature Development Chain Template"
```

### 6. Keep Steps Focused
```
Each prompt should have ONE clear objective.
Too many tasks = diluted quality.
```

### 7. Add Validation Prompts
```
After important steps, add a validation prompt:
"Review this code for: [specific concerns]"
```

### 8. Document Your Chain
```
Write down your chain structure for reuse:
- Step 1: Purpose, input, expected output
- Step 2: Purpose, input, expected output
...
```

---

## Prompt Chain Templates

### Template 1: Feature Development Chain
```
Step 1: Requirements Analysis
Step 2: Data Model Design
Step 3: API Design
Step 4: Implementation - Entity
Step 5: Implementation - Repository
Step 6: Implementation - Service
Step 7: Implementation - Controller
Step 8: Exception Handling
Step 9: Unit Tests
Step 10: Integration Tests
Step 11: Documentation
```

### Template 2: Bug Fix Chain
```
Step 1: Problem Reproduction
Step 2: Root Cause Analysis
Step 3: Solution Design
Step 4: Implementation
Step 5: Testing
Step 6: Regression Check
```

### Template 3: Code Review Chain
```
Step 1: Code Analysis
Step 2: Issue Identification
Step 3: Severity Assessment
Step 4: Recommendation Generation
Step 5: Refactoring Plan
```

### Template 4: Database Migration Chain
```
Step 1: Current Schema Analysis
Step 2: Target Schema Design
Step 3: Migration Strategy
Step 4: Data Transformation Logic
Step 5: Rollback Plan
Step 6: Migration Script Generation
Step 7: Testing Strategy
```

---

## Tools for Prompt Chaining

### 1. Manual Chaining (What we're doing)
- Copy output from one prompt
- Paste into next prompt with context
- **Best for**: Learning, complex tasks, customization

### 2. Copilot Chat with @workspace
```
Prompt 1: "@workspace Analyze the User entity"
Prompt 2: "@workspace Based on User entity, create UserService"
Prompt 3: "@workspace Based on UserService, create UserController"
```

### 3. Custom Scripts/Tools
- LangChain (Python framework for chains)
- Prompt flow tools
- Custom automation scripts

### 4. Copilot Instructions + Custom Prompts
```
Use .github/copilot-instructions.md (automatic context)
+ 
.github/prompts/[specific-task].md (task-specific guidance)
```

---

## Common Pitfalls

### 1. ❌ Losing Context Between Steps
**Problem**: Later prompts forget earlier context.
**Solution**: Explicitly include relevant output from previous steps.

### 2. ❌ Steps Too Large
**Problem**: Each step still tries to do too much.
**Solution**: Break down further until each step is simple.

### 3. ❌ No Validation
**Problem**: Errors in early steps propagate to later steps.
**Solution**: Add validation/review prompts between major steps.

### 4. ❌ Inflexible Chains
**Problem**: Chain doesn't adapt to unexpected results.
**Solution**: Add conditional logic or human review points.

### 5. ❌ Over-Engineering
**Problem**: Creating complex chains for simple tasks.
**Solution**: Use chaining only when it adds value.

---

## Measuring Success

### How to Know if Your Chain Works:

✅ **Quality Indicators**
- Each step produces clear, usable output
- Final result is better than single-prompt approach
- Fewer errors and hallucinations

✅ **Efficiency Indicators**
- Less time spent on revisions
- Can reuse chain for similar tasks
- Easier to debug problems

✅ **Process Indicators**
- Clear audit trail
- Can explain how result was achieved
- Others can follow and replicate

---

## Real-World Example: Complete User Feature

Let me show you a complete chain for creating a User management feature:

```
=== PROMPT CHAIN: User Management Feature ===

[STEP 1: REQUIREMENTS ANALYSIS]
Prompt: "Analyze requirements for a User Management API with:
- User registration and authentication
- CRUD operations
- Role-based access control
List all entities, fields, relationships, and operations needed."

Expected Output: 
- User entity (fields, validations)
- Role entity (if separate)
- Relationships
- Required operations

[STEP 2: ENTITY DESIGN]
Prompt: "@workspace Use .github/prompts/create-entity.md

Based on this analysis: [paste Step 1 output]
Create a User entity with:
- All fields from analysis
- Proper JPA annotations
- Audit timestamps
- Relationship with Role"

Expected Output: Complete User.java entity class

[STEP 3: REPOSITORY DESIGN]
Prompt: "@workspace Use .github/prompts/create-repository.md

For this User entity: [paste User entity]
Create UserRepository with methods:
- findByUsername
- findByEmail
- existsByUsername
- existsByEmail
- findByEnabledTrue with pagination"

Expected Output: UserRepository interface

[STEP 4: DTO CREATION]
Prompt: "@workspace Use .github/prompts/create-dto.md

For this User entity: [paste User entity]
Create:
1. UserRegistrationDto (username, email, password, firstName, lastName)
2. UserResponseDto (id, username, email, firstName, lastName, enabled, createdAt)
3. UserUpdateDto (email, firstName, lastName)

Include all validation annotations."

Expected Output: Three DTO record classes

[STEP 5: CUSTOM EXCEPTIONS]
Prompt: "Create custom exceptions:
- UserNotFoundException
- DuplicateUserException
- InvalidCredentialsException

Each should extend BusinessException with appropriate messages."

Expected Output: Exception classes

[STEP 6: SERVICE LAYER]
Prompt: "@workspace Use .github/prompts/create-service.md

Create UserService with these methods:
1. register(UserRegistrationDto) - Check for duplicates, hash password
2. findById(Long) - Throw exception if not found
3. findByUsername(String)
4. update(Long, UserUpdateDto) - Validate and update
5. delete(Long) - Soft delete by setting enabled=false
6. findAll(Pageable)

Use: UserRepository [paste repo]
DTOs: [paste DTOs]
Exceptions: [paste exceptions]"

Expected Output: Complete UserService class

[STEP 7: CONTROLLER LAYER]
Prompt: "@workspace Use .github/prompts/create-controller.md

Create UserController with endpoints:
- POST /api/v1/users/register - Register new user (public)
- GET /api/v1/users/{id} - Get user by ID (authenticated)
- GET /api/v1/users - Get all users with pagination (authenticated)
- PUT /api/v1/users/{id} - Update user (own profile or admin)
- DELETE /api/v1/users/{id} - Delete user (admin only)

Use UserService: [paste service]
Add appropriate security annotations."

Expected Output: Complete UserController class

[STEP 8: EXCEPTION HANDLER]
Prompt: "@workspace Use .github/prompts/create-exception-handler.md

Extend the global exception handler to handle:
- UserNotFoundException → 404
- DuplicateUserException → 409
- InvalidCredentialsException → 401

Use consistent error response format."

Expected Output: Exception handler additions

[STEP 9: UNIT TESTS]
Prompt: "Create unit tests for UserService:
- Test successful user registration
- Test duplicate username detection
- Test duplicate email detection
- Test user not found scenarios
- Test successful update
- Test successful soft delete

Use JUnit 5 and Mockito.
Mock: UserRepository, PasswordEncoder"

Expected Output: UserServiceTest class

[STEP 10: INTEGRATION TESTS]
Prompt: "Create integration tests for UserController:
- Test POST /api/v1/users/register (success and validation errors)
- Test GET /api/v1/users/{id} (success and not found)
- Test PUT /api/v1/users/{id} (success and not found)
- Test DELETE /api/v1/users/{id} (success and authorization)

Use @WebMvcTest, MockMvc, and mock UserService."

Expected Output: UserControllerTest class

[STEP 11: DOCUMENTATION]
Prompt: "Generate OpenAPI documentation for UserController:
- Add @Tag, @Operation, @ApiResponses annotations
- Include request/response examples
- Document all error responses"

Expected Output: Annotated controller with API docs

=== END OF CHAIN ===
```

---

## Quick Reference

### When to Chain:
- ✅ Multi-step tasks
- ✅ Complex problems
- ✅ Need quality control
- ✅ Building large features
- ❌ Simple one-off tasks
- ❌ Time-critical requests

### How to Chain:
1. Break task into clear steps
2. Each step has ONE goal
3. Pass output as input to next step
4. Validate between steps
5. Keep context explicit

### Chain Patterns:
- **Sequential**: A → B → C → D
- **Parallel**: A, B, C → D (merge)
- **Iterative**: A → B → (validate) → repeat or continue
- **Conditional**: A → (decide) → B or C

---

## Summary

**Prompt chaining** transforms complex, overwhelming tasks into manageable, sequential steps. Instead of asking AI to "do everything," you guide it through a structured process, validating and refining at each stage.

**Key Takeaway**: Think of yourself as a project manager directing the AI through phases of work, rather than dumping everything in one request.

Use chaining when quality, control, and complexity management matter more than speed. Skip it for simple, straightforward tasks.
