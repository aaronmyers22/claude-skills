# T3 Stack Patterns & Best Practices

## Database Patterns

### Prisma Schema with User Roles
```prisma
enum UserRole {
  ADMIN
  EDITOR
  PUBLIC
}

model User {
  id            String    @id @default(cuid())
  name          String?
  email         String    @unique
  password      String?   // For credentials provider
  emailVerified DateTime?
  image         String?
  role          UserRole  @default(PUBLIC)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  // Relations
  accounts      Account[]
  posts         Post[]
}

model Post {
  id          String     @id @default(cuid())
  title       String
  slug        String     @unique
  content     String?    @db.Text
  excerpt     String?
  status      PostStatus @default(DRAFT)
  publishedAt DateTime?
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  // Relations
  author      User       @relation(fields: [authorId], references: [id], onDelete: Cascade)
  authorId    String

  @@index([authorId])
  @@index([slug])
  @@index([status])
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}
```

### Database Best Practices
- Always add `@@index` for foreign keys and frequently queried fields
- Use `cuid()` for IDs (URL-safe, sortable)
- Include `createdAt` and `updatedAt` on all models
- Use `onDelete: Cascade` for child relations
- Use `@db.Text` for long text fields
- Add `slug` fields with unique constraint for SEO-friendly URLs

## tRPC Patterns

### Role-Based Procedures Setup
```typescript
// server/api/trpc.ts
import { TRPCError } from "@trpc/server";

// Public - no auth required
export const publicProcedure = t.procedure.use(timingMiddleware);

// Protected - requires authentication
export const protectedProcedure = t.procedure
  .use(timingMiddleware)
  .use(({ ctx, next }) => {
    if (!ctx.session?.user) {
      throw new TRPCError({ code: "UNAUTHORIZED" });
    }
    return next({ ctx: { session: ctx.session } });
  });

// Editor - requires EDITOR or ADMIN role
export const editorProcedure = protectedProcedure.use(({ ctx, next }) => {
  if (ctx.session.user.role !== "EDITOR" && ctx.session.user.role !== "ADMIN") {
    throw new TRPCError({ code: "FORBIDDEN" });
  }
  return next({ ctx });
});

// Admin - requires ADMIN role only
export const adminProcedure = protectedProcedure.use(({ ctx, next }) => {
  if (ctx.session.user.role !== "ADMIN") {
    throw new TRPCError({ code: "FORBIDDEN" });
  }
  return next({ ctx });
});
```

### Router Organization
```typescript
// server/api/root.ts
import { createTRPCRouter } from "~/server/api/trpc";
import { postRouter } from "./routers/posts";
import { userRouter } from "./routers/users";
import { categoryRouter } from "./routers/categories";

export const appRouter = createTRPCRouter({
  posts: postRouter,
  users: userRouter,
  categories: categoryRouter,
});

export type AppRouter = typeof appRouter;
```

### Input Validation with Zod
```typescript
// lib/validations/post.ts
import { z } from "zod";

export const createPostSchema = z.object({
  title: z.string().min(1, "Title is required").max(100),
  slug: z.string().min(1).regex(/^[a-z0-9-]+$/, "Invalid slug format"),
  content: z.string().optional(),
  excerpt: z.string().max(300).optional(),
  status: z.enum(["DRAFT", "PUBLISHED", "ARCHIVED"]).default("DRAFT"),
});

export const updatePostSchema = createPostSchema.partial().extend({
  id: z.string().cuid(),
});

export type CreatePostInput = z.infer<typeof createPostSchema>;
```

### Full CRUD Router with Roles
```typescript
export const postRouter = createTRPCRouter({
  // Public read
  getPublished: publicProcedure
    .input(z.object({ limit: z.number().default(10) }))
    .query(({ ctx, input }) => {
      return ctx.db.post.findMany({
        where: { status: "PUBLISHED" },
        take: input.limit,
        orderBy: { publishedAt: "desc" },
      });
    }),

  getBySlug: publicProcedure
    .input(z.object({ slug: z.string() }))
    .query(async ({ ctx, input }) => {
      const post = await ctx.db.post.findUnique({
        where: { slug: input.slug },
      });
      if (!post) throw new TRPCError({ code: "NOT_FOUND" });
      return post;
    }),

  // Editor access
  getAll: editorProcedure.query(({ ctx }) => {
    return ctx.db.post.findMany({ orderBy: { createdAt: "desc" } });
  }),

  create: editorProcedure
    .input(createPostSchema)
    .mutation(({ ctx, input }) => {
      return ctx.db.post.create({
        data: { ...input, authorId: ctx.session.user.id },
      });
    }),

  update: editorProcedure
    .input(updatePostSchema)
    .mutation(({ ctx, input }) => {
      const { id, ...data } = input;
      return ctx.db.post.update({ where: { id }, data });
    }),

  // Admin only
  delete: adminProcedure
    .input(z.object({ id: z.string() }))
    .mutation(({ ctx, input }) => {
      return ctx.db.post.delete({ where: { id: input.id } });
    }),
});
```

## Authentication Patterns

### NextAuth with Credentials Provider
```typescript
// server/auth.ts
import { PrismaAdapter } from "@auth/prisma-adapter";
import CredentialsProvider from "next-auth/providers/credentials";
import bcrypt from "bcryptjs";

export const authOptions: NextAuthOptions = {
  adapter: PrismaAdapter(db),
  session: { strategy: "jwt" }, // Required for credentials
  providers: [
    CredentialsProvider({
      name: "credentials",
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) return null;

        const user = await db.user.findUnique({
          where: { email: credentials.email },
        });

        if (!user?.password) return null;

        const isValid = await bcrypt.compare(credentials.password, user.password);
        if (!isValid) return null;

        return { id: user.id, email: user.email, name: user.name, role: user.role };
      },
    }),
  ],
  callbacks: {
    jwt: ({ token, user }) => {
      if (user) token.role = user.role;
      return token;
    },
    session: ({ session, token }) => {
      if (token) session.user.role = token.role;
      return session;
    },
  },
};
```

### Protected Layout Pattern
```typescript
// app/admin/layout.tsx
import { redirect } from "next/navigation";
import { getServerAuthSession } from "~/server/auth";

export default async function AdminLayout({ children }: { children: React.ReactNode }) {
  const session = await getServerAuthSession();

  if (!session) {
    redirect("/login?callbackUrl=/admin");
  }

  if (session.user.role !== "ADMIN" && session.user.role !== "EDITOR") {
    redirect("/");
  }

  return (
    <div className="min-h-screen bg-neutral-50">
      <AdminSidebar />
      <main className="ml-64 p-6">{children}</main>
    </div>
  );
}
```

### Session in Client Components
```typescript
"use client";

import { useSession } from "next-auth/react";

export function UserMenu() {
  const { data: session, status } = useSession();

  if (status === "loading") return <Skeleton />;
  if (!session) return <SignInButton />;

  return (
    <div className="flex items-center gap-3">
      <span>{session.user.name}</span>
      <span className="rounded-full bg-primary-50 px-2 py-1 text-xs">
        {session.user.role}
      </span>
    </div>
  );
}
```

## Route Groups Pattern

### Layout Organization
```
src/app/
├── (public)/              # Public layout with header/footer
│   ├── layout.tsx         # <Header /> + {children} + <Footer />
│   ├── page.tsx           # Homepage
│   ├── about/page.tsx
│   ├── blog/
│   │   ├── page.tsx
│   │   └── [slug]/page.tsx
│   └── contact/page.tsx
├── (auth)/                # Minimal centered layout
│   ├── layout.tsx         # Centered card layout
│   └── login/page.tsx
├── admin/                 # Admin layout with sidebar (not a route group)
│   ├── layout.tsx         # Auth check + sidebar
│   ├── page.tsx           # Dashboard
│   └── posts/page.tsx
└── api/
```

### Public Layout
```typescript
// app/(public)/layout.tsx
import { Header } from "~/components/layout/header";
import { Footer } from "~/components/layout/footer";

export default function PublicLayout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <Header />
      <main>{children}</main>
      <Footer />
    </>
  );
}
```

### Auth Layout
```typescript
// app/(auth)/layout.tsx
export default function AuthLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex min-h-screen items-center justify-center bg-gradient-mesh p-4">
      <div className="w-full max-w-md">{children}</div>
    </div>
  );
}
```

## Error Handling Patterns

### tRPC Error Handling
```typescript
import { TRPCError } from "@trpc/server";

// In router
if (!resource) {
  throw new TRPCError({
    code: "NOT_FOUND",
    message: "Resource not found",
  });
}

if (resource.userId !== ctx.session.user.id) {
  throw new TRPCError({
    code: "FORBIDDEN",
    message: "Not authorized",
  });
}
```

### Error Boundary
```typescript
// app/error.tsx
"use client";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="flex flex-col items-center justify-center min-h-[400px]">
      <h2 className="text-xl font-semibold mb-4">Something went wrong</h2>
      <Button onClick={reset} variant="outline">
        Try again
      </Button>
    </div>
  );
}
```

## Performance Patterns

### Server Component Data Fetching
```typescript
// app/(public)/blog/page.tsx
import { api } from "~/trpc/server";

export default async function BlogPage() {
  const posts = await api.posts.getPublished({ limit: 10 });

  return (
    <section className="section">
      <div className="container-lg">
        <h1 className="text-3xl font-bold mb-8">Blog</h1>
        <div className="grid gap-8 md:grid-cols-2 lg:grid-cols-3">
          {posts.map((post) => (
            <PostCard key={post.id} post={post} />
          ))}
        </div>
      </div>
    </section>
  );
}
```

### Optimistic Updates
```typescript
"use client";

export function CreatePost() {
  const utils = api.useUtils();

  const mutation = api.posts.create.useMutation({
    onSuccess: () => {
      utils.posts.getAll.invalidate();
    },
  });
}
```

### Suspense Boundaries
```typescript
import { Suspense } from "react";

export default function DashboardPage() {
  return (
    <div className="grid gap-6 md:grid-cols-2">
      <Suspense fallback={<CardSkeleton />}>
        <StatsCard />
      </Suspense>
      <Suspense fallback={<CardSkeleton />}>
        <RecentActivity />
      </Suspense>
    </div>
  );
}
```
