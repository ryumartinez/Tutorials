Here is the complete **End-to-End Tutorial: From .NET 8 to React Query with Orval**.

This guide ensures strict typing, clean hook names, and fully documented JSDoc tooltips.

-----

### Step 1: Configuring the .NET 8 Backend

We need to configure .NET to generate a descriptive and strict `openapi.json` file.

**1. Install the Annotations Package:**

```bash
dotnet add package Swashbuckle.AspNetCore.Annotations
```

**2. Update `Project.csproj`:**
Enable XML documentation so your C\# comments become JSDoc.

```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn> </PropertyGroup>
```

**3. Update `Program.cs`:**
Configure Swagger to use annotations, XML comments, and String Enums.

```csharp
builder.Services.AddControllers()
    .AddJsonOptions(options => {
        // Force Enums to be strings ("Admin") instead of ints (1)
        options.JsonSerializerOptions.Converters.Add(new System.Text.Json.Serialization.JsonStringEnumConverter());
    });

builder.Services.AddSwaggerGen(c =>
{
    c.EnableAnnotations(); // Enables [SwaggerTag]

    // Link the XML file for JSDoc generation
    var xmlFile = $"{System.Reflection.Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    c.IncludeXmlComments(xmlPath, includeControllerXmlComments: true);
});
```

-----

### Step 2: Defining the Controller

Create a controller that acts as a strict contract. We use specific attributes to ensure Orval generates clean code (e.g., `useGetUsers` instead of `useApiUsersGet`).

**`Controllers/UsersController.cs`**

```csharp
using Microsoft.AspNetCore.Mvc;
using Swashbuckle.AspNetCore.Annotations;
using System.ComponentModel.DataAnnotations;

[ApiController]
[Route("api/[controller]")]
[SwaggerTag("Manages user accounts.")] // Adds description to the service
public class UsersController : ControllerBase
{
    /// <summary>
    /// Retrieves a list of users.
    /// </summary>
    [HttpGet(Name = "GetUsers")] // Generated Hook: useGetUsers
    [ProducesResponseType(typeof(List<UserDto>), StatusCodes.Status200OK)]
    public IActionResult GetAll()
    {
        return Ok(new List<UserDto>());
    }

    /// <summary>
    /// Creates a new user.
    /// </summary>
    [HttpPost(Name = "CreateUser")] // Generated Hook: useCreateUser
    [ProducesResponseType(typeof(UserDto), StatusCodes.Status201Created)]
    public IActionResult Create([FromBody] CreateUserRequest request)
    {
        return CreatedAtRoute("GetUsers", new UserDto());
    }
}

// DTOs
public class UserDto {
    public int Id { get; set; }
    public required string Username { get; set; }
    public UserRole Role { get; set; }
}

public class CreateUserRequest {
    [Required] public required string Username { get; set; }
    public UserRole Role { get; set; }
}

public enum UserRole { Admin, User }
```

-----

### Step 3: Configuring the React App

Now we set up the frontend tooling to ingest that API.

**1. Install Dependencies:**

```bash
npm install axios @tanstack/react-query
npm install -D orval
```

**2. Create Custom Axios Instance (`src/api/axios-instance.ts`):**
This handles your base URL and authentication headers.

```typescript
import axios, { AxiosRequestConfig } from 'axios';

export const AXIOS_INSTANCE = axios.create({
  baseURL: 'https://localhost:7000' // Your .NET URL
});

// Implementation required by Orval
export const customInstance = <T>(config: AxiosRequestConfig, options?: AxiosRequestConfig): Promise<T> => {
  const source = axios.CancelToken.source();
  const promise = AXIOS_INSTANCE({ ...config, ...options, cancelToken: source.token }).then(({ data }) => data);
  // @ts-ignore
  promise.cancel = () => source.cancel('Query cancelled');
  return promise;
};
```

**3. Create Orval Config (`orval.config.ts`):**

```typescript
import { defineConfig } from 'orval';

export default defineConfig({
  api: {
    input: 'https://localhost:7000/swagger/v1/swagger.json', // Direct link to .NET Swagger
    output: {
      target: './src/api/generated.ts',
      client: 'react-query',
      override: {
        mutator: {
          path: './src/api/axios-instance.ts',
          name: 'customInstance',
        },
      },
    },
  },
});
```

-----

### Step 4: Generating the Code

Run your .NET backend so the Swagger URL is live, then run the generator.

```bash
# In your React project terminal
npx orval
```

*Result:* A file at `src/api/generated.ts` containing strict TypeScript interfaces and React Query hooks (`useGetUsers`, `useCreateUser`).

Based on the `UsersController` we defined earlier, here is an exact simulation of the code Orval will generate in your `src/api/generated.ts` file.

I have broken this down into three parts: **Types**, **Query Helpers**, and **Hooks**.

### 1\. The TypeScript Interfaces (DTOs)

Orval converts your C\# classes (`UserDto`, `CreateUserRequest`) and Enums into strict TypeScript types.

```typescript
/**
 * Generated by orval v6.28.2 🍺
 * Do not edit manually.
 * My API
 * OpenAPI spec version: 1.0
 */
import { useQuery, useMutation } from '@tanstack/react-query';
import type {
  UseQueryOptions,
  UseMutationOptions,
  QueryFunction,
  MutationFunction,
  UseQueryResult,
  QueryKey
} from '@tanstack/react-query';
import { customInstance } from './axios-instance'; // Your custom axios instance

// --- ENUMS (Strictly typed as strings) ---
// Orval generates a 'const' object so you can use UserRole.Admin in code
export const UserRole = {
  Admin: 'Admin',
  User: 'User',
} as const;

// It also generates a Type so you can use it in interfaces
export type UserRole = typeof UserRole[keyof typeof UserRole];

// --- DTOs (Mapped from C# classes) ---
export interface UserDto {
  id: number;
  username: string;
  role: UserRole;
}

export interface CreateUserRequest {
  username: string;
  role?: UserRole; // Optional because we had a default value in C#
}
```

### 2\. The Query Key Factories

Orval generates these helper functions so you never have to hardcode a cache key string like `['users', page]`.

```typescript
// Helper to generate the exact cache key for "GetUsers"
export const getGetUsersQueryKey = () => {
  return ['/api/Users'] as const;
};
```

### 3\. The React Query Hooks

These are the actual functions you use in your components.

**The Query Hook (GET):**
Notice how it includes the JSDoc from your C\# XML comments.

```typescript
/**
 * Retrieves a list of users.
 * @summary Retrieves a list of users.
 */
export const useGetUsers = <TData = UserDto[], TError = unknown>(
  options?: {
    query?: UseQueryOptions<UserDto[], TError, TData>,
  }
): UseQueryResult<TData, TError> & { queryKey: QueryKey } => {

  const { query: queryOptions } = options ?? {};
  const queryKey = queryOptions?.queryKey ?? getGetUsersQueryKey();

  const queryFn: QueryFunction<UserDto[]> = ({ signal }) =>
    customInstance({ url: `/api/Users`, method: 'GET', signal });

  const query = useQuery<UserDto[], TError, TData>({
    queryKey,
    queryFn,
    ...queryOptions,
  });

  return {
    queryKey,
    ...query,
  };
};
```

**The Mutation Hook (POST):**
It types the input `data` argument strictly against `CreateUserRequest`.

```typescript
/**
 * Creates a new user.
 * @summary Creates a new user.
 */
export const useCreateUser = <TError = unknown, TContext = unknown>(options?: {
  mutation?: UseMutationOptions<
    UserDto,     // Response Type
    TError,      // Error Type
    { data: CreateUserRequest }, // Variables Type (Input)
    TContext
  >,
}) => {
  const { mutation: mutationOptions } = options ?? {};

  const mutationFn: MutationFunction<
    UserDto,
    { data: CreateUserRequest }
  > = (props) => {
    const { data } = props ?? {};

    return customInstance<UserDto>({
      url: `/api/Users`,
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      data,
    });
  };

  return useMutation<UserDto, TError, { data: CreateUserRequest }, TContext>({
    mutationFn,
    ...mutationOptions,
  });
};
```

### Why this generated code is safe

1.  **Impossible to Typo:** You cannot request a user property that doesn't exist. `data.email` will be a compile error because `UserDto` has no email.
2.  **Impossible to Mistake Enums:** You cannot pass `role: 'SuperAdmin'` to the create mutation. TypeScript will force you to use `UserRole.Admin` or `UserRole.User`.
3.  **Centralized Control:** If you change the C\# controller to require an `Email` field, the next time you run `npx orval`, the frontend build will break immediately, telling you exactly where you need to fix your code.

-----

### Step 5: Using the Generated Code

#### Part A: The List Component (Fetching Data)

*Note: We export `queryClient` setup in your `App.tsx` or main entry point to use it here.*

**`src/components/UserList.tsx`**

```tsx
import { useGetUsers } from '../api/generated';

export const UserList = () => {
  // 1. The generated hook handles the GET request
  const { data: users, isLoading } = useGetUsers();

  if (isLoading) return <p>Loading users...</p>;

  return (
    <div className="p-4">
      <h2 className="text-xl font-bold mb-4">User Directory</h2>
      <ul className="space-y-2">
        {users?.map((user) => (
          <li key={user.id} className="border p-2 rounded shadow-sm">
            {user.username} <span className="text-gray-500">({user.role})</span>
          </li>
        ))}
      </ul>
    </div>
  );
};
```

#### Part B: The Form Component (Mutation)

This is where the magic happens. We use the generated `useCreateUser` hook.

**`src/components/CreateUserForm.tsx`**

```tsx
import { useState } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import { useCreateUser, getGetUsersQueryKey } from '../api/generated';
import { UserRole } from '../api/generated'; // Orval even generates your Enums!

export const CreateUserForm = () => {
  const [username, setUsername] = useState('');
  const queryClient = useQueryClient();

  // 1. The generated Mutation Hook
  const { mutate, isPending } = useCreateUser();

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    // 2. Call mutate.
    // TypeScript ensures 'data' matches the C# CreateUserRequest exactly.
    mutate(
      {
        data: {
          username: username,
          role: UserRole.User,
        },
      },
      {
        // 3. On Success: Refresh the list automatically
        onSuccess: () => {
          // Orval generates this key helper function for you!
          queryClient.invalidateQueries({ queryKey: getGetUsersQueryKey() });
          setUsername(''); // Clear form
        },
        onError: (error) => {
          alert(`Failed to create user: ${error.message}`);
        }
      }
    );
  };

  return (
    <form onSubmit={handleSubmit} className="p-4 border-t mt-4">
      <h3 className="font-bold mb-2">Add New User</h3>
      <div className="flex gap-2">
        <input
          type="text"
          value={username}
          onChange={(e) => setUsername(e.target.value)}
          placeholder="Enter username"
          className="border p-2 rounded"
          disabled={isPending}
        />
        <button
          type="submit"
          disabled={isPending}
          className="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600 disabled:bg-gray-400"
        >
          {isPending ? 'Saving...' : 'Create'}
        </button>
      </div>
    </form>
  );
};
```

### Why this is powerful:

1.  **Strict Typing:** If you try to pass `email` to `mutate({ data: { ... } })`, TypeScript will yell at you immediately because your C\# `CreateUserRequest` didn't have an email field.
2.  **Enum Support:** You can import `UserRole` directly from the generated file. You don't need to manually redefine "Admin" vs "User" on the frontend.
3.  **Key Factory:** `getGetUsersQueryKey()` is a helper generated by Orval. You don't have to guess if the cache key is `['users']` or `['GetUsers']`—Orval manages the keys for you.

**Next Step:**
Would you like to see how to implement **Zod Validation** on the frontend side using the same OpenAPI spec? Orval can generate Zod schemas (`userSchema`, `createUserSchema`) alongside the hooks so you can validate forms *before* sending them to the API.
