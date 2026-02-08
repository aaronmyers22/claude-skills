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

### NextAuth v5 (Auth.js) with Credentials Provider
```typescript
// server/auth.ts
import NextAuth from "next-auth";
import { PrismaAdapter } from "@auth/prisma-adapter";
import Credentials from "next-auth/providers/credentials";
import bcrypt from "bcryptjs";
import { db } from "~/server/db";

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(db),
  session: { strategy: "jwt" }, // Required for credentials
  providers: [
    Credentials({
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) return null;

        const user = await db.user.findUnique({
          where: { email: credentials.email as string },
        });

        if (!user?.password) return null;

        const isValid = await bcrypt.compare(credentials.password as string, user.password);
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
});
```

### Protected Layout Pattern (Mobile Responsive)
```typescript
// app/admin/layout.tsx
import { redirect } from "next/navigation";
import { auth } from "~/server/auth";
import { AdminLayoutClient } from "~/components/layout/admin-layout-client";

export default async function AdminLayout({ children }: { children: React.ReactNode }) {
  const session = await auth();

  if (!session) {
    redirect("/login?callbackUrl=/admin");
  }

  if (session.user.role !== "ADMIN" && session.user.role !== "EDITOR") {
    redirect("/");
  }

  return (
    <AdminLayoutClient
      userName={session.user.name ?? null}
      userEmail={session.user.email ?? null}
      userRole={session.user.role}
    >
      {children}
    </AdminLayoutClient>
  );
}
```

### Mobile Responsive Admin Layout Client
```typescript
// components/layout/admin-layout-client.tsx
"use client";

import { useState, useEffect } from "react";
import Link from "next/link";
import { usePathname } from "next/navigation";
import { cn } from "~/lib/utils";

interface AdminLayoutClientProps {
  children: React.ReactNode;
  userName: string | null;
  userEmail: string | null;
  userRole: string;
}

export function AdminLayoutClient({ children, userName, userEmail, userRole }: AdminLayoutClientProps) {
  const [sidebarOpen, setSidebarOpen] = useState(false);
  const pathname = usePathname();

  // Close sidebar on route change (mobile)
  useEffect(() => {
    setSidebarOpen(false);
  }, [pathname]);

  // Lock body scroll when sidebar is open on mobile
  useEffect(() => {
    if (sidebarOpen) {
      document.body.style.overflow = "hidden";
    } else {
      document.body.style.overflow = "";
    }
    return () => { document.body.style.overflow = ""; };
  }, [sidebarOpen]);

  return (
    <div className="min-h-screen bg-neutral-50">
      {/* Mobile backdrop */}
      {sidebarOpen && (
        <div className="fixed inset-0 z-40 bg-black/40 lg:hidden" onClick={() => setSidebarOpen(false)} />
      )}

      {/* Sidebar - hidden on mobile, visible on lg+ */}
      <aside className={cn(
        "fixed left-0 top-0 z-50 h-screen w-64 border-r border-neutral-200 bg-white transition-transform duration-300 lg:translate-x-0",
        sidebarOpen ? "translate-x-0" : "-translate-x-full"
      )}>
        {/* Sidebar content */}
      </aside>

      {/* Main content - full width on mobile */}
      <div className="lg:ml-64 min-h-screen">
        <header className="sticky top-0 z-30 flex h-16 items-center justify-between border-b bg-white/80 backdrop-blur-xl px-4 lg:px-6">
          {/* Hamburger for mobile */}
          <button className="lg:hidden" onClick={() => setSidebarOpen(true)}>☰</button>
          <div className="hidden lg:block" />
          <div className="flex items-center gap-3">
            <span className="text-sm text-neutral-600 hidden sm:block">{userName ?? userEmail}</span>
            <span className="rounded-full bg-primary-50 px-3 py-1 text-xs font-medium text-primary-700">{userRole}</span>
          </div>
        </header>
        <main className="p-4 lg:p-6">{children}</main>
      </div>
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

## User Management Patterns

### Full User Router with CRUD, Password Reset, Delete
```typescript
// server/api/routers/users.ts
import { z } from "zod";
import { createTRPCRouter, adminProcedure } from "~/server/api/trpc";
import { hashPassword } from "~/server/auth";
import { TRPCError } from "@trpc/server";

export const usersRouter = createTRPCRouter({
  getAll: adminProcedure.query(async ({ ctx }) => {
    return ctx.db.user.findMany({
      orderBy: { createdAt: "desc" },
      select: { id: true, name: true, email: true, image: true, role: true, createdAt: true },
    });
  }),

  create: adminProcedure
    .input(z.object({
      name: z.string().min(1),
      email: z.string().email(),
      password: z.string().min(8),
      role: z.enum(["ADMIN", "EDITOR", "PUBLIC"]).default("EDITOR"),
    }))
    .mutation(async ({ ctx, input }) => {
      const existing = await ctx.db.user.findUnique({ where: { email: input.email } });
      if (existing) throw new TRPCError({ code: "CONFLICT", message: "Email already exists" });

      const hashedPassword = await hashPassword(input.password);
      return ctx.db.user.create({
        data: { name: input.name, email: input.email, password: hashedPassword, role: input.role },
      });
    }),

  update: adminProcedure
    .input(z.object({
      id: z.string(),
      name: z.string().min(1).optional(),
      email: z.string().email().optional(),
      image: z.string().nullable().optional(),
      role: z.enum(["ADMIN", "EDITOR", "PUBLIC"]).optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      const { id, ...data } = input;
      if (data.email) {
        const existing = await ctx.db.user.findFirst({ where: { email: data.email, NOT: { id } } });
        if (existing) throw new TRPCError({ code: "CONFLICT", message: "Email already exists" });
      }
      return ctx.db.user.update({ where: { id }, data });
    }),

  resetPassword: adminProcedure
    .input(z.object({ id: z.string(), newPassword: z.string().min(8) }))
    .mutation(async ({ ctx, input }) => {
      const hashedPassword = await hashPassword(input.newPassword);
      return ctx.db.user.update({ where: { id: input.id }, data: { password: hashedPassword } });
    }),

  delete: adminProcedure
    .input(z.object({ id: z.string() }))
    .mutation(async ({ ctx, input }) => {
      // Prevent self-deletion
      if (input.id === ctx.session.user.id) {
        throw new TRPCError({ code: "BAD_REQUEST", message: "Cannot delete your own account" });
      }
      return ctx.db.user.delete({ where: { id: input.id } });
    }),
});
```

### Mobile Responsive Admin List Page Pattern
```typescript
// Card layout for mobile, table for desktop
{items.length > 0 && (
  <>
    {/* Mobile card layout */}
    <div className="space-y-4 md:hidden">
      {items.map((item) => (
        <div key={item.id} className="rounded-lg border p-4">
          <div className="flex items-start justify-between gap-2">
            <div className="flex-1 min-w-0">
              <p className="font-medium truncate">{item.name}</p>
              <p className="text-sm text-gray-600 truncate">{item.email}</p>
            </div>
            <span className="shrink-0 rounded-full px-2 py-0.5 text-xs bg-green-100 text-green-800">
              {item.status}
            </span>
          </div>
          <div className="mt-3 flex gap-2">
            <Button variant="outline" size="sm" className="flex-1">Edit</Button>
            <Button variant="destructive" size="sm" className="flex-1">Delete</Button>
          </div>
        </div>
      ))}
    </div>

    {/* Desktop table layout */}
    <div className="hidden md:block overflow-x-auto">
      <table className="w-full">
        <thead>
          <tr className="border-b text-left">
            <th className="pb-3 font-medium">Name</th>
            <th className="pb-3 font-medium">Status</th>
            <th className="pb-3 font-medium">Actions</th>
          </tr>
        </thead>
        <tbody>
          {items.map((item) => (
            <tr key={item.id} className="border-b">
              <td className="py-3">{item.name}</td>
              <td className="py-3"><StatusBadge status={item.status} /></td>
              <td className="py-3"><ActionButtons item={item} /></td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  </>
)}
```

### Inline Expandable Edit Form Pattern
```typescript
// User card with expandable edit/reset password forms
<div className="rounded-lg border">
  {/* User info row */}
  <div className="p-4">
    <div className="flex flex-col sm:flex-row sm:items-center gap-4">
      <div className="flex items-center gap-3 flex-1 min-w-0">
        <Avatar user={user} />
        <div className="flex-1 min-w-0">
          <p className="font-medium truncate">{user.name}</p>
          <p className="text-sm text-gray-600 truncate">{user.email}</p>
        </div>
      </div>
      <div className="flex flex-wrap items-center gap-2">
        <RoleBadge role={user.role} />
        <Button variant="outline" size="sm" onClick={() => toggleEdit(user.id)}>Edit</Button>
        <Button variant="outline" size="sm" onClick={() => toggleResetPassword(user.id)}>Reset PW</Button>
        <Button variant="destructive" size="sm" onClick={() => handleDelete(user)}>Delete</Button>
      </div>
    </div>
  </div>

  {/* Expandable edit form */}
  {editingId === user.id && (
    <div className="border-t bg-gray-50 p-4">
      <form onSubmit={handleUpdate} className="space-y-4">
        <ImageUpload value={editImage} onChange={setEditImage} folder="users" />
        <Input value={editName} onChange={setEditName} />
        <Input value={editEmail} onChange={setEditEmail} />
        <Button type="submit">Save Changes</Button>
      </form>
    </div>
  )}

  {/* Expandable reset password form */}
  {resetPasswordId === user.id && (
    <div className="border-t bg-yellow-50 p-4">
      <form onSubmit={handleResetPassword}>
        <Input type="password" value={newPassword} onChange={setNewPassword} minLength={8} />
        <Button type="submit">Reset Password</Button>
      </form>
    </div>
  )}
</div>
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

## File Upload & Media Patterns

### Media Model
```prisma
model Media {
  id        String   @id @default(cuid())
  filename  String
  mimeType  String
  size      Int
  url       String
  alt       String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

### Upload API Route (Vercel Blob)
```typescript
// app/api/upload/route.ts
import { put } from "@vercel/blob";
import { auth } from "~/server/auth";

export async function POST(request: Request) {
  const session = await auth();
  if (!session || (session.user.role !== "ADMIN" && session.user.role !== "EDITOR")) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  const formData = await request.formData();
  const file = formData.get("file") as File;
  const folder = formData.get("folder") as string || "uploads";

  if (!file) {
    return Response.json({ error: "No file provided" }, { status: 400 });
  }

  const blob = await put(`${folder}/${file.name}`, file, {
    access: "public",
  });

  return Response.json({ url: blob.url, filename: file.name });
}
```

### Media tRPC Router
```typescript
export const mediaRouter = createTRPCRouter({
  getAll: editorProcedure
    .input(z.object({ limit: z.number().default(50) }))
    .query(({ ctx, input }) => {
      return ctx.db.media.findMany({
        take: input.limit,
        orderBy: { createdAt: "desc" },
      });
    }),

  create: editorProcedure
    .input(z.object({
      filename: z.string(),
      mimeType: z.string(),
      size: z.number(),
      url: z.string(),
      alt: z.string().optional(),
    }))
    .mutation(({ ctx, input }) => {
      return ctx.db.media.create({ data: input });
    }),

  delete: editorProcedure
    .input(z.object({ id: z.string() }))
    .mutation(({ ctx, input }) => {
      return ctx.db.media.delete({ where: { id: input.id } });
    }),
});
```

### Drag-and-Drop Upload Component
```typescript
"use client";

import { useState, useCallback } from "react";

export function useFileUpload() {
  const [isDragging, setIsDragging] = useState(false);
  const [uploading, setUploading] = useState(false);

  const handleDragOver = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    setIsDragging(true);
  }, []);

  const handleDragLeave = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    setIsDragging(false);
  }, []);

  const handleDrop = useCallback(async (
    e: React.DragEvent,
    onUpload: (file: File) => Promise<void>
  ) => {
    e.preventDefault();
    setIsDragging(false);
    const files = Array.from(e.dataTransfer.files).filter(
      (file) => file.type.startsWith("image/")
    );
    setUploading(true);
    for (const file of files) {
      await onUpload(file);
    }
    setUploading(false);
  }, []);

  return { isDragging, uploading, handleDragOver, handleDragLeave, handleDrop };
}
```

### MediaPicker Component Usage
```typescript
import { MediaPicker } from "~/components/ui/media-picker";

<MediaPicker
  label="Image"
  value={imageUrl}
  onChange={setImageUrl}
/>
```

### RichTextEditor with Images
```typescript
import { RichTextEditor } from "~/components/ui/rich-text-editor";

<RichTextEditor
  content={content}
  onChange={setContent}
  placeholder="Start writing..."
/>
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
