# T3 Stack Component Templates

## Design System

### Tailwind Config with Glass Design System
```typescript
// tailwind.config.ts
import { type Config } from "tailwindcss";

export default {
  content: ["./src/**/*.tsx"],
  theme: {
    extend: {
      colors: {
        // Primary - Coral/orange
        primary: {
          50: "#fef7f4",
          100: "#fdeee8",
          200: "#fbd9cc",
          300: "#f8bba3",
          400: "#f39470",
          500: "#eb6d45",
          600: "#d85332",
          700: "#b54228",
          800: "#943826",
          900: "#7a3224",
          950: "#421710",
        },
        // Secondary - Slate blue
        secondary: {
          50: "#f6f8fb",
          100: "#ebeef5",
          200: "#d3dbe8",
          300: "#acbbd3",
          400: "#7f96b9",
          500: "#5f78a0",
          600: "#4b6085",
          700: "#3e4e6c",
          800: "#36445b",
          900: "#303b4d",
          950: "#1f2633",
        },
        // Neutral - Warm grays
        neutral: {
          50: "#fafaf9",
          100: "#f5f5f4",
          200: "#e7e5e4",
          300: "#d6d3d1",
          400: "#a8a29e",
          500: "#78716c",
          600: "#57534e",
          700: "#44403c",
          800: "#292524",
          900: "#1c1917",
          950: "#0c0a09",
        },
      },
      boxShadow: {
        glass: "0 4px 30px rgba(0, 0, 0, 0.05)",
        "glass-lg": "0 8px 32px rgba(0, 0, 0, 0.08)",
        soft: "0 2px 8px rgba(0, 0, 0, 0.04), 0 4px 16px rgba(0, 0, 0, 0.04)",
        "soft-lg": "0 4px 12px rgba(0, 0, 0, 0.05), 0 8px 32px rgba(0, 0, 0, 0.08)",
      },
      backgroundImage: {
        "gradient-mesh": `
          radial-gradient(at 40% 20%, rgba(235, 109, 69, 0.08) 0px, transparent 50%),
          radial-gradient(at 80% 0%, rgba(95, 120, 160, 0.08) 0px, transparent 50%),
          radial-gradient(at 0% 50%, rgba(235, 109, 69, 0.05) 0px, transparent 50%)
        `,
      },
      animation: {
        "fade-in": "fadeIn 0.5s ease-out",
        "slide-up": "slideUp 0.5s ease-out",
      },
      keyframes: {
        fadeIn: {
          "0%": { opacity: "0" },
          "100%": { opacity: "1" },
        },
        slideUp: {
          "0%": { opacity: "0", transform: "translateY(10px)" },
          "100%": { opacity: "1", transform: "translateY(0)" },
        },
      },
    },
  },
  plugins: [],
} satisfies Config;
```

### Global CSS with Glass Effects
```css
/* globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --glass-blur: 12px;
    --glass-bg: rgba(255, 255, 255, 0.7);
    --glass-border: rgba(255, 255, 255, 0.2);
  }

  html {
    scroll-behavior: smooth;
    -webkit-font-smoothing: antialiased;
  }

  :focus-visible {
    outline: 2px solid #eb6d45;
    outline-offset: 2px;
  }
}

@layer components {
  .glass {
    background: var(--glass-bg);
    backdrop-filter: blur(var(--glass-blur));
    -webkit-backdrop-filter: blur(var(--glass-blur));
    border: 1px solid var(--glass-border);
  }

  .container-lg {
    @apply mx-auto max-w-7xl px-4 sm:px-6 lg:px-8;
  }

  .section {
    @apply py-16 md:py-24;
  }

  .card-hover {
    @apply transition-all duration-300 ease-out;
  }

  .card-hover:hover {
    @apply -translate-y-1 shadow-soft-lg;
  }
}
```

## UI Components

### Button Component with Glass Variant
```typescript
// components/ui/button.tsx
import { forwardRef, type ButtonHTMLAttributes } from "react";
import { cn } from "~/lib/utils";

export interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary" | "outline" | "ghost" | "destructive" | "glass";
  size?: "sm" | "md" | "lg";
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant = "primary", size = "md", ...props }, ref) => {
    return (
      <button
        ref={ref}
        className={cn(
          "inline-flex items-center justify-center font-medium transition-all duration-200 ease-out",
          "focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500 focus-visible:ring-offset-2",
          "disabled:pointer-events-none disabled:opacity-50",
          "active:scale-[0.98]",
          {
            "bg-gradient-to-b from-primary-500 to-primary-600 text-white shadow-soft hover:from-primary-600 hover:to-primary-700":
              variant === "primary",
            "bg-neutral-100 text-neutral-700 hover:bg-neutral-200 shadow-soft":
              variant === "secondary",
            "border border-neutral-200 bg-white/50 text-neutral-700 shadow-soft hover:bg-neutral-50":
              variant === "outline",
            "text-neutral-600 hover:bg-neutral-100 hover:text-neutral-900":
              variant === "ghost",
            "bg-gradient-to-b from-red-500 to-red-600 text-white shadow-soft hover:from-red-600 hover:to-red-700":
              variant === "destructive",
            "bg-white/70 backdrop-blur-md border border-white/20 text-neutral-800 shadow-glass hover:bg-white/80":
              variant === "glass",
          },
          {
            "h-8 rounded-lg px-3 text-sm": size === "sm",
            "h-10 rounded-xl px-4 text-sm": size === "md",
            "h-12 rounded-xl px-6 text-base": size === "lg",
          },
          className
        )}
        {...props}
      />
    );
  }
);

Button.displayName = "Button";
```

### Card Component with Variants
```typescript
// components/ui/card.tsx
import { type HTMLAttributes, forwardRef } from "react";
import { cn } from "~/lib/utils";

export interface CardProps extends HTMLAttributes<HTMLDivElement> {
  variant?: "default" | "glass" | "elevated";
}

export const Card = forwardRef<HTMLDivElement, CardProps>(
  ({ className, variant = "default", ...props }, ref) => (
    <div
      ref={ref}
      className={cn(
        "rounded-2xl transition-all duration-300",
        {
          "bg-white border border-neutral-100 shadow-soft": variant === "default",
          "bg-white/70 backdrop-blur-xl border border-white/20 shadow-glass": variant === "glass",
          "bg-white border border-neutral-100 shadow-soft-lg": variant === "elevated",
        },
        className
      )}
      {...props}
    />
  )
);
Card.displayName = "Card";

export const CardHeader = forwardRef<HTMLDivElement, HTMLAttributes<HTMLDivElement>>(
  ({ className, ...props }, ref) => (
    <div ref={ref} className={cn("flex flex-col space-y-1.5 p-6", className)} {...props} />
  )
);
CardHeader.displayName = "CardHeader";

export const CardTitle = forwardRef<HTMLHeadingElement, HTMLAttributes<HTMLHeadingElement>>(
  ({ className, ...props }, ref) => (
    <h3 ref={ref} className={cn("text-xl font-semibold text-neutral-900", className)} {...props} />
  )
);
CardTitle.displayName = "CardTitle";

export const CardContent = forwardRef<HTMLDivElement, HTMLAttributes<HTMLDivElement>>(
  ({ className, ...props }, ref) => (
    <div ref={ref} className={cn("p-6 pt-0", className)} {...props} />
  )
);
CardContent.displayName = "CardContent";
```

### Input Component with Glass Variant
```typescript
// components/ui/input.tsx
import { forwardRef, type InputHTMLAttributes } from "react";
import { cn } from "~/lib/utils";

export interface InputProps extends InputHTMLAttributes<HTMLInputElement> {
  variant?: "default" | "glass";
}

export const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ className, type, variant = "default", ...props }, ref) => {
    return (
      <input
        type={type}
        className={cn(
          "flex h-11 w-full rounded-xl px-4 py-2 text-sm transition-all duration-200",
          "placeholder:text-neutral-400",
          "focus:outline-none focus:ring-2 focus:ring-primary-500/20 focus:border-primary-500",
          "disabled:cursor-not-allowed disabled:opacity-50",
          {
            "border border-neutral-200 bg-white text-neutral-900 shadow-soft hover:border-neutral-300":
              variant === "default",
            "border border-white/20 bg-white/50 backdrop-blur-md text-neutral-900 shadow-glass":
              variant === "glass",
          },
          className
        )}
        ref={ref}
        {...props}
      />
    );
  }
);

Input.displayName = "Input";
```

## Layout Components

### Header with Scroll Effect
```typescript
// components/layout/header.tsx
"use client";

import Link from "next/link";
import { useState, useEffect } from "react";
import { cn } from "~/lib/utils";

export function Header() {
  const [scrolled, setScrolled] = useState(false);

  useEffect(() => {
    const handleScroll = () => setScrolled(window.scrollY > 10);
    window.addEventListener("scroll", handleScroll);
    return () => window.removeEventListener("scroll", handleScroll);
  }, []);

  return (
    <header
      className={cn(
        "sticky top-0 z-50 transition-all duration-300",
        scrolled
          ? "bg-white/80 backdrop-blur-xl border-b border-neutral-200/50 shadow-soft"
          : "bg-transparent"
      )}
    >
      <div className="container-lg">
        <div className="flex h-16 items-center justify-between">
          <Link href="/" className="text-xl font-bold text-neutral-900">
            Brand
          </Link>
          <nav className="hidden items-center gap-1 md:flex">
            {navItems.map((item) => (
              <Link
                key={item.href}
                href={item.href}
                className="px-3 py-2 text-sm font-medium text-neutral-600 rounded-lg hover:text-neutral-900 hover:bg-neutral-100 transition-all"
              >
                {item.label}
              </Link>
            ))}
          </nav>
        </div>
      </div>
    </header>
  );
}
```

### Admin Sidebar
```typescript
// components/layout/admin-sidebar.tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import { cn } from "~/lib/utils";

const navItems = [
  { label: "Dashboard", href: "/admin", icon: "home" },
  { label: "Posts", href: "/admin/posts", icon: "document" },
  { label: "Users", href: "/admin/users", icon: "users", adminOnly: true },
];

export function AdminSidebar() {
  const pathname = usePathname();

  return (
    <aside className="fixed left-0 top-0 z-40 h-screen w-64 border-r border-neutral-200 bg-white">
      <div className="flex h-16 items-center border-b border-neutral-200 px-6">
        <Link href="/admin" className="text-lg font-bold text-neutral-900">
          Admin
        </Link>
      </div>
      <nav className="p-4">
        <ul className="space-y-1">
          {navItems.map((item) => {
            const isActive = item.href === "/admin"
              ? pathname === "/admin"
              : pathname.startsWith(item.href);

            return (
              <li key={item.href}>
                <Link
                  href={item.href}
                  className={cn(
                    "flex items-center gap-3 rounded-xl px-3 py-2.5 text-sm font-medium transition-all",
                    isActive
                      ? "bg-primary-50 text-primary-700 shadow-soft"
                      : "text-neutral-600 hover:bg-neutral-100"
                  )}
                >
                  {item.label}
                </Link>
              </li>
            );
          })}
        </ul>
      </nav>
    </aside>
  );
}
```

## Feature Components

### Post Card
```typescript
// components/features/posts/post-card.tsx
import Link from "next/link";
import { format } from "date-fns";
import { Card, CardContent } from "~/components/ui/card";

interface PostCardProps {
  post: {
    id: string;
    title: string;
    slug: string;
    excerpt: string | null;
    publishedAt: Date | null;
    author: { name: string | null };
  };
}

export function PostCard({ post }: PostCardProps) {
  return (
    <Link href={`/blog/${post.slug}`} className="group block h-full">
      <Card className="h-full overflow-hidden card-hover">
        <div className="aspect-[16/10] bg-gradient-to-br from-neutral-100 to-neutral-200" />
        <CardContent className="p-6">
          <h3 className="text-lg font-semibold text-neutral-900 line-clamp-2 group-hover:text-primary-600 transition-colors">
            {post.title}
          </h3>
          {post.excerpt && (
            <p className="mt-2 text-sm text-neutral-600 line-clamp-2">
              {post.excerpt}
            </p>
          )}
          <div className="mt-4 flex items-center gap-3 text-sm text-neutral-500">
            <span>{post.author.name ?? "Anonymous"}</span>
            {post.publishedAt && (
              <>
                <span className="text-neutral-300">·</span>
                <span>{format(new Date(post.publishedAt), "MMM d, yyyy")}</span>
              </>
            )}
          </div>
        </CardContent>
      </Card>
    </Link>
  );
}
```

### Newsletter Form
```typescript
// components/features/newsletter-form.tsx
"use client";

import { useState } from "react";
import { api } from "~/trpc/react";
import { Button } from "~/components/ui/button";
import { Input } from "~/components/ui/input";

export function NewsletterForm() {
  const [email, setEmail] = useState("");
  const [success, setSuccess] = useState(false);

  const mutation = api.newsletter.subscribe.useMutation({
    onSuccess: () => {
      setSuccess(true);
      setEmail("");
    },
  });

  if (success) {
    return (
      <div className="rounded-2xl bg-white/10 backdrop-blur-sm border border-white/20 p-6 text-center">
        <p className="text-white font-medium">Thank you for subscribing!</p>
      </div>
    );
  }

  return (
    <form onSubmit={(e) => { e.preventDefault(); mutation.mutate({ email }); }} className="space-y-4">
      <div className="flex flex-col gap-3 sm:flex-row">
        <Input
          type="email"
          placeholder="Email address"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
          variant="glass"
          className="bg-white/10 border-white/20 text-white placeholder:text-white/50"
        />
        <Button
          type="submit"
          disabled={mutation.isPending}
          className="bg-white text-secondary-900 hover:bg-white/90"
        >
          Subscribe
        </Button>
      </div>
    </form>
  );
}
```

## Page Templates

### Homepage
```typescript
// app/(public)/page.tsx
import Link from "next/link";
import { api } from "~/trpc/server";
import { Button } from "~/components/ui/button";
import { PostCard } from "~/components/features/posts/post-card";

export default async function HomePage() {
  const posts = await api.posts.getPublished({ limit: 3 });

  return (
    <div className="min-h-screen">
      {/* Hero */}
      <section className="relative overflow-hidden bg-gradient-to-br from-primary-500 via-primary-600 to-primary-700 py-24 md:py-32">
        <div className="absolute -top-24 -right-24 h-96 w-96 rounded-full bg-white/10 blur-3xl" />
        <div className="container-lg relative">
          <div className="mx-auto max-w-3xl text-center">
            <h1 className="text-4xl font-bold text-white md:text-5xl animate-fade-in">
              Your Headline Here
            </h1>
            <p className="mt-6 text-lg text-white/90 animate-slide-up">
              Your subheadline describing the value proposition.
            </p>
            <div className="mt-10 flex justify-center gap-4 animate-slide-up">
              <Link href="/about">
                <Button size="lg" className="bg-white text-primary-700 hover:bg-white/90">
                  Learn More
                </Button>
              </Link>
            </div>
          </div>
        </div>
      </section>

      {/* Featured Posts */}
      <section className="section bg-neutral-50">
        <div className="container-lg">
          <h2 className="text-3xl font-bold text-neutral-900 mb-10">Latest Posts</h2>
          <div className="grid gap-8 md:grid-cols-2 lg:grid-cols-3">
            {posts.map((post) => (
              <PostCard key={post.id} post={post} />
            ))}
          </div>
        </div>
      </section>
    </div>
  );
}
```

### Login Page
```typescript
// app/(auth)/login/page.tsx
"use client";

import { useState } from "react";
import { signIn } from "next-auth/react";
import { useRouter } from "next/navigation";
import { Button } from "~/components/ui/button";
import { Input } from "~/components/ui/input";
import { Card, CardContent, CardHeader, CardTitle } from "~/components/ui/card";

export default function LoginPage() {
  const router = useRouter();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");
  const [isLoading, setIsLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setIsLoading(true);
    setError("");

    const result = await signIn("credentials", {
      email,
      password,
      redirect: false,
    });

    if (result?.error) {
      setError("Invalid email or password");
      setIsLoading(false);
    } else {
      router.push("/admin");
      router.refresh();
    }
  };

  return (
    <Card variant="elevated">
      <CardHeader className="text-center">
        <CardTitle>Welcome back</CardTitle>
      </CardHeader>
      <CardContent>
        <form onSubmit={handleSubmit} className="space-y-4">
          {error && (
            <div className="rounded-xl bg-red-50 border border-red-100 px-4 py-3 text-sm text-red-600">
              {error}
            </div>
          )}
          <Input
            type="email"
            placeholder="Email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
          />
          <Input
            type="password"
            placeholder="Password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
          />
          <Button type="submit" className="w-full" disabled={isLoading}>
            {isLoading ? "Signing in..." : "Sign In"}
          </Button>
        </form>
      </CardContent>
    </Card>
  );
}
```

## Utility Functions

### cn() Utility
```typescript
// lib/utils.ts
import { type ClassValue, clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

### Slug Generation
```typescript
// lib/utils.ts
export function generateSlug(text: string): string {
  return text
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, "-")
    .replace(/^-+|-+$/g, "");
}
```
