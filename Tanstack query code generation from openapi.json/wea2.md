Here is the **Ultimate End-to-End Tutorial**.

This guide takes you from a **.NET 8** backend all the way to **Automated Frontend Tests**, connecting every piece (OpenAPI, React Query, Zod, MSW, and Vitest) automatically.

-----

### Step 1: The Backend Contract (.NET 8)

We define the API with strict attributes. This is the "Source of Truth".

**`Controllers/UsersController.cs`**

```csharp
using Microsoft.AspNetCore.Mvc;
using Swashbuckle.AspNetCore.Annotations;
using System.ComponentModel.DataAnnotations;

[ApiController]
[Route("api/[controller]")]
[SwaggerTag("Manages user accounts.")]
public class UsersController : ControllerBase
{
    /// <summary>
    /// Retrieves a list of users.
    /// </summary>
    [HttpGet(Name = "GetUsers")]
    [ProducesResponseType(typeof(List<UserDto>), StatusCodes.Status200OK)]
    public IActionResult GetAll() => Ok(new List<UserDto>());

    /// <summary>
    /// Creates a new user.
    /// </summary>
    [HttpPost(Name = "CreateUser")]
    [ProducesResponseType(typeof(UserDto), StatusCodes.Status201Created)]
    public IActionResult Create([FromBody] CreateUserRequest request)
    {
        return CreatedAtRoute("GetUsers", new UserDto());
    }
}

public class CreateUserRequest
{
    [Required]
    [StringLength(50, MinimumLength = 3)] // Becomes zod.min(3)
    public required string Username { get; set; }

    [Required]
    [EmailAddress] // Becomes zod.email()
    public required string Email { get; set; }

    public UserRole Role { get; set; } = UserRole.User;
}

[System.Text.Json.Serialization.JsonConverter(typeof(System.Text.Json.Serialization.JsonStringEnumConverter))]
public enum UserRole { Admin, User }
```

-----

### Step 2: The Orval Configuration

We configure Orval to generate **three** things:

1.  **Client:** React Query Hooks & Types.
2.  **Validation:** Zod Schemas.
3.  **Mocks:** MSW Handlers (Fake API).

**`orval.config.ts`**

```typescript
import { defineConfig } from 'orval';

export default defineConfig({
  api: {
    input: 'http://localhost:5000/swagger/v1/swagger.json',
    output: {
      target: './src/api/generated.ts',
      client: 'react-query',
      mock: true, // <--- Generates MSW Handlers
      override: {
        mutator: {
          path: './src/api/axios-instance.ts', // (See previous responses for this file)
          name: 'customInstance',
        },
      },
    },
  },
  zod: {
    input: 'http://localhost:5000/swagger/v1/swagger.json',
    output: {
      target: './src/api/generated-zod.ts',
      client: 'zod',
    },
  },
});
```

-----

### Step 3: Run the Generator

```bash
npx orval
```

**Output Files:**

  * `generated.ts`: The Hooks (`useGetUsers`) and Types.
  * `generated.msw.ts`: The Fake API (`getGetUsersMockHandler`).
  * `generated-zod.ts`: The Validation Logic (`createUserRequestSchema`).

-----

### Step 4: The Feature Component

We build the form using the generated Hooks and Zod schema.

**`src/components/CreateUserForm.tsx`**

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { useCreateUser } from '../api/generated';
import { createUserRequestSchema } from '../api/generated-zod';
import type { CreateUserRequest } from '../api/generated';

export const CreateUserForm = () => {
  const { mutate, isPending, isSuccess } = useCreateUser();

  const { register, handleSubmit, formState: { errors } } = useForm<CreateUserRequest>({
    resolver: zodResolver(createUserRequestSchema)
  });

  return (
    <form onSubmit={handleSubmit((data) => mutate({ data }))}>
      <input {...register('username')} placeholder="Username" />
      {errors.username && <span role="alert">{errors.username.message}</span>}

      <button disabled={isPending}>
        {isPending ? 'Saving...' : 'Create'}
      </button>

      {isSuccess && <div role="status">User created!</div>}
    </form>
  );
};
```

-----

### Step 5: Manual Testing (Browser Mocks)

Use this to develop without the backend running.

**`src/mocks/browser.ts`**

```typescript
import { setupWorker } from 'msw/browser';
import { getGetUsersMockHandler, getCreateUserMockHandler } from '../api/generated.msw';

// Orval generates these handlers for you!
export const worker = setupWorker(
  ...getGetUsersMockHandler(),
  ...getCreateUserMockHandler()
);
```

-----

### Step 6: Automated Testing (Vitest + MSW)

This is the final piece. We run a headless test that clicks the button and verifies the success message using the **generated MSW handlers**.

**1. Install Test Deps:**

```bash
npm install -D vitest jsdom @testing-library/react @testing-library/user-event msw
```

**2. Configure Vitest (`vitest.config.ts`):**

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom',
    setupFiles: './src/setupTests.ts',
  },
});
```

**3. Setup the Mock Server (`src/setupTests.ts`):**
We create a Node.js mock server that listens to all tests.

```typescript
import { beforeAll, afterEach, afterAll } from 'vitest';
import { cleanup } from '@testing-library/react';
import { setupServer } from 'msw/node';
import { getCreateUserMockHandler } from './api/generated.msw';

// 1. Setup server with Orval generated handlers
export const server = setupServer(...getCreateUserMockHandler());

// 2. Lifecycle hooks
beforeAll(() => server.listen());
afterEach(() => {
  server.resetHandlers();
  cleanup();
});
afterAll(() => server.close());
```

**4. The Test (`src/components/CreateUserForm.test.tsx`):**
Notice we don't mock Axios manually. MSW intercepts the network call.

```tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect } from 'vitest';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { CreateUserForm } from './CreateUserForm';

// Wrap component with QueryClient
const renderWithClient = (ui: React.ReactNode) => {
  const client = new QueryClient();
  return render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
};

describe('CreateUserForm', () => {
  it('submits the form and shows success message', async () => {
    renderWithClient(<CreateUserForm />);

    // 1. Type into inputs (Validation is active via Zod)
    await userEvent.type(screen.getByPlaceholderText(/username/i), 'NewUser123');

    // 2. Click Submit
    await userEvent.click(screen.getByRole('button', { name: /create/i }));

    // 3. Verify Success
    // MSW intercepted the POST request and returned 200 OK automatically
    await waitFor(() => {
      expect(screen.getByRole('status')).toHaveTextContent('User created!');
    });
  });
});
```

### Why this is the "Holy Grail" of DX:

1.  **Zero Flakiness:** The test doesn't rely on a real database.
2.  **Zero Boilerplate:** You didn't write `server.post('/api/users', ...)` manually. Orval generated it.
3.  **Full Confidence:** If you change the C\# `[Required]` attribute, the generated Zod schema changes. If your test tries to submit an empty form, **the test fails** because the form validation blocks the submit button click.

For a deeper dive into how MSW can simulate network conditions (like loading states and errors) which is crucial for the testing strategy above, check out this video.

[Mock Service Worker - Stop Mocking Fetch](https://www.google.com/search?q=https://www.youtube.com/watch%3Fv%3Dv77fjkKQyo4)
This video is relevant because it explains the philosophy behind using MSW instead of manually mocking `axios` or `fetch`, reinforcing why the integration in Step 6 is the robust industry standard.

Here is the comprehensive file structure for the **full-stack solution** we just built.

I have separated this into **Backend (API)** and **Frontend (Client)** so you can see exactly where every piece belongs.

### 📂 Project Root

```text
/MyProject
├── 📁 Api/                   # .NET 8 Backend
│   └── ...
└── 📁 Client/                # React Frontend
    └── ...
```

-----

### 1\. The Backend Structure (`/Api`)

This is your standard .NET 8 Web API. The critical parts for Orval are the Controller attributes and the Swagger configuration in `Program.cs`.

```text
/Api
├── 📁 Controllers
│   └── 📄 UsersController.cs       # 🧠 The Source of Truth (Attributes & XML Comments)
├── 📁 Properties
│   └── 📄 launchSettings.json      # Defines localhost:5000 / 7000
├── 📄 Api.csproj                   # ⚙️ <GenerateDocumentationFile> enabled here
├── 📄 Api.xml                      # 📄 Auto-generated XML comments (ignored in git)
├── 📄 Program.cs                   # ⚙️ Swagger & StringEnumConverter config
└── 📄 appsettings.json
```

-----

### 2\. The Frontend Structure (`/Client`)

This is where the magic happens. I have marked files as **[MANUAL]** (you write them once) or **[AUTO]** (Orval overwrites them).

```text
/Client
├── 📁 public
│   └── 📄 mockServiceWorker.js     # 🤖 MSW worker script (generated by `msw init`)
│
├── 📁 src
│   ├── 📁 api                      # ⚡ The Core Integration Layer
│   │   ├── 📄 axios-instance.ts    # [MANUAL] Your custom Axios config (BaseURL, Auth Interceptor)
│   │   ├── 📄 generated.ts         # [AUTO] React Query Hooks & TS Interfaces
│   │   ├── 📄 generated-zod.ts     # [AUTO] Zod Validation Schemas
│   │   └── 📄 generated.msw.ts     # [AUTO] MSW Mock Handlers & Faker Data
│   │
│   ├── 📁 components
│   │   ├── 📄 CreateUserForm.tsx   # ⚛️ Uses `useCreateUser` + `zodResolver`
│   │   └── 📄 CreateUserForm.test.tsx # 🧪 Uses Vitest + MSW to test the component
│   │
│   ├── 📁 mocks
│   │   └── 📄 browser.ts           # 🛠️ Setup file to start MSW in the browser
│   │
│   ├── 📄 App.tsx
│   ├── 📄 main.tsx                 # 🏁 Checks environment to start MSW before rendering
│   └── 📄 setupTests.ts            # 🧪 Configures MSW Server for Vitest (Node env)
│
├── 📄 orval.config.ts              # 🎛️ The Master Config for generation
├── 📄 package.json
├── 📄 tsconfig.json
├── 📄 vite.config.ts
└── 📄 vitest.config.ts             # 🧪 Test runner configuration
```

### Key File Relationships

1.  **`UsersController.cs`** ➔ feeds into ➔ **`orval.config.ts`** (via Swagger URL).
2.  **`orval.config.ts`** ➔ generates ➔ **`src/api/generated*.ts`** files.
3.  **`generated.ts`** (Hooks) ➔ used by ➔ **`CreateUserForm.tsx`**.
4.  **`generated-zod.ts`** (Schema) ➔ used by ➔ **`CreateUserForm.tsx`** (for validation).
5.  **`generated.msw.ts`** (Mocks) ➔ used by ➔ **`browser.ts`** (Browser Dev) AND **`setupTests.ts`** (Vitest).

### Summary of Commands

Once this structure is in place, your workflow is simple:

1.  **Backend Change:** Modify `UsersController.cs`.
2.  **Sync:** Run `npx orval`.
3.  **Frontend Update:** TypeScript errors will immediately tell you where to fix your code in `src/components`.

**Next Step:**
This is a solid architectural foundation. Would you like me to create a **`.gitignore`** snippet specifically for this setup? (You usually want to commit the generated files in frontend projects, but ignore the `.xml` in backend, and there's a debate about committing generated mocks).
