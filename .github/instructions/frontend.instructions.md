---
applyTo: "frontend/**"
---

# Frontend Instructions

## Project Structure

- `src/client/` — **Auto-generated** API client. Never edit manually. Regenerate via `bash scripts/generate-client.sh` from project root.
- `src/routes/` — TanStack Router file-based routes. `routeTree.gen.ts` is auto-generated.
- `src/components/ui/` — shadcn/ui primitives. Don't edit directly; add new ones via shadcn CLI.
- `src/components/{Items,Admin,UserSettings}/` — Feature components grouped by domain.
- `src/components/Common/` — Shared components: `DataTable`, `AuthLayout`, `ErrorComponent`, `NotFound`.
- `src/hooks/` — Custom hooks: `useAuth`, `useCustomToast`, `useCopyToClipboard`.
- `src/lib/utils.ts` — `cn()` helper for class merging (clsx + tailwind-merge).

## Key Conventions

- **Path alias**: `@/` maps to `src/`. Always use `@/` imports (e.g., `import { Button } from "@/components/ui/button"`).
- **Linting**: Biome (not ESLint/Prettier). Run: `bun run lint`. Config in `biome.json`.
- **Package manager**: Bun for scripts, but project uses npm-compatible `package.json`.
- **Styling**: Tailwind CSS v4 via `@tailwindcss/vite` plugin. Use utility classes, no CSS modules.

## Routing Pattern

Routes in `src/routes/` use TanStack Router's file-based convention:

```tsx
import { createFileRoute } from "@tanstack/react-router"

export const Route = createFileRoute("/_layout/items")({
  component: Items,
  head: () => ({
    meta: [{ title: "Items - FastAPI Template" }],
  }),
})
```

- `_layout.tsx` — Wraps authenticated pages with sidebar. Redirects to `/login` if `!isLoggedIn()`.
- `_layout/*.tsx` — Authenticated pages: `index.tsx`, `items.tsx`, `admin.tsx`, `settings.tsx`.
- Top-level routes (`login.tsx`, `signup.tsx`, etc.) are public.

## API Client & Data Fetching

The client is generated from the backend's OpenAPI schema via `@hey-api/openapi-ts`. Services are class-based:

```tsx
import { ItemsService } from "@/client"

// In queries:
const { data } = useSuspenseQuery({
  queryFn: () => ItemsService.readItems({ skip: 0, limit: 100 }),
  queryKey: ["items"],
})

// In mutations:
const mutation = useMutation({
  mutationFn: (data: ItemCreate) => ItemsService.createItem({ requestBody: data }),
  onSettled: () => queryClient.invalidateQueries({ queryKey: ["items"] }),
})
```

- Query keys: `["currentUser"]`, `["users"]`, `["items"]` — always invalidate on mutations via `onSettled`.
- Auth token: stored in `localStorage` as `"access_token"`, auto-attached by `OpenAPI.TOKEN` in `main.tsx`.
- 401/403 errors globally handled — clears token and redirects to `/login`.

## Form Pattern

All forms use `react-hook-form` + `zod` + shadcn Form components:

```tsx
const formSchema = z.object({
  title: z.string().min(1, { message: "Title is required" }),
  description: z.string().optional(),
})

const form = useForm<z.infer<typeof formSchema>>({
  resolver: zodResolver(formSchema),
  mode: "onBlur",
  criteriaMode: "all",
  defaultValues: { title: "", description: "" },
})
```

## Error Handling

Use `handleError` from `src/utils.ts` bound to toast: `onError: handleError.bind(showErrorToast)`. This extracts messages from Axios errors and API error bodies.

## Component Pattern for CRUD Features

Each entity has these components in `src/components/{Entity}/`:
- `Add{Entity}.tsx` — Dialog with form + create mutation
- `Edit{Entity}.tsx` — Dialog with form + update mutation
- `Delete{Entity}.tsx` — Confirmation dialog + delete mutation
- `columns.tsx` — TanStack Table column definitions (`ColumnDef<EntityPublic>[]`)
- `{Entity}ActionsMenu.tsx` — Dropdown with Edit/Delete actions

Data tables use the shared `DataTable` component from `src/components/Common/DataTable.tsx`.

## E2E Testing (Playwright)

- Tests in `frontend/tests/`. Run via Docker: `docker compose run --rm playwright bunx playwright test`.
- Auth setup in `tests/auth.setup.ts` — logs in as superuser, saves storage state.
- Test config reads from root `.env` (superuser credentials).
- Test utilities: `tests/utils/privateApi.ts` (direct API calls), `tests/utils/random.ts` (test data generators).
- Tests use `data-testid` attributes for stable selectors (e.g., `getByTestId("email-input")`).

## Adding a New Page/Feature

1. Regenerate client if backend changed: `bash scripts/generate-client.sh`
2. Create route file: `src/routes/_layout/entity.tsx`
3. Create components: `src/components/Entity/{Add,Edit,Delete,columns,ActionsMenu}.tsx`
4. Add navigation link in `src/components/Sidebar/AppSidebar.tsx`
5. Run `bun run lint` to verify
