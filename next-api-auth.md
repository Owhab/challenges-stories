# Handling REST API Authentication in Next.js Middleware with Zustand, React Query, Zod, and React Hook Form

## Table of Contents
1. [Introduction](#introduction)
2. [Setting Up Next.js Middleware](#setting-up-nextjs-middleware)
3. [State Management with Zustand](#state-management-with-zustand)
4. [Data Fetching with React Query](#data-fetching-with-react-query)
5. [Form Validation with Zod and React Hook Form](#form-validation-with-zod-and-react-hook-form)
6. [Putting It All Together](#putting-it-all-together)
7. [Conclusion](#conclusion)

## Introduction
In this guide, we will explore how to handle REST API authentication in a Next.js application using middleware, Zustand for state management, React Query for data fetching, and Zod with React Hook Form for form validation.

## Setting Up Next.js Middleware
Next.js middleware allows you to run code before a request is completed. This is useful for authentication.

```javascript
// middleware.js
import { NextResponse } from 'next/server';

export function middleware(req) {
    const token = req.cookies['auth-token'];
    if (!token) {
        return NextResponse.redirect('/login');
    }
    return NextResponse.next();
}
```

## State Management with Zustand
Zustand is a small, fast, and scalable state management solution.

```javascript
// store.js
import create from 'zustand';

const useAuthStore = create((set) => ({
    token: null,
    setToken: (token) => set({ token }),
}));

export default useAuthStore;
```

## Data Fetching with React Query
React Query simplifies data fetching and caching.

```javascript
// api.js
import { useQuery } from 'react-query';

const fetchUser = async (token) => {
    const response = await fetch('/api/user', {
        headers: {
            Authorization: `Bearer ${token}`,
        },
    });
    return response.json();
};

export const useUser = (token) => useQuery(['user', token], () => fetchUser(token));
```

## Form Validation with Zod and React Hook Form
Zod is a TypeScript-first schema declaration and validation library.

```javascript
// loginForm.js
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';

const schema = z.object({
    email: z.string().email(),
    password: z.string().min(6),
});

const LoginForm = () => {
    const { register, handleSubmit, formState: { errors } } = useForm({
        resolver: zodResolver(schema),
    });

    const onSubmit = (data) => {
        // handle login
    };

    return (
        <form onSubmit={handleSubmit(onSubmit)}>
            <input {...register('email')} />
            {errors.email && <span>{errors.email.message}</span>}
            <input {...register('password')} type="password" />
            {errors.password && <span>{errors.password.message}</span>}
            <button type="submit">Login</button>
        </form>
    );
};

export default LoginForm;
```

## Putting It All Together
Integrate all the parts into your Next.js application.

```javascript
// pages/_app.js
import { QueryClient, QueryClientProvider } from 'react-query';
import useAuthStore from '../store';

const queryClient = new QueryClient();

function MyApp({ Component, pageProps }) {
    const { token, setToken } = useAuthStore();

    return (
        <QueryClientProvider client={queryClient}>
            <Component {...pageProps} />
        </QueryClientProvider>
    );
}

export default MyApp;
```

## Conclusion
By combining Next.js middleware, Zustand, React Query, Zod, and React Hook Form, you can create a robust authentication system for your application. This setup ensures that your application is secure, scalable, and easy to maintain.