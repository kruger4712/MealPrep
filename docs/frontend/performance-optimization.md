# Frontend Performance Optimization

## Overview
Comprehensive performance optimization guide for the MealPrep React application, covering code splitting, lazy loading, caching strategies, bundle optimization, and runtime performance monitoring.

## Performance Architecture

### Performance Goals
```typescript
// Performance targets for MealPrep
const PERFORMANCE_TARGETS = {
  // Core Web Vitals
  largestContentfulPaint: 2.5, // seconds
  firstInputDelay: 100, // milliseconds
  cumulativeLayoutShift: 0.1, // score
  
  // Custom metrics
  timeToInteractive: 3.5, // seconds
  firstContentfulPaint: 1.8, // seconds
  bundleSize: {
    initial: 250, // KB (gzipped)
    total: 1000, // KB (gzipped)
  },
  
  // API response times
  apiResponseTime: 200, // milliseconds (average)
  aiSuggestionTime: 3000, // milliseconds (AI calls are slower)
  
  // User experience
  navigationTime: 100, // milliseconds (route changes)
  searchResponseTime: 300, // milliseconds
} as const;
```

## Code Splitting and Lazy Loading

### Route-Based Code Splitting
```typescript
// router/lazyRoutes.ts
import { lazy } from 'react';
import { LoadingSpinner } from '@/components/ui/LoadingSpinner';

// Lazy load route components with loading states
export const HomePage = lazy(() => 
  import('@/pages/HomePage').then(module => ({
    default: module.HomePage
  }))
);

export const DashboardPage = lazy(() => 
  import('@/pages/dashboard/DashboardPage')
);

export const RecipesPage = lazy(() => 
  import('@/pages/recipes/RecipesPage')
);

export const MenuPlanningPage = lazy(() => 
  import('@/pages/menu-planning/MenuPlanningPage')
);

// AI-heavy pages - separate chunk for better caching
export const AiSuggestionsPage = lazy(() => 
  import('@/pages/ai/AiSuggestionsPage')
);

// Route wrapper with optimized loading
export const LazyRoute: React.FC<{
  Component: React.LazyExoticComponent<React.ComponentType>;
  fallback?: React.ReactNode;
}> = ({ Component, fallback = <LoadingSpinner /> }) => (
  <Suspense fallback={fallback}>
    <Component />
  </Suspense>
);
```

### Component-Level Code Splitting
```typescript
// components/RecipeCard/index.tsx
import { lazy, useState } from 'react';
import { RecipeCardSkeleton } from './RecipeCardSkeleton';

// Lazy load expensive components
const RecipeDetails = lazy(() => import('./RecipeDetails'));
const RecipeNutrition = lazy(() => import('./RecipeNutrition'));
const RecipeReviews = lazy(() => import('./RecipeReviews'));

export const RecipeCard: React.FC<{ recipe: Recipe }> = ({ recipe }) => {
  const [showDetails, setShowDetails] = useState(false);
  const [activeTab, setActiveTab] = useState<'nutrition' | 'reviews' | null>(null);

  return (
    <Card className="recipe-card">
      {/* Basic recipe info - always loaded */}
      <RecipeCardHeader recipe={recipe} />
      <RecipeCardContent recipe={recipe} />
      
      {/* Conditionally load expensive components */}
      {showDetails && (
        <Suspense fallback={<RecipeCardSkeleton />}>
          <RecipeDetails recipe={recipe} />
        </Suspense>
      )}
      
      {activeTab === 'nutrition' && (
        <Suspense fallback={<div className="animate-pulse h-32" />}>
          <RecipeNutrition recipeId={recipe.id} />
        </Suspense>
      )}
      
      {activeTab === 'reviews' && (
        <Suspense fallback={<div className="animate-pulse h-24" />}>
          <RecipeReviews recipeId={recipe.id} />
        </Suspense>
      )}
      
      <RecipeCardActions 
        onShowDetails={() => setShowDetails(true)}
        onShowNutrition={() => setActiveTab('nutrition')}
        onShowReviews={() => setActiveTab('reviews')}
      />
    </Card>
  );
};
```

### Dynamic Imports for Features
```typescript
// utils/dynamicFeatures.ts
export const loadFeatureModule = async <T>(
  moduleLoader: () => Promise<{ default: T }>,
  fallback?: T
): Promise<T> => {
  try {
    const module = await moduleLoader();
    return module.default;
  } catch (error) {
    console.error('Failed to load feature module:', error);
    if (fallback) return fallback;
    throw error;
  }
};

// Usage in components
export const AiMealSuggestions: React.FC = () => {
  const [AiEngine, setAiEngine] = useState<any>(null);
  const [isLoading, setIsLoading] = useState(false);

  const loadAiEngine = async () => {
    setIsLoading(true);
    try {
      const engine = await loadFeatureModule(
        () => import('@/features/ai/MealSuggestionEngine'),
        null // No fallback for AI features
      );
      setAiEngine(engine);
    } catch (error) {
      // Handle AI module loading failure
      showNotification('AI features temporarily unavailable', 'warning');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div>
      {!AiEngine && (
        <button onClick={loadAiEngine} disabled={isLoading}>
          {isLoading ? 'Loading AI Features...' : 'Enable AI Suggestions'}
        </button>
      )}
      {AiEngine && <AiEngine />}
    </div>
  );
};
```

## React Query Optimization

### Smart Caching Strategies
```typescript
// api/optimizedQueries.ts
import { useQuery, useInfiniteQuery, QueryClient } from '@tanstack/react-query';

// Optimized recipe queries with stale-while-revalidate
export const useOptimizedRecipes = (filters: RecipeFilters) => {
  return useQuery({
    queryKey: ['recipes', 'optimized', filters],
    queryFn: () => recipeApi.getRecipes(filters),
    staleTime: 5 * 60 * 1000, // 5 minutes
    gcTime: 30 * 60 * 1000, // 30 minutes
    refetchOnWindowFocus: false,
    refetchOnMount: false,
    select: useCallback((data: Recipe[]) => {
      // Optimize data structure for rendering
      return data.map(recipe => ({
        ...recipe,
        // Pre-calculate commonly used values
        totalTime: recipe.prepTime + recipe.cookTime,
        costPerServing: recipe.estimatedCost / recipe.servings,
        // Only include essential fields for list view
        description: recipe.description?.substring(0, 150) + '...',
      }));
    }, []),
  });
};

// Infinite scroll with optimized pagination
export const useInfiniteRecipesOptimized = (filters: RecipeFilters) => {
  return useInfiniteQuery({
    queryKey: ['recipes', 'infinite', filters],
    queryFn: ({ pageParam = 1 }) => 
      recipeApi.getRecipes({ ...filters, page: pageParam, limit: 20 }),
    initialPageParam: 1,
    getNextPageParam: (lastPage, allPages) => {
      return lastPage.hasMore ? allPages.length + 1 : undefined;
    },
    staleTime: 10 * 60 * 1000, // 10 minutes for infinite queries
    select: useCallback((data) => ({
      pages: data.pages,
      pageParams: data.pageParams,
      recipes: data.pages.flatMap(page => page.recipes),
      totalCount: data.pages[0]?.totalCount || 0,
    }), []),
    // Performance optimization for large lists
    maxPages: 10, // Limit memory usage
  });
};

// Prefetch strategies
export const usePrefetchOptimizations = () => {
  const queryClient = useQueryClient();

  const prefetchRecipeDetails = useCallback((recipeId: string) => {
    queryClient.prefetchQuery({
      queryKey: ['recipe', recipeId],
      queryFn: () => recipeApi.getRecipe(recipeId),
      staleTime: 15 * 60 * 1000, // 15 minutes
    });
  }, [queryClient]);

  const prefetchUserData = useCallback((userId: string) => {
    // Prefetch user-related data on app start
    queryClient.prefetchQuery({
      queryKey: ['user', userId],
      queryFn: () => userApi.getUser(userId),
      staleTime: 60 * 60 * 1000, // 1 hour
    });

    queryClient.prefetchQuery({
      queryKey: ['family', userId],
      queryFn: () => familyApi.getFamily(userId),
      staleTime: 60 * 60 * 1000, // 1 hour
    });
  }, [queryClient]);

  return { prefetchRecipeDetails, prefetchUserData };
};
```

### Background Sync and Optimistic Updates
```typescript
// hooks/useOptimisticMutations.ts
export const useOptimisticRecipeUpdate = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (updates: Partial<Recipe> & { id: string }) => 
      recipeApi.updateRecipe(updates.id, updates),
    
    onMutate: async (updates) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['recipe', updates.id] });
      
      // Snapshot previous value
      const previousRecipe = queryClient.getQueryData(['recipe', updates.id]);
      
      // Optimistically update cache
      queryClient.setQueryData(['recipe', updates.id], (old: Recipe) => ({
        ...old,
        ...updates,
        updatedAt: new Date().toISOString(),
      }));

      // Update recipe lists
      queryClient.setQueriesData(
        { queryKey: ['recipes'], type: 'active' },
        (oldData: any) => {
          if (!oldData) return oldData;
          
          return {
            ...oldData,
            recipes: oldData.recipes?.map((recipe: Recipe) =>
              recipe.id === updates.id ? { ...recipe, ...updates } : recipe
            ),
          };
        }
      );
      
      return { previousRecipe };
    },
    
    onError: (err, updates, context) => {
      // Rollback on error
      if (context?.previousRecipe) {
        queryClient.setQueryData(['recipe', updates.id], context.previousRecipe);
      }
    },
    
    onSettled: (data, error, updates) => {
      // Always refetch after mutation
      queryClient.invalidateQueries({ queryKey: ['recipe', updates.id] });
    },
  });
};
```

## Image Optimization

### Progressive Image Loading
```typescript
// components/OptimizedImage.tsx
import { useState, useRef, useEffect } from 'react';
import { cn } from '@/lib/utils';

interface OptimizedImageProps {
  src: string;
  alt: string;
  className?: string;
  sizes?: string;
  priority?: boolean;
  placeholder?: 'blur' | 'empty';
  blurDataURL?: string;
}

export const OptimizedImage: React.FC<OptimizedImageProps> = ({
  src,
  alt,
  className,
  sizes = '(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw',
  priority = false,
  placeholder = 'blur',
  blurDataURL,
}) => {
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(false);
  const [currentSrc, setCurrentSrc] = useState(blurDataURL || '');
  const imgRef = useRef<HTMLImageElement>(null);

  // Generate responsive image URLs
  const generateSrcSet = (baseSrc: string) => {
    const widths = [320, 640, 768, 1024, 1280, 1920];
    return widths
      .map(width => `${baseSrc}?w=${width}&q=75 ${width}w`)
      .join(', ');
  };

  useEffect(() => {
    if (!src) return;

    const img = new Image();
    img.onload = () => {
      setCurrentSrc(src);
      setIsLoading(false);
    };
    img.onerror = () => {
      setError(true);
      setIsLoading(false);
    };
    
    // Load image with timeout
    const timeoutId = setTimeout(() => {
      if (isLoading) {
        setError(true);
        setIsLoading(false);
      }
    }, 10000); // 10 second timeout

    img.src = src;

    return () => clearTimeout(timeoutId);
  }, [src, isLoading]);

  // Intersection Observer for lazy loading
  useEffect(() => {
    if (priority || !imgRef.current) return;

    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            // Start loading when image enters viewport
            observer.disconnect();
          }
        });
      },
      { threshold: 0.1, rootMargin: '50px' }
    );

    observer.observe(imgRef.current);
    return () => observer.disconnect();
  }, [priority]);

  if (error) {
    return (
      <div className={cn(
        'flex items-center justify-center bg-gray-100 text-gray-400',
        className
      )}>
        <ImageIcon className="w-8 h-8" />
      </div>
    );
  }

  return (
    <div className={cn('relative overflow-hidden', className)}>
      <img
        ref={imgRef}
        src={currentSrc}
        srcSet={generateSrcSet(src)}
        sizes={sizes}
        alt={alt}
        className={cn(
          'w-full h-full object-cover transition-opacity duration-300',
          isLoading ? 'opacity-0' : 'opacity-100'
        )}
        loading={priority ? 'eager' : 'lazy'}
        decoding="async"
      />
      
      {/* Loading skeleton */}
      {isLoading && placeholder === 'blur' && (
        <div className="absolute inset-0 bg-gray-200 animate-pulse" />
      )}
      
      {/* Loading indicator */}
      {isLoading && (
        <div className="absolute inset-0 flex items-center justify-center">
          <LoadingSpinner size="sm" />
        </div>
      )}
    </div>
  );
};

// Image preloading utility
export const preloadImages = (urls: string[]) => {
  urls.forEach(url => {
    const link = document.createElement('link');
    link.rel = 'preload';
    link.as = 'image';
    link.href = url;
    document.head.appendChild(link);
  });
};
```

### Image CDN Integration
```typescript
// utils/imageOptimization.ts
export interface ImageTransformOptions {
  width?: number;
  height?: number;
  quality?: number;
  format?: 'webp' | 'avif' | 'jpeg' | 'png';
  fit?: 'cover' | 'contain' | 'fill';
  blur?: number;
}

export const optimizeImageUrl = (
  originalUrl: string,
  options: ImageTransformOptions = {}
): string => {
  const {
    width,
    height,
    quality = 80,
    format = 'webp',
    fit = 'cover',
    blur,
  } = options;

  // Use a CDN service like Cloudinary, ImageKit, or custom solution
  const baseUrl = process.env.VITE_IMAGE_CDN_URL || originalUrl;
  
  const params = new URLSearchParams();
  if (width) params.set('w', width.toString());
  if (height) params.set('h', height.toString());
  if (quality) params.set('q', quality.toString());
  if (format) params.set('f', format);
  if (fit) params.set('fit', fit);
  if (blur) params.set('blur', blur.toString());

  return `${baseUrl}?${params.toString()}`;
};

// Hook for responsive images
export const useResponsiveImage = (
  src: string,
  options: ImageTransformOptions = {}
) => {
  const { isMobile, isTablet, isDesktop } = useResponsive();

  return useMemo(() => {
    if (isMobile) {
      return optimizeImageUrl(src, { ...options, width: 400, quality: 70 });
    }
    if (isTablet) {
      return optimizeImageUrl(src, { ...options, width: 600, quality: 75 });
    }
    if (isDesktop) {
      return optimizeImageUrl(src, { ...options, width: 800, quality: 80 });
    }
    return src;
  }, [src, options, isMobile, isTablet, isDesktop]);
};
```

## Bundle Optimization

### Webpack Bundle Analysis
```typescript
// scripts/analyze-bundle.ts
import { BundleAnalyzerPlugin } from 'webpack-bundle-analyzer';

// Add to vite.config.ts
export default defineConfig({
  plugins: [
    // Bundle analyzer for development
    process.env.ANALYZE && new BundleAnalyzerPlugin({
      analyzerMode: 'server',
      openAnalyzer: true,
    }),
  ],
  
  build: {
    // Code splitting strategy
    rollupOptions: {
      output: {
        manualChunks: {
          // Vendor chunks
          'react-vendor': ['react', 'react-dom'],
          'ui-vendor': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
          'query-vendor': ['@tanstack/react-query'],
          'router-vendor': ['react-router-dom'],
          
          // Feature chunks
          'ai-features': [
            'src/features/ai',
            'src/services/aiService',
          ],
          'recipe-features': [
            'src/features/recipes',
            'src/components/recipe',
          ],
          'menu-features': [
            'src/features/menu-planning',
            'src/components/menu',
          ],
        },
      },
    },
    
    // Optimization settings
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true, // Remove console.logs in production
        drop_debugger: true,
        pure_funcs: ['console.log', 'console.info'], // Remove specific functions
      },
    },
    
    // Source map for production debugging
    sourcemap: 'hidden',
    
    // Asset optimization
    assetsInlineLimit: 4096, // Inline assets smaller than 4KB
  },
});
```

### Tree Shaking Optimization
```typescript
// utils/optimizedImports.ts
// Use named imports for better tree shaking
import { debounce, throttle } from 'lodash-es'; // ? Good
// import _ from 'lodash'; // ? Bad - imports entire library

// Optimize icon imports
import { 
  CalendarIcon, 
  ClockIcon, 
  UsersIcon 
} from '@heroicons/react/24/outline'; // ? Good

// Create barrel exports carefully
// index.ts - only export what's needed
export { Button } from './Button';
export { Input } from './Input';
export { Card } from './Card';
// Don't export everything from a directory

// Optimize date library imports
import { format, parseISO, startOfWeek } from 'date-fns'; // ? Good
// import * as dateFns from 'date-fns'; // ? Bad

// Use dynamic imports for large utilities
export const loadLargeUtility = () => 
  import('./largeUtility').then(module => module.default);
```

## Runtime Performance Monitoring

### Performance Metrics Collection
```typescript
// utils/performanceMonitoring.ts
export class PerformanceMonitor {
  private metrics: Map<string, number[]> = new Map();
  private observer?: PerformanceObserver;

  constructor() {
    this.initializeObserver();
    this.collectCoreWebVitals();
  }

  private initializeObserver() {
    if ('PerformanceObserver' in window) {
      this.observer = new PerformanceObserver((list) => {
        list.getEntries().forEach((entry) => {
          this.recordMetric(entry.name, entry.duration);
        });
      });

      this.observer.observe({ entryTypes: ['measure', 'navigation'] });
    }
  }

  private collectCoreWebVitals() {
    // Largest Contentful Paint
    new PerformanceObserver((list) => {
      const entries = list.getEntries();
      const lastEntry = entries[entries.length - 1];
      this.recordMetric('LCP', lastEntry.startTime);
    }).observe({ entryTypes: ['largest-contentful-paint'] });

    // First Input Delay
    new PerformanceObserver((list) => {
      list.getEntries().forEach((entry) => {
        if (entry.processingStart && entry.startTime) {
          const fid = entry.processingStart - entry.startTime;
          this.recordMetric('FID', fid);
        }
      });
    }).observe({ entryTypes: ['first-input'] });

    // Cumulative Layout Shift
    let cumulativeLayoutShift = 0;
    new PerformanceObserver((list) => {
      list.getEntries().forEach((entry) => {
        if (!entry.hadRecentInput) {
          cumulativeLayoutShift += entry.value;
        }
      });
      this.recordMetric('CLS', cumulativeLayoutShift);
    }).observe({ entryTypes: ['layout-shift'] });
  }

  recordMetric(name: string, value: number) {
    if (!this.metrics.has(name)) {
      this.metrics.set(name, []);
    }
    this.metrics.get(name)!.push(value);
  }

  markStart(name: string) {
    performance.mark(`${name}-start`);
  }

  markEnd(name: string) {
    performance.mark(`${name}-end`);
    performance.measure(name, `${name}-start`, `${name}-end`);
  }

  getMetrics() {
    const results: Record<string, any> = {};
    
    this.metrics.forEach((values, name) => {
      results[name] = {
        count: values.length,
        average: values.reduce((sum, val) => sum + val, 0) / values.length,
        min: Math.min(...values),
        max: Math.max(...values),
        p95: this.percentile(values, 95),
      };
    });

    return results;
  }

  private percentile(values: number[], p: number): number {
    const sorted = [...values].sort((a, b) => a - b);
    const index = (p / 100) * (sorted.length - 1);
    return sorted[Math.round(index)];
  }

  // Send metrics to analytics service
  async reportMetrics() {
    const metrics = this.getMetrics();
    
    try {
      await fetch('/api/analytics/performance', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          url: window.location.pathname,
          userAgent: navigator.userAgent,
          timestamp: Date.now(),
          metrics,
        }),
      });
    } catch (error) {
      console.warn('Failed to report performance metrics:', error);
    }
  }
}

// Global performance monitor instance
export const performanceMonitor = new PerformanceMonitor();

// React hook for component performance tracking
export const usePerformanceTracking = (componentName: string) => {
  useEffect(() => {
    performanceMonitor.markStart(componentName);
    
    return () => {
      performanceMonitor.markEnd(componentName);
    };
  }, [componentName]);
};
```

### Real User Monitoring (RUM)
```typescript
// components/PerformanceProvider.tsx
export const PerformanceProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  useEffect(() => {
    // Track route changes
    const handleRouteChange = () => {
      performanceMonitor.markStart('route-change');
      
      // Track route change completion
      requestIdleCallback(() => {
        performanceMonitor.markEnd('route-change');
      });
    };

    // Track user interactions
    const handleUserInteraction = (event: Event) => {
      performanceMonitor.markStart(`interaction-${event.type}`);
      
      requestAnimationFrame(() => {
        performanceMonitor.markEnd(`interaction-${event.type}`);
      });
    };

    // Add event listeners
    window.addEventListener('popstate', handleRouteChange);
    document.addEventListener('click', handleUserInteraction);
    document.addEventListener('keydown', handleUserInteraction);

    // Report metrics periodically
    const reportInterval = setInterval(() => {
      performanceMonitor.reportMetrics();
    }, 30000); // Every 30 seconds

    return () => {
      window.removeEventListener('popstate', handleRouteChange);
      document.removeEventListener('click', handleUserInteraction);
      document.removeEventListener('keydown', handleUserInteraction);
      clearInterval(reportInterval);
    };
  }, []);

  return <>{children}</>;
};
```

## Memory Management

### Component Memory Optimization
```typescript
// hooks/useMemoryOptimization.ts
export const useMemoryOptimization = () => {
  const abortControllerRef = useRef<AbortController>();
  const timeoutRefs = useRef<Set<NodeJS.Timeout>>(new Set());

  // Cleanup function
  const cleanup = useCallback(() => {
    // Cancel ongoing requests
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }

    // Clear timeouts
    timeoutRefs.current.forEach(timeout => clearTimeout(timeout));
    timeoutRefs.current.clear();
  }, []);

  // Auto cleanup on unmount
  useEffect(() => {
    return cleanup;
  }, [cleanup]);

  // Create abort controller for requests
  const createAbortController = useCallback(() => {
    cleanup();
    abortControllerRef.current = new AbortController();
    return abortControllerRef.current;
  }, [cleanup]);

  // Safe timeout that cleans up automatically
  const safeSetTimeout = useCallback((callback: () => void, delay: number) => {
    const timeout = setTimeout(() => {
      callback();
      timeoutRefs.current.delete(timeout);
    }, delay);
    
    timeoutRefs.current.add(timeout);
    return timeout;
  }, []);

  return {
    createAbortController,
    safeSetTimeout,
    cleanup,
  };
};

// Memory-efficient list virtualization
export const VirtualizedRecipeList: React.FC<{
  recipes: Recipe[];
  onRecipeClick: (recipe: Recipe) => void;
}> = ({ recipes, onRecipeClick }) => {
  const containerRef = useRef<HTMLDivElement>(null);
  const [visibleRange, setVisibleRange] = useState({ start: 0, end: 10 });
  
  const ITEM_HEIGHT = 200;
  const BUFFER_SIZE = 5;

  // Update visible range based on scroll position
  const handleScroll = useCallback(
    throttle(() => {
      if (!containerRef.current) return;
      
      const scrollTop = containerRef.current.scrollTop;
      const containerHeight = containerRef.current.clientHeight;
      
      const start = Math.max(0, Math.floor(scrollTop / ITEM_HEIGHT) - BUFFER_SIZE);
      const end = Math.min(
        recipes.length,
        Math.ceil((scrollTop + containerHeight) / ITEM_HEIGHT) + BUFFER_SIZE
      );
      
      setVisibleRange({ start, end });
    }, 16), // ~60fps
    [recipes.length]
  );

  const visibleRecipes = useMemo(() => 
    recipes.slice(visibleRange.start, visibleRange.end),
    [recipes, visibleRange]
  );

  return (
    <div
      ref={containerRef}
      className="h-96 overflow-auto"
      onScroll={handleScroll}
    >
      <div style={{ height: recipes.length * ITEM_HEIGHT, position: 'relative' }}>
        {visibleRecipes.map((recipe, index) => (
          <div
            key={recipe.id}
            style={{
              position: 'absolute',
              top: (visibleRange.start + index) * ITEM_HEIGHT,
              height: ITEM_HEIGHT,
              width: '100%',
            }}
          >
            <RecipeCard recipe={recipe} onClick={onRecipeClick} />
          </div>
        ))}
      </div>
    </div>
  );
};
```

This comprehensive performance optimization guide provides strategies for maximizing the speed, efficiency, and user experience of the MealPrep React application across all performance dimensions.

*This performance guide should be updated regularly as new optimization techniques emerge and application usage patterns evolve.*