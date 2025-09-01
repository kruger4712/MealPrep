# State Management

## Overview
Comprehensive guide to state management in the MealPrep React application using modern patterns including React Query, Zustand, and Context API for optimal performance and developer experience.

## State Management Architecture

### State Categories
```typescript
// Global Application State
interface AppState {
  // Authentication & User Data
  auth: AuthState;
  user: UserState;
  
  // Family & Member Data (Persistent)
  family: FamilyState;
  members: FamilyMembersState;
  
  // UI State (Session-based)
  ui: UIState;
  navigation: NavigationState;
  
  // Feature-specific State
  recipes: RecipeState;
  menuPlanning: MenuPlanningState;
  shoppingLists: ShoppingListState;
}

// Server State (React Query)
interface ServerState {
  // Cached API responses
  recipes: Recipe[];
  menuPlans: MenuPlan[];
  aiSuggestions: MealSuggestion[];
  familyPreferences: FamilyPreferences;
}

// Local Component State (useState/useReducer)
interface LocalState {
  // Form data
  formInputs: FormData;
  validation: ValidationState;
  
  // UI interactions
  modals: ModalState;
  dropdowns: DropdownState;
  loading: LoadingState;
}
```

### Technology Stack
- **React Query v5**: Server state management and caching
- **Zustand**: Lightweight client state management
- **React Context**: Cross-cutting concerns (theme, auth)
- **React Hook Form**: Form state management
- **Immer**: Immutable state updates

## React Query Configuration

### Query Client Setup
```typescript
// lib/react-query.ts
import { QueryClient } from '@tanstack/react-query';
import { persistQueryClient } from '@tanstack/react-query-persist-client-core';
import { createSyncStoragePersister } from '@tanstack/query-sync-storage-persister';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      gcTime: 10 * 60 * 1000, // 10 minutes (formerly cacheTime)
      retry: (failureCount, error) => {
        // Don't retry on 4xx errors
        if (error instanceof Error && error.message.includes('4')) {
          return false;
        }
        return failureCount < 3;
      },
      refetchOnWindowFocus: false,
      refetchOnReconnect: true,
    },
    mutations: {
      retry: 1,
      onError: (error) => {
        console.error('Mutation error:', error);
        // Global error handling
        toast.error('Something went wrong. Please try again.');
      },
    },
  },
});

// Persist queries to localStorage for offline support
const localStoragePersister = createSyncStoragePersister({
  storage: window.localStorage,
  key: 'MEALPREP_QUERIES',
});

persistQueryClient({
  queryClient,
  persister: localStoragePersister,
  maxAge: 24 * 60 * 60 * 1000, // 24 hours
  buster: process.env.REACT_APP_VERSION, // Clear cache on app updates
});
```

### Query Hooks for MealPrep Features

#### Recipe Queries
```typescript
// hooks/queries/useRecipes.ts
export const useRecipes = (filters?: RecipeFilters) => {
  return useQuery({
    queryKey: ['recipes', filters],
    queryFn: () => recipeApi.getRecipes(filters),
    staleTime: 10 * 60 * 1000, // Recipes don't change often
    select: (data) => {
      // Transform data if needed
      return data.map(recipe => ({
        ...recipe,
        totalTime: recipe.prepTime + recipe.cookTime,
        isBookmarked: bookmarkedRecipes.includes(recipe.id),
      }));
    },
  });
};

export const useRecipe = (recipeId: string) => {
  return useQuery({
    queryKey: ['recipe', recipeId],
    queryFn: () => recipeApi.getRecipe(recipeId),
    enabled: !!recipeId,
    staleTime: 30 * 60 * 1000, // Individual recipes are very stable
  });
};

export const useRecipeSearch = (query: string, options?: SearchOptions) => {
  return useQuery({
    queryKey: ['recipeSearch', query, options],
    queryFn: () => recipeApi.searchRecipes(query, options),
    enabled: query.length >= 2, // Only search with meaningful input
    staleTime: 2 * 60 * 1000, // Search results can change more frequently
    placeholderData: (previousData) => previousData, // Keep previous results while loading
  });
};

// Infinite query for recipe pagination
export const useInfiniteRecipes = (filters?: RecipeFilters) => {
  return useInfiniteQuery({
    queryKey: ['recipes', 'infinite', filters],
    queryFn: ({ pageParam = 1 }) => 
      recipeApi.getRecipes({ ...filters, page: pageParam }),
    initialPageParam: 1,
    getNextPageParam: (lastPage, allPages) => {
      return lastPage.hasMore ? allPages.length + 1 : undefined;
    },
    select: (data) => ({
      pages: data.pages,
      pageParams: data.pageParams,
      recipes: data.pages.flatMap(page => page.recipes),
    }),
  });
};
```

#### AI Suggestion Queries
```typescript
// hooks/queries/useAiSuggestions.ts
export const useMealSuggestions = (request: MealSuggestionRequest) => {
  return useQuery({
    queryKey: ['aiSuggestions', 'meals', request],
    queryFn: () => aiApi.getMealSuggestions(request),
    enabled: !!request.familyMembers?.length,
    staleTime: 30 * 60 * 1000, // AI suggestions are relatively stable
    retry: (failureCount, error) => {
      // Handle AI service failures gracefully
      if (error?.message?.includes('AI_SERVICE_UNAVAILABLE')) {
        return false; // Don't retry, use fallback
      }
      return failureCount < 2;
    },
    meta: {
      // Metadata for error handling
      fallbackStrategy: 'cached_suggestions',
      requiresAuth: true,
    },
  });
};

export const useWeeklyMenuGeneration = (request: WeeklyMenuRequest) => {
  return useMutation({
    mutationFn: (request: WeeklyMenuRequest) => aiApi.generateWeeklyMenu(request),
    onSuccess: (data) => {
      // Invalidate related queries
      queryClient.invalidateQueries({ queryKey: ['menuPlans'] });
      queryClient.setQueryData(['weeklyMenu', data.id], data);
      
      // Show success notification
      toast.success('Weekly menu generated successfully!');
    },
    onError: (error) => {
      console.error('Menu generation failed:', error);
      // Fallback to manual menu planning
      toast.error('AI menu generation failed. You can create a menu manually.');
    },
  });
};

// Optimistic updates for AI feedback
export const useMealRating = () => {
  return useMutation({
    mutationFn: (rating: MealRating) => aiApi.submitMealRating(rating),
    onMutate: async (newRating) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['mealRatings'] });
      
      // Snapshot previous value
      const previousRatings = queryClient.getQueryData(['mealRatings']);
      
      // Optimistically update cache
      queryClient.setQueryData(['mealRatings'], (old: MealRating[]) => [
        ...(old || []),
        { ...newRating, id: `temp-${Date.now()}`, submittedAt: new Date() }
      ]);
      
      return { previousRatings };
    },
    onError: (err, newRating, context) => {
      // Rollback on error
      queryClient.setQueryData(['mealRatings'], context?.previousRatings);
      toast.error('Failed to save rating. Please try again.');
    },
    onSettled: () => {
      // Always refetch after error or success
      queryClient.invalidateQueries({ queryKey: ['mealRatings'] });
    },
  });
};
```

#### Family and Member Queries
```typescript
// hooks/queries/useFamily.ts
export const useFamily = () => {
  return useQuery({
    queryKey: ['family'],
    queryFn: familyApi.getFamily,
    staleTime: 15 * 60 * 1000, // Family data changes infrequently
    select: (data) => ({
      ...data,
      memberCount: data.members.length,
      averageAge: data.members.reduce((sum, m) => sum + m.age, 0) / data.members.length,
      commonAllergens: getCommonAllergens(data.members),
      dietaryRestrictions: getCommonDietaryRestrictions(data.members),
    }),
  });
};

export const useFamilyMembers = () => {
  return useQuery({
    queryKey: ['familyMembers'],
    queryFn: familyApi.getFamilyMembers,
    staleTime: 15 * 60 * 1000,
    select: (members) => ({
      members,
      activeMembers: members.filter(m => m.isActive),
      children: members.filter(m => m.age < 18),
      adults: members.filter(m => m.age >= 18),
    }),
  });
};

// Mutation for updating family member preferences
export const useUpdateMemberPreferences = () => {
  return useMutation({
    mutationFn: ({ memberId, preferences }: { 
      memberId: string; 
      preferences: Partial<FamilyMemberPreferences> 
    }) => familyApi.updateMemberPreferences(memberId, preferences),
    onSuccess: (data, variables) => {
      // Update specific member in cache
      queryClient.setQueryData(['familyMembers'], (old: FamilyMember[]) =>
        old?.map(member => 
          member.id === variables.memberId 
            ? { ...member, preferences: { ...member.preferences, ...data } }
            : member
        )
      );
      
      // Invalidate AI suggestions since preferences changed
      queryClient.invalidateQueries({ queryKey: ['aiSuggestions'] });
      
      toast.success('Preferences updated successfully!');
    },
  });
};
```

## Zustand State Management

### Global UI State Store
```typescript
// stores/uiStore.ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface UIState {
  // Theme and appearance
  theme: 'light' | 'dark' | 'system';
  sidebarCollapsed: boolean;
  
  // Navigation and routing
  currentPage: string;
  navigationHistory: string[];
  
  // Modal and overlay state
  modals: {
    addRecipe: boolean;
    editFamily: boolean;
    mealPlanner: boolean;
    aiSuggestions: boolean;
  };
  
  // Loading states
  globalLoading: boolean;
  loadingStates: Record<string, boolean>;
  
  // Notifications
  notifications: Notification[];
  
  // Actions
  setTheme: (theme: UIState['theme']) => void;
  toggleSidebar: () => void;
  openModal: (modal: keyof UIState['modals']) => void;
  closeModal: (modal: keyof UIState['modals']) => void;
  closeAllModals: () => void;
  setLoading: (key: string, loading: boolean) => void;
  addNotification: (notification: Omit<Notification, 'id'>) => void;
  removeNotification: (id: string) => void;
  navigateTo: (page: string) => void;
}

export const useUIStore = create<UIState>()(
  devtools(
    persist(
      immer((set, get) => ({
        // Initial state
        theme: 'system',
        sidebarCollapsed: false,
        currentPage: '/',
        navigationHistory: [],
        modals: {
          addRecipe: false,
          editFamily: false,
          mealPlanner: false,
          aiSuggestions: false,
        },
        globalLoading: false,
        loadingStates: {},
        notifications: [],
        
        // Actions
        setTheme: (theme) => set((state) => {
          state.theme = theme;
        }),
        
        toggleSidebar: () => set((state) => {
          state.sidebarCollapsed = !state.sidebarCollapsed;
        }),
        
        openModal: (modal) => set((state) => {
          state.modals[modal] = true;
        }),
        
        closeModal: (modal) => set((state) => {
          state.modals[modal] = false;
        }),
        
        closeAllModals: () => set((state) => {
          Object.keys(state.modals).forEach(key => {
            state.modals[key as keyof typeof state.modals] = false;
          });
        }),
        
        setLoading: (key, loading) => set((state) => {
          if (loading) {
            state.loadingStates[key] = true;
          } else {
            delete state.loadingStates[key];
          }
          state.globalLoading = Object.keys(state.loadingStates).length > 0;
        }),
        
        addNotification: (notification) => set((state) => {
          const newNotification = {
            ...notification,
            id: `notification-${Date.now()}-${Math.random()}`,
            createdAt: new Date(),
          };
          state.notifications.push(newNotification);
        }),
        
        removeNotification: (id) => set((state) => {
          state.notifications = state.notifications.filter(n => n.id !== id);
        }),
        
        navigateTo: (page) => set((state) => {
          state.navigationHistory.push(state.currentPage);
          state.currentPage = page;
          
          // Keep only last 10 pages in history
          if (state.navigationHistory.length > 10) {
            state.navigationHistory = state.navigationHistory.slice(-10);
          }
        }),
      })),
      {
        name: 'mealprep-ui-state',
        partialize: (state) => ({ 
          theme: state.theme, 
          sidebarCollapsed: state.sidebarCollapsed 
        }),
      }
    ),
    { name: 'UI Store' }
  )
);
```

### Menu Planning State Store
```typescript
// stores/menuPlanningStore.ts
interface MenuPlanningState {
  // Current planning session
  currentMenuPlan: Partial<MenuPlan> | null;
  selectedWeek: Date;
  planningMode: 'ai' | 'manual' | 'hybrid';
  
  // Drag and drop state
  draggedMeal: PlannedMeal | null;
  dropZone: { date: string; mealType: string } | null;
  
  // Selection and editing
  selectedMeals: string[];
  editingMeal: string | null;
  
  // AI suggestion state
  aiSuggestionRequest: MealSuggestionRequest | null;
  pendingAiRequests: Set<string>;
  
  // Actions
  initializeMenuPlan: (startDate: Date) => void;
  updatePlannedMeal: (mealId: string, updates: Partial<PlannedMeal>) => void;
  addMealToDate: (date: string, mealType: string, recipe: Recipe) => void;
  removeMealFromDate: (date: string, mealType: string) => void;
  moveMeal: (mealId: string, fromDate: string, toDate: string, mealType: string) => void;
  toggleMealSelection: (mealId: string) => void;
  clearSelection: () => void;
  setDraggedMeal: (meal: PlannedMeal | null) => void;
  setDropZone: (zone: { date: string; mealType: string } | null) => void;
  requestAiSuggestions: (request: MealSuggestionRequest) => void;
  applyAiSuggestion: (suggestion: MealSuggestion, date: string, mealType: string) => void;
}

export const useMenuPlanningStore = create<MenuPlanningState>()(
  devtools(
    immer((set, get) => ({
      // Initial state
      currentMenuPlan: null,
      selectedWeek: startOfWeek(new Date()),
      planningMode: 'hybrid',
      draggedMeal: null,
      dropZone: null,
      selectedMeals: [],
      editingMeal: null,
      aiSuggestionRequest: null,
      pendingAiRequests: new Set(),
      
      // Actions
      initializeMenuPlan: (startDate) => set((state) => {
        state.selectedWeek = startOfWeek(startDate);
        state.currentMenuPlan = {
          id: `draft-${Date.now()}`,
          name: `Menu for ${format(startDate, 'MMM d, yyyy')}`,
          startDate: startOfWeek(startDate),
          endDate: endOfWeek(startDate),
          status: 'draft',
          plannedMeals: [],
        };
      }),
      
      updatePlannedMeal: (mealId, updates) => set((state) => {
        if (!state.currentMenuPlan?.plannedMeals) return;
        
        const mealIndex = state.currentMenuPlan.plannedMeals.findIndex(m => m.id === mealId);
        if (mealIndex !== -1) {
          Object.assign(state.currentMenuPlan.plannedMeals[mealIndex], updates);
        }
      }),
      
      addMealToDate: (date, mealType, recipe) => set((state) => {
        if (!state.currentMenuPlan) return;
        
        const newMeal: PlannedMeal = {
          id: `meal-${Date.now()}`,
          recipeId: recipe.id,
          recipe,
          plannedDate: date,
          mealType,
          servings: recipe.servings,
          status: 'planned',
          createdAt: new Date(),
        };
        
        if (!state.currentMenuPlan.plannedMeals) {
          state.currentMenuPlan.plannedMeals = [];
        }
        
        state.currentMenuPlan.plannedMeals.push(newMeal);
      }),
      
      moveMeal: (mealId, fromDate, toDate, mealType) => set((state) => {
        if (!state.currentMenuPlan?.plannedMeals) return;
        
        const meal = state.currentMenuPlan.plannedMeals.find(m => m.id === mealId);
        if (meal) {
          meal.plannedDate = toDate;
          meal.mealType = mealType;
        }
      }),
      
      requestAiSuggestions: (request) => set((state) => {
        state.aiSuggestionRequest = request;
        const requestKey = `${request.date}-${request.mealType}`;
        state.pendingAiRequests.add(requestKey);
      }),
      
      applyAiSuggestion: (suggestion, date, mealType) => set((state) => {
        // Remove any existing meal for this slot
        if (state.currentMenuPlan?.plannedMeals) {
          state.currentMenuPlan.plannedMeals = state.currentMenuPlan.plannedMeals
            .filter(m => !(m.plannedDate === date && m.mealType === mealType));
        }
        
        // Add the AI suggestion as a planned meal
        const newMeal: PlannedMeal = {
          id: `ai-meal-${Date.now()}`,
          recipeId: suggestion.id,
          recipe: suggestion,
          plannedDate: date,
          mealType,
          servings: suggestion.servings,
          status: 'planned',
          aiGenerated: true,
          familyFitScore: suggestion.familyFitScore,
          aiReasoning: suggestion.reasoning,
          createdAt: new Date(),
        };
        
        if (!state.currentMenuPlan?.plannedMeals) {
          state.currentMenuPlan.plannedMeals = [];
        }
        
        state.currentMenuPlan.plannedMeals.push(newMeal);
        
        // Clear the pending request
        const requestKey = `${date}-${mealType}`;
        state.pendingAiRequests.delete(requestKey);
      }),
    })),
    { name: 'Menu Planning Store' }
  )
);
```

## Context Providers

### Authentication Context
```typescript
// contexts/AuthContext.tsx
interface AuthContextType {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  register: (userData: RegisterData) => Promise<void>;
  logout: () => Promise<void>;
  refreshToken: () => Promise<void>;
  updateUser: (updates: Partial<User>) => Promise<void>;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  
  const queryClient = useQueryClient();

  // Initialize auth state on app load
  useEffect(() => {
    const initializeAuth = async () => {
      try {
        const token = localStorage.getItem('auth_token');
        if (token) {
          const userData = await authApi.validateToken(token);
          setUser(userData);
        }
      } catch (error) {
        console.error('Auth initialization failed:', error);
        localStorage.removeItem('auth_token');
      } finally {
        setIsLoading(false);
      }
    };

    initializeAuth();
  }, []);

  const login = async (email: string, password: string) => {
    setIsLoading(true);
    try {
      const { user, token } = await authApi.login(email, password);
      localStorage.setItem('auth_token', token);
      setUser(user);
      
      // Clear any cached data from previous user
      queryClient.clear();
    } catch (error) {
      console.error('Login failed:', error);
      throw error;
    } finally {
      setIsLoading(false);
    }
  };

  const logout = async () => {
    try {
      await authApi.logout();
    } catch (error) {
      console.error('Logout request failed:', error);
    } finally {
      localStorage.removeItem('auth_token');
      setUser(null);
      queryClient.clear();
      
      // Reset all Zustand stores
      useUIStore.getState().closeAllModals();
      useMenuPlanningStore.setState({
        currentMenuPlan: null,
        selectedMeals: [],
        aiSuggestionRequest: null,
      });
    }
  };

  const value = {
    user,
    isAuthenticated: !!user,
    isLoading,
    login,
    register: async (userData: RegisterData) => {
      const { user, token } = await authApi.register(userData);
      localStorage.setItem('auth_token', token);
      setUser(user);
    },
    logout,
    refreshToken: async () => {
      const { token } = await authApi.refreshToken();
      localStorage.setItem('auth_token', token);
    },
    updateUser: async (updates: Partial<User>) => {
      const updatedUser = await authApi.updateUser(updates);
      setUser(updatedUser);
    },
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
};
```

## Form State Management

### Recipe Form with React Hook Form
```typescript
// components/forms/RecipeForm.tsx
interface RecipeFormData {
  name: string;
  description: string;
  ingredients: IngredientInput[];
  instructions: string[];
  prepTime: number;
  cookTime: number;
  servings: number;
  difficulty: 'Easy' | 'Medium' | 'Hard';
  cuisine: string;
  tags: string[];
  image?: File;
}

export const RecipeForm: React.FC<{ 
  initialData?: Partial<Recipe>; 
  onSubmit: (data: RecipeFormData) => Promise<void>;
}> = ({ initialData, onSubmit }) => {
  const {
    register,
    control,
    handleSubmit,
    watch,
    setValue,
    formState: { errors, isSubmitting, isDirty },
    reset,
  } = useForm<RecipeFormData>({
    defaultValues: {
      name: initialData?.name || '',
      description: initialData?.description || '',
      ingredients: initialData?.ingredients || [{ name: '', quantity: 0, unit: '' }],
      instructions: initialData?.instructions || [''],
      prepTime: initialData?.prepTime || 0,
      cookTime: initialData?.cookTime || 0,
      servings: initialData?.servings || 4,
      difficulty: initialData?.difficulty || 'Medium',
      cuisine: initialData?.cuisine || '',
      tags: initialData?.tags || [],
    },
    resolver: zodResolver(recipeFormSchema),
  });

  const { fields: ingredientFields, append: appendIngredient, remove: removeIngredient } = 
    useFieldArray({ control, name: 'ingredients' });
  
  const { fields: instructionFields, append: appendInstruction, remove: removeInstruction } = 
    useFieldArray({ control, name: 'instructions' });

  // Auto-save draft functionality
  const watchedValues = watch();
  const { mutate: saveDraft } = useMutation({
    mutationFn: (data: Partial<RecipeFormData>) => recipeApi.saveDraft(data),
    onSuccess: () => {
      toast.success('Draft saved automatically');
    },
  });

  // Auto-save every 30 seconds if form is dirty
  useEffect(() => {
    if (!isDirty) return;

    const interval = setInterval(() => {
      saveDraft(watchedValues);
    }, 30000);

    return () => clearInterval(interval);
  }, [watchedValues, isDirty, saveDraft]);

  const onFormSubmit = async (data: RecipeFormData) => {
    try {
      await onSubmit(data);
      reset(); // Clear form after successful submission
      toast.success('Recipe saved successfully!');
    } catch (error) {
      console.error('Recipe submission failed:', error);
      toast.error('Failed to save recipe. Please try again.');
    }
  };

  return (
    <form onSubmit={handleSubmit(onFormSubmit)} className="space-y-6">
      {/* Basic Information */}
      <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
        <Input
          label="Recipe Name"
          {...register('name')}
          error={errors.name?.message}
          required
        />
        
        <Select
          label="Difficulty"
          {...register('difficulty')}
          options={[
            { value: 'Easy', label: 'Easy' },
            { value: 'Medium', label: 'Medium' },
            { value: 'Hard', label: 'Hard' },
          ]}
        />
      </div>

      {/* Ingredients Section */}
      <div>
        <label className="block text-sm font-medium mb-2">Ingredients</label>
        {ingredientFields.map((field, index) => (
          <div key={field.id} className="flex items-center space-x-2 mb-2">
            <Input
              placeholder="Ingredient name"
              {...register(`ingredients.${index}.name` as const)}
              error={errors.ingredients?.[index]?.name?.message}
            />
            <Input
              type="number"
              placeholder="Qty"
              {...register(`ingredients.${index}.quantity` as const, { valueAsNumber: true })}
              className="w-20"
            />
            <Select
              placeholder="Unit"
              {...register(`ingredients.${index}.unit` as const)}
              options={MEASUREMENT_UNITS}
              className="w-24"
            />
            <Button
              type="button"
              variant="ghost"
              size="sm"
              onClick={() => removeIngredient(index)}
              disabled={ingredientFields.length === 1}
            >
              <TrashIcon className="w-4 h-4" />
            </Button>
          </div>
        ))}
        <Button
          type="button"
          variant="outline"
          onClick={() => appendIngredient({ name: '', quantity: 0, unit: '' })}
        >
          Add Ingredient
        </Button>
      </div>

      {/* Instructions Section */}
      <div>
        <label className="block text-sm font-medium mb-2">Instructions</label>
        {instructionFields.map((field, index) => (
          <div key={field.id} className="flex items-start space-x-2 mb-2">
            <span className="mt-2 text-sm text-gray-500 font-medium">{index + 1}.</span>
            <Textarea
              placeholder={`Step ${index + 1} instructions...`}
              {...register(`instructions.${index}` as const)}
              error={errors.instructions?.[index]?.message}
              className="flex-1"
            />
            <Button
              type="button"
              variant="ghost"
              size="sm"
              onClick={() => removeInstruction(index)}
              disabled={instructionFields.length === 1}
            >
              <TrashIcon className="w-4 h-4" />
            </Button>
          </div>
        ))}
        <Button
          type="button"
          variant="outline"
          onClick={() => appendInstruction('')}
        >
          Add Step
        </Button>
      </div>

      {/* Submit Actions */}
      <div className="flex justify-end space-x-3">
        <Button
          type="button"
          variant="outline"
          onClick={() => reset()}
          disabled={!isDirty}
        >
          Reset
        </Button>
        <Button
          type="submit"
          isLoading={isSubmitting}
          disabled={!isDirty}
        >
          Save Recipe
        </Button>
      </div>
    </form>
  );
};
```

## Performance Optimizations

### Memoization and Optimization Patterns
```typescript
// hooks/useOptimizedQueries.ts
export const useOptimizedRecipeList = (filters: RecipeFilters) => {
  // Memoize filters to prevent unnecessary re-renders
  const memoizedFilters = useMemo(() => filters, [
    filters.cuisine,
    filters.difficulty,
    filters.maxPrepTime,
    filters.dietaryRestrictions?.join(','),
  ]);

  return useQuery({
    queryKey: ['recipes', 'optimized', memoizedFilters],
    queryFn: () => recipeApi.getRecipes(memoizedFilters),
    select: useCallback((data: Recipe[]) => ({
      recipes: data,
      quickFilters: {
        cuisines: [...new Set(data.map(r => r.cuisine))],
        difficulties: [...new Set(data.map(r => r.difficulty))],
        avgPrepTime: data.reduce((sum, r) => sum + r.prepTime, 0) / data.length,
      },
    }), []),
  });
};

// Optimized component updates
export const OptimizedRecipeCard = React.memo<{ recipe: Recipe }>(({ recipe }) => {
  const [isBookmarked, setIsBookmarked] = useState(recipe.isBookmarked);
  
  // Only re-render if essential props change
  return (
    <Card>
      {/* Recipe card content */}
    </Card>
  );
}, (prevProps, nextProps) => {
  // Custom comparison function
  return (
    prevProps.recipe.id === nextProps.recipe.id &&
    prevProps.recipe.name === nextProps.recipe.name &&
    prevProps.recipe.isBookmarked === nextProps.recipe.isBookmarked
  );
});
```

This comprehensive state management system provides a scalable, performant foundation for the MealPrep React application with clear separation of concerns between server state, client state, and local component state.

*This guide should be updated as the application grows and new state management patterns emerge.*