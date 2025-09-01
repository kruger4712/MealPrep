# Application Routing Structure

## Overview
Comprehensive guide to the routing architecture in the MealPrep React application using React Router v6 with authentication guards, nested layouts, and performance optimizations.

## Routing Architecture

### Router Configuration
```typescript
// router/index.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import { QueryClient } from '@tanstack/react-query';

import { AuthProvider } from '@/contexts/AuthContext';
import { ProtectedRoute } from '@/components/auth/ProtectedRoute';
import { PublicRoute } from '@/components/auth/PublicRoute';
import { RootLayout } from '@/layouts/RootLayout';
import { AuthLayout } from '@/layouts/AuthLayout';
import { DashboardLayout } from '@/layouts/DashboardLayout';

// Lazy load route components for code splitting
const HomePage = lazy(() => import('@/pages/HomePage'));
const LoginPage = lazy(() => import('@/pages/auth/LoginPage'));
const RegisterPage = lazy(() => import('@/pages/auth/RegisterPage'));
const DashboardPage = lazy(() => import('@/pages/dashboard/DashboardPage'));
const RecipesPage = lazy(() => import('@/pages/recipes/RecipesPage'));
const RecipeDetailPage = lazy(() => import('@/pages/recipes/RecipeDetailPage'));
const MenuPlanningPage = lazy(() => import('@/pages/menu-planning/MenuPlanningPage'));
const FamilySettingsPage = lazy(() => import('@/pages/family/FamilySettingsPage'));
const ProfilePage = lazy(() => import('@/pages/profile/ProfilePage'));
const NotFoundPage = lazy(() => import('@/pages/errors/NotFoundPage'));

export const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorBoundary />,
    children: [
      // Public routes
      {
        index: true,
        element: (
          <PublicRoute>
            <HomePage />
          </PublicRoute>
        ),
      },
      
      // Authentication routes
      {
        path: 'auth',
        element: <AuthLayout />,
        children: [
          {
            path: 'login',
            element: (
              <PublicRoute>
                <LoginPage />
              </PublicRoute>
            ),
          },
          {
            path: 'register',
            element: (
              <PublicRoute>
                <RegisterPage />
              </PublicRoute>
            ),
          },
          {
            path: 'forgot-password',
            element: (
              <PublicRoute>
                <ForgotPasswordPage />
              </PublicRoute>
            ),
          },
          {
            path: 'reset-password/:token',
            element: (
              <PublicRoute>
                <ResetPasswordPage />
              </PublicRoute>
            ),
          },
        ],
      },
      
      // Protected dashboard routes
      {
        path: 'dashboard',
        element: (
          <ProtectedRoute>
            <DashboardLayout />
          </ProtectedRoute>
        ),
        children: [
          {
            index: true,
            element: <DashboardPage />,
            loader: dashboardLoader,
          },
          
          // Recipe management
          {
            path: 'recipes',
            children: [
              {
                index: true,
                element: <RecipesPage />,
                loader: recipesLoader,
              },
              {
                path: 'new',
                element: <CreateRecipePage />,
              },
              {
                path: ':recipeId',
                element: <RecipeDetailPage />,
                loader: recipeDetailLoader,
              },
              {
                path: ':recipeId/edit',
                element: <EditRecipePage />,
                loader: recipeDetailLoader,
              },
            ],
          },
          
          // Menu planning
          {
            path: 'menu-planning',
            children: [
              {
                index: true,
                element: <MenuPlanningPage />,
                loader: menuPlanningLoader,
              },
              {
                path: 'week/:weekStart',
                element: <WeeklyMenuPlanPage />,
                loader: weeklyMenuLoader,
              },
              {
                path: 'ai-suggestions',
                element: <AiSuggestionsPage />,
              },
            ],
          },
          
          // Family management
          {
            path: 'family',
            children: [
              {
                index: true,
                element: <FamilyOverviewPage />,
                loader: familyLoader,
              },
              {
                path: 'settings',
                element: <FamilySettingsPage />,
                loader: familyLoader,
              },
              {
                path: 'members/:memberId',
                element: <FamilyMemberDetailPage />,
                loader: familyMemberLoader,
              },
            ],
          },
          
          // Shopping lists
          {
            path: 'shopping',
            children: [
              {
                index: true,
                element: <ShoppingListsPage />,
                loader: shoppingListsLoader,
              },
              {
                path: ':listId',
                element: <ShoppingListDetailPage />,
                loader: shoppingListLoader,
              },
            ],
          },
          
          // User profile and settings
          {
            path: 'profile',
            element: <ProfilePage />,
            loader: profileLoader,
          },
        ],
      },
      
      // Error handling
      {
        path: '*',
        element: <NotFoundPage />,
      },
    ],
  },
]);

export const AppRouter: React.FC = () => (
  <AuthProvider>
    <Suspense fallback={<PageLoadingSpinner />}>
      <RouterProvider router={router} />
    </Suspense>
  </AuthProvider>
);
```

## Route Guards and Authentication

### Protected Route Component
```typescript
// components/auth/ProtectedRoute.tsx
import { useAuth } from '@/contexts/AuthContext';
import { Navigate, useLocation } from 'react-router-dom';

interface ProtectedRouteProps {
  children: React.ReactNode;
  requiredRole?: string;
  fallbackPath?: string;
}

export const ProtectedRoute: React.FC<ProtectedRouteProps> = ({
  children,
  requiredRole,
  fallbackPath = '/auth/login',
}) => {
  const { isAuthenticated, user, isLoading } = useAuth();
  const location = useLocation();

  if (isLoading) {
    return <PageLoadingSpinner />;
  }

  if (!isAuthenticated) {
    // Redirect to login with return path
    return (
      <Navigate
        to={fallbackPath}
        state={{ from: location.pathname }}
        replace
      />
    );
  }

  if (requiredRole && user?.role !== requiredRole) {
    // Redirect to unauthorized page
    return <Navigate to="/unauthorized" replace />;
  }

  return <>{children}</>;
};

// Public route component (redirects authenticated users)
export const PublicRoute: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const { isAuthenticated, isLoading } = useAuth();

  if (isLoading) {
    return <PageLoadingSpinner />;
  }

  if (isAuthenticated) {
    return <Navigate to="/dashboard" replace />;
  }

  return <>{children}</>;
};
```

### Role-Based Access Control
```typescript
// hooks/useRoleAccess.ts
import { useAuth } from '@/contexts/AuthContext';
import { useMemo } from 'react';

export type UserRole = 'admin' | 'premium' | 'basic' | 'free';
export type Permission = 
  | 'view_recipes' 
  | 'create_recipes' 
  | 'edit_recipes'
  | 'delete_recipes'
  | 'ai_suggestions'
  | 'menu_planning'
  | 'family_management'
  | 'analytics'
  | 'export_data';

const rolePermissions: Record<UserRole, Permission[]> = {
  admin: [
    'view_recipes', 'create_recipes', 'edit_recipes', 'delete_recipes',
    'ai_suggestions', 'menu_planning', 'family_management', 'analytics', 'export_data'
  ],
  premium: [
    'view_recipes', 'create_recipes', 'edit_recipes', 'delete_recipes',
    'ai_suggestions', 'menu_planning', 'family_management', 'export_data'
  ],
  basic: [
    'view_recipes', 'create_recipes', 'edit_recipes',
    'ai_suggestions', 'menu_planning', 'family_management'
  ],
  free: [
    'view_recipes', 'create_recipes', 'menu_planning', 'family_management'
  ],
};

export const useRoleAccess = () => {
  const { user } = useAuth();
  
  const permissions = useMemo(() => {
    if (!user?.role) return [];
    return rolePermissions[user.role as UserRole] || [];
  }, [user?.role]);

  const hasPermission = (permission: Permission): boolean => {
    return permissions.includes(permission);
  };

  const hasAnyPermission = (requiredPermissions: Permission[]): boolean => {
    return requiredPermissions.some(permission => permissions.includes(permission));
  };

  const hasAllPermissions = (requiredPermissions: Permission[]): boolean => {
    return requiredPermissions.every(permission => permissions.includes(permission));
  };

  return {
    permissions,
    hasPermission,
    hasAnyPermission,
    hasAllPermissions,
    userRole: user?.role as UserRole,
  };
};

// Permission-based component wrapper
export const WithPermission: React.FC<{
  permission: Permission | Permission[];
  children: React.ReactNode;
  fallback?: React.ReactNode;
}> = ({ permission, children, fallback = null }) => {
  const { hasPermission, hasAnyPermission } = useRoleAccess();
  
  const hasAccess = Array.isArray(permission) 
    ? hasAnyPermission(permission)
    : hasPermission(permission);

  return hasAccess ? <>{children}</> : <>{fallback}</>;
};
```

## Layout Components

### Root Layout
```typescript
// layouts/RootLayout.tsx
import { Outlet } from 'react-router-dom';
import { useUIStore } from '@/stores/uiStore';
import { cn } from '@/lib/utils';

export const RootLayout: React.FC = () => {
  const { theme } = useUIStore();

  return (
    <div className={cn('min-h-screen bg-background font-sans antialiased', theme)}>
      <Outlet />
    </div>
  );
};
```

### Dashboard Layout
```typescript
// layouts/DashboardLayout.tsx
import { Outlet, useLocation } from 'react-router-dom';
import { Sidebar } from '@/components/navigation/Sidebar';
import { Header } from '@/components/navigation/Header';
import { Breadcrumbs } from '@/components/navigation/Breadcrumbs';
import { useUIStore } from '@/stores/uiStore';

export const DashboardLayout: React.FC = () => {
  const { sidebarCollapsed } = useUIStore();
  const location = useLocation();

  return (
    <div className="flex h-screen bg-gray-50">
      {/* Sidebar */}
      <Sidebar />
      
      {/* Main content area */}
      <div className={cn(
        'flex-1 flex flex-col overflow-hidden transition-all duration-300',
        sidebarCollapsed ? 'ml-16' : 'ml-64'
      )}>
        {/* Header */}
        <Header />
        
        {/* Breadcrumbs */}
        <div className="px-6 py-4 border-b bg-white">
          <Breadcrumbs />
        </div>
        
        {/* Page content */}
        <main className="flex-1 overflow-auto p-6">
          <div className="max-w-7xl mx-auto">
            <Outlet />
          </div>
        </main>
      </div>
    </div>
  );
};
```

## Data Loaders

### Recipe Loaders
```typescript
// loaders/recipeLoaders.ts
import { QueryClient } from '@tanstack/react-query';
import { LoaderFunctionArgs } from 'react-router-dom';
import { recipeApi } from '@/api/recipeApi';

export const recipesLoader = (queryClient: QueryClient) => 
  async ({ request }: LoaderFunctionArgs) => {
    const url = new URL(request.url);
    const filters = {
      search: url.searchParams.get('search') || '',
      cuisine: url.searchParams.get('cuisine') || '',
      difficulty: url.searchParams.get('difficulty') || '',
      page: parseInt(url.searchParams.get('page') || '1'),
    };

    // Prefetch recipes data
    await queryClient.prefetchQuery({
      queryKey: ['recipes', filters],
      queryFn: () => recipeApi.getRecipes(filters),
      staleTime: 5 * 60 * 1000, // 5 minutes
    });

    return { filters };
  };

export const recipeDetailLoader = (queryClient: QueryClient) =>
  async ({ params }: LoaderFunctionArgs) => {
    const { recipeId } = params;
    
    if (!recipeId) {
      throw new Response('Recipe ID is required', { status: 400 });
    }

    try {
      // Prefetch recipe details
      await queryClient.prefetchQuery({
        queryKey: ['recipe', recipeId],
        queryFn: () => recipeApi.getRecipe(recipeId),
        staleTime: 10 * 60 * 1000, // 10 minutes
      });

      // Prefetch related recipes
      await queryClient.prefetchQuery({
        queryKey: ['recipes', 'related', recipeId],
        queryFn: () => recipeApi.getRelatedRecipes(recipeId),
        staleTime: 15 * 60 * 1000, // 15 minutes
      });

      return { recipeId };
    } catch (error) {
      throw new Response('Recipe not found', { status: 404 });
    }
  };
```

### Menu Planning Loaders
```typescript
// loaders/menuPlanningLoaders.ts
export const menuPlanningLoader = (queryClient: QueryClient) =>
  async ({ request }: LoaderFunctionArgs) => {
    const url = new URL(request.url);
    const weekStart = url.searchParams.get('week') || format(startOfWeek(new Date()), 'yyyy-MM-dd');

    // Prefetch current week's menu plan
    await queryClient.prefetchQuery({
      queryKey: ['menuPlan', weekStart],
      queryFn: () => menuPlanApi.getWeeklyMenuPlan(weekStart),
      staleTime: 2 * 60 * 1000, // 2 minutes
    });

    // Prefetch family data for meal planning
    await queryClient.prefetchQuery({
      queryKey: ['family'],
      queryFn: () => familyApi.getFamily(),
      staleTime: 10 * 60 * 1000, // 10 minutes
    });

    return { weekStart };
  };

export const weeklyMenuLoader = (queryClient: QueryClient) =>
  async ({ params }: LoaderFunctionArgs) => {
    const { weekStart } = params;
    
    if (!weekStart || !isValid(parseISO(weekStart))) {
      throw new Response('Invalid week start date', { status: 400 });
    }

    try {
      // Prefetch specific week's menu plan
      await queryClient.prefetchQuery({
        queryKey: ['menuPlan', weekStart],
        queryFn: () => menuPlanApi.getWeeklyMenuPlan(weekStart),
      });

      // Prefetch shopping list for the week
      await queryClient.prefetchQuery({
        queryKey: ['shoppingList', weekStart],
        queryFn: () => shoppingListApi.getWeeklyShoppingList(weekStart),
      });

      return { weekStart };
    } catch (error) {
      throw new Response('Menu plan not found', { status: 404 });
    }
  };
```

## Navigation Components

### Sidebar Navigation
```typescript
// components/navigation/Sidebar.tsx
import { NavLink, useLocation } from 'react-router-dom';
import { useUIStore } from '@/stores/uiStore';
import { useRoleAccess } from '@/hooks/useRoleAccess';

interface NavItem {
  path: string;
  label: string;
  icon: React.ComponentType<{ className?: string }>;
  permission?: Permission;
  badge?: string | number;
}

const navigationItems: NavItem[] = [
  {
    path: '/dashboard',
    label: 'Dashboard',
    icon: HomeIcon,
  },
  {
    path: '/dashboard/recipes',
    label: 'Recipes',
    icon: BookOpenIcon,
    permission: 'view_recipes',
  },
  {
    path: '/dashboard/menu-planning',
    label: 'Menu Planning',
    icon: CalendarIcon,
    permission: 'menu_planning',
  },
  {
    path: '/dashboard/family',
    label: 'Family',
    icon: UsersIcon,
    permission: 'family_management',
  },
  {
    path: '/dashboard/shopping',
    label: 'Shopping Lists',
    icon: ShoppingCartIcon,
  },
];

export const Sidebar: React.FC = () => {
  const { sidebarCollapsed, toggleSidebar } = useUIStore();
  const { hasPermission } = useRoleAccess();
  const location = useLocation();

  const isActiveRoute = (path: string) => {
    if (path === '/dashboard') {
      return location.pathname === '/dashboard';
    }
    return location.pathname.startsWith(path);
  };

  return (
    <div className={cn(
      'fixed left-0 top-0 z-40 h-full bg-white border-r border-gray-200 transition-all duration-300',
      sidebarCollapsed ? 'w-16' : 'w-64'
    )}>
      {/* Logo */}
      <div className="flex items-center justify-between p-4 border-b">
        {!sidebarCollapsed && (
          <Link to="/dashboard" className="flex items-center space-x-2">
            <ChefHatIcon className="w-8 h-8 text-primary-600" />
            <span className="text-xl font-bold text-gray-900">MealPrep</span>
          </Link>
        )}
        <button
          onClick={toggleSidebar}
          className="p-2 rounded-lg hover:bg-gray-100 transition-colors"
        >
          <MenuIcon className="w-5 h-5" />
        </button>
      </div>

      {/* Navigation */}
      <nav className="mt-6">
        <ul className="space-y-1 px-3">
          {navigationItems.map((item) => {
            // Check permissions
            if (item.permission && !hasPermission(item.permission)) {
              return null;
            }

            const isActive = isActiveRoute(item.path);
            const Icon = item.icon;

            return (
              <li key={item.path}>
                <NavLink
                  to={item.path}
                  className={cn(
                    'flex items-center space-x-3 px-3 py-2 rounded-lg transition-colors',
                    isActive
                      ? 'bg-primary-100 text-primary-700'
                      : 'text-gray-600 hover:bg-gray-100 hover:text-gray-900'
                  )}
                  title={sidebarCollapsed ? item.label : undefined}
                >
                  <Icon className="w-5 h-5 flex-shrink-0" />
                  {!sidebarCollapsed && (
                    <>
                      <span className="flex-1 font-medium">{item.label}</span>
                      {item.badge && (
                        <span className="bg-primary-100 text-primary-700 text-xs font-medium px-2 py-1 rounded-full">
                          {item.badge}
                        </span>
                      )}
                    </>
                  )}
                </NavLink>
              </li>
            );
          })}
        </ul>
      </nav>
    </div>
  );
};
```

### Breadcrumb Navigation
```typescript
// components/navigation/Breadcrumbs.tsx
import { useLocation, Link } from 'react-router-dom';
import { ChevronRightIcon } from '@heroicons/react/20/solid';

interface BreadcrumbItem {
  label: string;
  path?: string;
}

const routeLabels: Record<string, string> = {
  dashboard: 'Dashboard',
  recipes: 'Recipes',
  'menu-planning': 'Menu Planning',
  family: 'Family',
  shopping: 'Shopping Lists',
  profile: 'Profile',
  settings: 'Settings',
  new: 'New',
  edit: 'Edit',
};

export const Breadcrumbs: React.FC = () => {
  const location = useLocation();
  
  const generateBreadcrumbs = (): BreadcrumbItem[] => {
    const pathSegments = location.pathname.split('/').filter(Boolean);
    const breadcrumbs: BreadcrumbItem[] = [];
    
    let currentPath = '';
    
    pathSegments.forEach((segment, index) => {
      currentPath += `/${segment}`;
      
      // Skip the first 'dashboard' segment for cleaner breadcrumbs
      if (segment === 'dashboard' && index === 0) {
        return;
      }
      
      const label = routeLabels[segment] || segment;
      const isLast = index === pathSegments.length - 1;
      
      breadcrumbs.push({
        label,
        path: isLast ? undefined : currentPath,
      });
    });
    
    return breadcrumbs;
  };

  const breadcrumbs = generateBreadcrumbs();

  if (breadcrumbs.length <= 1) {
    return null;
  }

  return (
    <nav className="flex" aria-label="Breadcrumb">
      <ol className="flex items-center space-x-4">
        <li>
          <Link
            to="/dashboard"
            className="text-gray-500 hover:text-gray-700 transition-colors"
          >
            Dashboard
          </Link>
        </li>
        
        {breadcrumbs.map((crumb, index) => (
          <li key={index} className="flex items-center">
            <ChevronRightIcon className="w-5 h-5 text-gray-400 mx-2" />
            {crumb.path ? (
              <Link
                to={crumb.path}
                className="text-gray-500 hover:text-gray-700 transition-colors"
              >
                {crumb.label}
              </Link>
            ) : (
              <span className="text-gray-900 font-medium">{crumb.label}</span>
            )}
          </li>
        ))}
      </ol>
    </nav>
  );
};
```

## Performance Optimizations

### Route-Based Code Splitting
```typescript
// utils/lazyRoute.ts
import { lazy, ComponentType } from 'react';

interface LazyRouteOptions {
  fallback?: React.ComponentType;
  preload?: boolean;
}

export const lazyRoute = <T extends ComponentType<any>>(
  importFn: () => Promise<{ default: T }>,
  options: LazyRouteOptions = {}
) => {
  const LazyComponent = lazy(importFn);
  
  // Preload component if specified
  if (options.preload) {
    importFn();
  }
  
  return LazyComponent;
};

// Preload critical routes
export const preloadCriticalRoutes = () => {
  // Preload dashboard and recipes as they're commonly accessed
  import('@/pages/dashboard/DashboardPage');
  import('@/pages/recipes/RecipesPage');
};
```

### Route Transition Animation
```typescript
// components/routing/RouteTransition.tsx
import { motion, AnimatePresence } from 'framer-motion';
import { useLocation } from 'react-router-dom';

export const RouteTransition: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const location = useLocation();

  return (
    <AnimatePresence mode="wait">
      <motion.div
        key={location.pathname}
        initial={{ opacity: 0, y: 20 }}
        animate={{ opacity: 1, y: 0 }}
        exit={{ opacity: 0, y: -20 }}
        transition={{ duration: 0.2 }}
      >
        {children}
      </motion.div>
    </AnimatePresence>
  );
};
```

This comprehensive routing structure provides a scalable, secure, and performant navigation system for the MealPrep application with proper authentication guards, role-based access control, and performance optimizations.

*This routing guide should be updated as new routes are added and navigation patterns evolve.*