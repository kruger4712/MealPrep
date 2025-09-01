# React Component Library

## Overview
Comprehensive documentation for the MealPrep React component library, including reusable UI components, design system implementation, and usage guidelines.

## Component Architecture

### Design System Foundation
```typescript
// theme/index.ts
export const theme = {
  colors: {
    primary: {
      50: '#fff7ed',
      100: '#ffedd5',
      500: '#f97316', // Main orange
      600: '#ea580c',
      900: '#9a3412'
    },
    secondary: {
      50: '#f0fdf4',
      100: '#dcfce7',
      500: '#22c55e', // Green
      600: '#16a34a',
      900: '#14532d'
    },
    neutral: {
      50: '#fafafa',
      100: '#f5f5f5',
      500: '#737373',
      800: '#262626',
      900: '#171717'
    },
    error: '#ef4444',
    warning: '#f59e0b',
    success: '#10b981',
    info: '#3b82f6'
  },
  spacing: {
    xs: '0.25rem',   // 4px
    sm: '0.5rem',    // 8px
    md: '1rem',      // 16px
    lg: '1.5rem',    // 24px
    xl: '2rem',      // 32px
    '2xl': '3rem',   // 48px
    '3xl': '4rem'    // 64px
  },
  typography: {
    fontFamily: {
      sans: ['Inter', 'sans-serif'],
      serif: ['Merriweather', 'serif']
    },
    fontSize: {
      xs: '0.75rem',
      sm: '0.875rem',
      base: '1rem',
      lg: '1.125rem',
      xl: '1.25rem',
      '2xl': '1.5rem',
      '3xl': '1.875rem',
      '4xl': '2.25rem'
    }
  },
  borderRadius: {
    sm: '0.25rem',
    md: '0.375rem',
    lg: '0.5rem',
    xl: '0.75rem',
    full: '9999px'
  },
  shadows: {
    sm: '0 1px 2px 0 rgb(0 0 0 / 0.05)',
    md: '0 4px 6px -1px rgb(0 0 0 / 0.1)',
    lg: '0 10px 15px -3px rgb(0 0 0 / 0.1)',
    xl: '0 20px 25px -5px rgb(0 0 0 / 0.1)'
  }
};
```

### Component Categories

## 1. Basic UI Components

### Button Component
```typescript
// components/ui/Button/Button.tsx
export interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'outline' | 'ghost' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
  fullWidth?: boolean;
}

export const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({
    variant = 'primary',
    size = 'md',
    isLoading = false,
    leftIcon,
    rightIcon,
    fullWidth = false,
    children,
    className,
    disabled,
    ...props
  }, ref) => {
    const baseClasses = 'inline-flex items-center justify-center font-medium rounded-lg transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2';
    
    const variantClasses = {
      primary: 'bg-primary-500 text-white hover:bg-primary-600 focus:ring-primary-500',
      secondary: 'bg-secondary-500 text-white hover:bg-secondary-600 focus:ring-secondary-500',
      outline: 'border border-primary-500 text-primary-500 hover:bg-primary-50 focus:ring-primary-500',
      ghost: 'text-primary-500 hover:bg-primary-50 focus:ring-primary-500',
      danger: 'bg-red-500 text-white hover:bg-red-600 focus:ring-red-500'
    };

    const sizeClasses = {
      sm: 'px-3 py-1.5 text-sm',
      md: 'px-4 py-2 text-base',
      lg: 'px-6 py-3 text-lg'
    };

    const classes = cn(
      baseClasses,
      variantClasses[variant],
      sizeClasses[size],
      {
        'w-full': fullWidth,
        'opacity-50 cursor-not-allowed': disabled || isLoading
      },
      className
    );

    return (
      <button
        ref={ref}
        className={classes}
        disabled={disabled || isLoading}
        {...props}
      >
        {isLoading && (
          <LoadingSpinner size="sm" className="mr-2" />
        )}
        {leftIcon && !isLoading && (
          <span className="mr-2">{leftIcon}</span>
        )}
        {children}
        {rightIcon && (
          <span className="ml-2">{rightIcon}</span>
        )}
      </button>
    );
  }
);

// Usage Examples
export const ButtonExamples = () => (
  <div className="space-y-4">
    <Button>Default Button</Button>
    <Button variant="secondary">Secondary</Button>
    <Button variant="outline">Outline</Button>
    <Button variant="ghost">Ghost</Button>
    <Button variant="danger">Delete</Button>
    
    <Button size="sm">Small</Button>
    <Button size="lg">Large</Button>
    
    <Button isLoading>Loading...</Button>
    <Button leftIcon={<PlusIcon />}>Add Recipe</Button>
    <Button rightIcon={<ArrowRightIcon />}>Next Step</Button>
  </div>
);
```

### Input Component
```typescript
// components/ui/Input/Input.tsx
export interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label?: string;
  error?: string;
  helperText?: string;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
  fullWidth?: boolean;
}

export const Input = React.forwardRef<HTMLInputElement, InputProps>(
  ({
    label,
    error,
    helperText,
    leftIcon,
    rightIcon,
    fullWidth = false,
    className,
    id,
    ...props
  }, ref) => {
    const inputId = id || `input-${Math.random().toString(36).substr(2, 9)}`;
    
    const inputClasses = cn(
      'block px-3 py-2 border rounded-lg shadow-sm placeholder-gray-400 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500',
      {
        'w-full': fullWidth,
        'pl-10': leftIcon,
        'pr-10': rightIcon,
        'border-red-300 text-red-900 placeholder-red-300 focus:ring-red-500 focus:border-red-500': error,
        'border-gray-300': !error
      },
      className
    );

    return (
      <div className={cn('relative', { 'w-full': fullWidth })}>
        {label && (
          <label
            htmlFor={inputId}
            className="block text-sm font-medium text-gray-700 mb-1"
          >
            {label}
          </label>
        )}
        
        <div className="relative">
          {leftIcon && (
            <div className="absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none">
              <span className="text-gray-400 text-sm">{leftIcon}</span>
            </div>
          )}
          
          <input
            ref={ref}
            id={inputId}
            className={inputClasses}
            {...props}
          />
          
          {rightIcon && (
            <div className="absolute inset-y-0 right-0 pr-3 flex items-center">
              <span className="text-gray-400 text-sm">{rightIcon}</span>
            </div>
          )}
        </div>
        
        {error && (
          <p className="mt-1 text-sm text-red-600" id={`${inputId}-error`}>
            {error}
          </p>
        )}
        
        {helperText && !error && (
          <p className="mt-1 text-sm text-gray-500" id={`${inputId}-description`}>
            {helperText}
          </p>
        )}
      </div>
    );
  }
);
```

### Card Component
```typescript
// components/ui/Card/Card.tsx
export interface CardProps extends React.HTMLAttributes<HTMLDivElement> {
  variant?: 'elevated' | 'outlined' | 'filled';
  padding?: 'sm' | 'md' | 'lg';
  hoverable?: boolean;
}

export const Card = React.forwardRef<HTMLDivElement, CardProps>(
  ({
    variant = 'elevated',
    padding = 'md',
    hoverable = false,
    className,
    children,
    ...props
  }, ref) => {
    const baseClasses = 'rounded-lg';
    
    const variantClasses = {
      elevated: 'bg-white shadow-md',
      outlined: 'bg-white border border-gray-200',
      filled: 'bg-gray-50'
    };

    const paddingClasses = {
      sm: 'p-4',
      md: 'p-6',
      lg: 'p-8'
    };

    const classes = cn(
      baseClasses,
      variantClasses[variant],
      paddingClasses[padding],
      {
        'hover:shadow-lg transition-shadow cursor-pointer': hoverable
      },
      className
    );

    return (
      <div ref={ref} className={classes} {...props}>
        {children}
      </div>
    );
  }
);

export const CardHeader = ({ children, className, ...props }: React.HTMLAttributes<HTMLDivElement>) => (
  <div className={cn('mb-4', className)} {...props}>
    {children}
  </div>
);

export const CardTitle = ({ children, className, ...props }: React.HTMLAttributes<HTMLHeadingElement>) => (
  <h3 className={cn('text-lg font-semibold text-gray-900', className)} {...props}>
    {children}
  </h3>
);

export const CardContent = ({ children, className, ...props }: React.HTMLAttributes<HTMLDivElement>) => (
  <div className={cn('text-gray-600', className)} {...props}>
    {children}
  </div>
);

export const CardActions = ({ children, className, ...props }: React.HTMLAttributes<HTMLDivElement>) => (
  <div className={cn('mt-4 flex items-center justify-end space-x-2', className)} {...props}>
    {children}
  </div>
);
```

## 2. MealPrep-Specific Components

### RecipeCard Component
```typescript
// components/recipes/RecipeCard/RecipeCard.tsx
export interface RecipeCardProps {
  recipe: Recipe;
  onFavoriteToggle?: (recipeId: string) => void;
  onAddToMenu?: (recipeId: string) => void;
  onViewDetails?: (recipeId: string) => void;
  showActions?: boolean;
  size?: 'sm' | 'md' | 'lg';
}

export const RecipeCard: React.FC<RecipeCardProps> = ({
  recipe,
  onFavoriteToggle,
  onAddToMenu,
  onViewDetails,
  showActions = true,
  size = 'md'
}) => {
  const [isFavorited, setIsFavorited] = useState(recipe.isFavorited);
  const [imageError, setImageError] = useState(false);

  const handleFavoriteClick = () => {
    setIsFavorited(!isFavorited);
    onFavoriteToggle?.(recipe.id);
  };

  const sizeClasses = {
    sm: 'w-64',
    md: 'w-80',
    lg: 'w-96'
  };

  return (
    <Card 
      className={cn(sizeClasses[size], 'overflow-hidden')} 
      hoverable
      onClick={() => onViewDetails?.(recipe.id)}
    >
      {/* Recipe Image */}
      <div className="relative h-48 bg-gray-200">
        {!imageError && recipe.imageUrl ? (
          <img
            src={recipe.imageUrl}
            alt={recipe.name}
            className="w-full h-full object-cover"
            onError={() => setImageError(true)}
          />
        ) : (
          <div className="w-full h-full flex items-center justify-center">
            <ChefHatIcon className="w-16 h-16 text-gray-400" />
          </div>
        )}
        
        {/* Difficulty Badge */}
        <div className="absolute top-2 left-2">
          <DifficultyBadge difficulty={recipe.difficulty} />
        </div>
        
        {/* Favorite Button */}
        {showActions && (
          <button
            onClick={(e) => {
              e.stopPropagation();
              handleFavoriteClick();
            }}
            className="absolute top-2 right-2 p-2 rounded-full bg-white/80 hover:bg-white transition-colors"
          >
            <HeartIcon 
              className={cn('w-5 h-5', {
                'text-red-500 fill-current': isFavorited,
                'text-gray-600': !isFavorited
              })} 
            />
          </button>
        )}
      </div>

      {/* Recipe Content */}
      <div className="p-4">
        <CardHeader>
          <CardTitle className="line-clamp-2">{recipe.name}</CardTitle>
          <p className="text-sm text-gray-500 mt-1 line-clamp-2">
            {recipe.description}
          </p>
        </CardHeader>

        <CardContent>
          {/* Recipe Stats */}
          <div className="flex items-center justify-between text-sm text-gray-600 mb-3">
            <div className="flex items-center">
              <ClockIcon className="w-4 h-4 mr-1" />
              {recipe.totalTime}m
            </div>
            <div className="flex items-center">
              <UsersIcon className="w-4 h-4 mr-1" />
              {recipe.servings} servings
            </div>
            <div className="flex items-center">
              <DollarSignIcon className="w-4 h-4 mr-1" />
              ${recipe.estimatedCost?.toFixed(2)}
            </div>
          </div>

          {/* Tags */}
          <div className="flex flex-wrap gap-1 mb-3">
            {recipe.tags?.slice(0, 3).map((tag) => (
              <span
                key={tag}
                className="px-2 py-1 text-xs bg-gray-100 text-gray-600 rounded-full"
              >
                {tag}
              </span>
            ))}
            {recipe.tags && recipe.tags.length > 3 && (
              <span className="px-2 py-1 text-xs bg-gray-100 text-gray-600 rounded-full">
                +{recipe.tags.length - 3}
              </span>
            )}
          </div>

          {/* Family Fit Score */}
          {recipe.familyFitScore && (
            <div className="mb-3">
              <FamilyFitScore score={recipe.familyFitScore} />
            </div>
          )}
        </CardContent>

        {/* Actions */}
        {showActions && (
          <CardActions>
            <Button
              variant="outline"
              size="sm"
              onClick={(e) => {
                e.stopPropagation();
                onAddToMenu?.(recipe.id);
              }}
            >
              Add to Menu
            </Button>
            <Button
              size="sm"
              onClick={() => onViewDetails?.(recipe.id)}
            >
              View Recipe
            </Button>
          </CardActions>
        )}
      </div>
    </Card>
  );
};
```

### MealPlanCalendar Component
```typescript
// components/meal-planning/MealPlanCalendar/MealPlanCalendar.tsx
export interface MealPlanCalendarProps {
  startDate: Date;
  endDate: Date;
  mealPlan: WeeklyMealPlan;
  onMealClick?: (meal: PlannedMeal) => void;
  onEmptySlotClick?: (date: Date, mealType: MealType) => void;
  onMealDrop?: (meal: PlannedMeal, targetDate: Date, targetMealType: MealType) => void;
  readOnly?: boolean;
}

export const MealPlanCalendar: React.FC<MealPlanCalendarProps> = ({
  startDate,
  endDate,
  mealPlan,
  onMealClick,
  onEmptySlotClick,
  onMealDrop,
  readOnly = false
}) => {
  const dates = eachDayOfInterval({ start: startDate, end: endDate });
  const mealTypes = ['breakfast', 'lunch', 'dinner'] as const;

  return (
    <div className="bg-white rounded-lg shadow-sm border">
      {/* Header */}
      <div className="grid grid-cols-8 border-b">
        <div className="p-4 font-medium text-gray-600">Meal</div>
        {dates.map((date) => (
          <div key={date.toISOString()} className="p-4 text-center border-l">
            <div className="font-medium text-gray-900">
              {format(date, 'EEE')}
            </div>
            <div className="text-sm text-gray-500">
              {format(date, 'MMM d')}
            </div>
          </div>
        ))}
      </div>

      {/* Meal Rows */}
      {mealTypes.map((mealType) => (
        <div key={mealType} className="grid grid-cols-8 border-b last:border-b-0">
          {/* Meal Type Label */}
          <div className="p-4 bg-gray-50 font-medium text-gray-700 capitalize flex items-center">
            <MealTypeIcon type={mealType} className="w-5 h-5 mr-2" />
            {mealType}
          </div>

          {/* Daily Meal Slots */}
          {dates.map((date) => {
            const meal = getMealForDateAndType(mealPlan, date, mealType);
            
            return (
              <MealSlot
                key={`${format(date, 'yyyy-MM-dd')}-${mealType}`}
                date={date}
                mealType={mealType}
                meal={meal}
                onMealClick={onMealClick}
                onEmptySlotClick={onEmptySlotClick}
                onMealDrop={onMealDrop}
                readOnly={readOnly}
              />
            );
          })}
        </div>
      ))}
    </div>
  );
};

const MealSlot: React.FC<{
  date: Date;
  mealType: MealType;
  meal?: PlannedMeal;
  onMealClick?: (meal: PlannedMeal) => void;
  onEmptySlotClick?: (date: Date, mealType: MealType) => void;
  onMealDrop?: (meal: PlannedMeal, targetDate: Date, targetMealType: MealType) => void;
  readOnly?: boolean;
}> = ({ date, mealType, meal, onMealClick, onEmptySlotClick, onMealDrop, readOnly }) => {
  const [isDragOver, setIsDragOver] = useState(false);

  const handleDrop = (e: React.DragEvent) => {
    e.preventDefault();
    setIsDragOver(false);
    
    const draggedMeal = JSON.parse(e.dataTransfer.getData('text/plain'));
    onMealDrop?.(draggedMeal, date, mealType);
  };

  const handleDragOver = (e: React.DragEvent) => {
    e.preventDefault();
    setIsDragOver(true);
  };

  const handleDragLeave = () => {
    setIsDragOver(false);
  };

  if (meal) {
    return (
      <div
        className={cn(
          'p-2 border-l min-h-[80px] flex items-center transition-colors',
          {
            'hover:bg-gray-50 cursor-pointer': !readOnly,
            'bg-blue-50 border-blue-200': isDragOver
          }
        )}
        onClick={() => !readOnly && onMealClick?.(meal)}
        onDrop={handleDrop}
        onDragOver={handleDragOver}
        onDragLeave={handleDragLeave}
      >
        <MealPlanItem 
          meal={meal} 
          draggable={!readOnly}
          compact
        />
      </div>
    );
  }

  return (
    <div
      className={cn(
        'p-2 border-l min-h-[80px] flex items-center justify-center transition-colors',
        {
          'hover:bg-gray-50 cursor-pointer': !readOnly,
          'bg-blue-50 border-blue-200': isDragOver
        }
      )}
      onClick={() => !readOnly && onEmptySlotClick?.(date, mealType)}
      onDrop={handleDrop}
      onDragOver={handleDragOver}
      onDragLeave={handleDragLeave}
    >
      {!readOnly && (
        <button className="w-full h-full flex items-center justify-center text-gray-400 hover:text-gray-600 rounded-lg border-2 border-dashed border-gray-200 hover:border-gray-300">
          <PlusIcon className="w-6 h-6" />
        </button>
      )}
    </div>
  );
};
```

### FamilyMemberSelector Component
```typescript
// components/family/FamilyMemberSelector/FamilyMemberSelector.tsx
export interface FamilyMemberSelectorProps {
  familyMembers: FamilyMember[];
  selectedMembers: string[];
  onSelectionChange: (memberIds: string[]) => void;
  title?: string;
  description?: string;
  allowSelectAll?: boolean;
  required?: boolean;
}

export const FamilyMemberSelector: React.FC<FamilyMemberSelectorProps> = ({
  familyMembers,
  selectedMembers,
  onSelectionChange,
  title = "Select Family Members",
  description,
  allowSelectAll = true,
  required = false
}) => {
  const allMemberIds = familyMembers.map(m => m.id);
  const isAllSelected = selectedMembers.length === familyMembers.length;
  const isPartialSelected = selectedMembers.length > 0 && selectedMembers.length < familyMembers.length;

  const handleSelectAll = () => {
    if (isAllSelected) {
      onSelectionChange([]);
    } else {
      onSelectionChange(allMemberIds);
    }
  };

  const handleMemberToggle = (memberId: string) => {
    if (selectedMembers.includes(memberId)) {
      onSelectionChange(selectedMembers.filter(id => id !== memberId));
    } else {
      onSelectionChange([...selectedMembers, memberId]);
    }
  };

  return (
    <div className="space-y-4">
      {/* Header */}
      <div>
        <h3 className="text-lg font-medium text-gray-900">
          {title}
          {required && <span className="text-red-500 ml-1">*</span>}
        </h3>
        {description && (
          <p className="text-sm text-gray-600 mt-1">{description}</p>
        )}
      </div>

      {/* Select All Option */}
      {allowSelectAll && (
        <div className="flex items-center space-x-3 p-3 bg-gray-50 rounded-lg">
          <Checkbox
            id="select-all"
            checked={isAllSelected}
            indeterminate={isPartialSelected}
            onChange={handleSelectAll}
          />
          <label htmlFor="select-all" className="text-sm font-medium text-gray-700">
            Select All Family Members
          </label>
        </div>
      )}

      {/* Family Member List */}
      <div className="space-y-2">
        {familyMembers.map((member) => (
          <FamilyMemberCard
            key={member.id}
            member={member}
            selected={selectedMembers.includes(member.id)}
            onToggle={() => handleMemberToggle(member.id)}
          />
        ))}
      </div>

      {/* Selection Summary */}
      {selectedMembers.length > 0 && (
        <div className="mt-4 p-3 bg-primary-50 rounded-lg">
          <p className="text-sm text-primary-700">
            Selected {selectedMembers.length} of {familyMembers.length} family members
          </p>
          <div className="mt-2 flex flex-wrap gap-1">
            {selectedMembers.map((memberId) => {
              const member = familyMembers.find(m => m.id === memberId);
              return member ? (
                <span
                  key={memberId}
                  className="inline-flex items-center px-2 py-1 rounded-full text-xs bg-primary-100 text-primary-700"
                >
                  {member.name}
                  <button
                    onClick={() => handleMemberToggle(memberId)}
                    className="ml-1 text-primary-500 hover:text-primary-700"
                  >
                    <XMarkIcon className="w-3 h-3" />
                  </button>
                </span>
              ) : null;
            })}
          </div>
        </div>
      )}
    </div>
  );
};

const FamilyMemberCard: React.FC<{
  member: FamilyMember;
  selected: boolean;
  onToggle: () => void;
}> = ({ member, selected, onToggle }) => {
  return (
    <div
      className={cn(
        'flex items-center space-x-3 p-3 rounded-lg border-2 cursor-pointer transition-all',
        {
          'border-primary-200 bg-primary-50': selected,
          'border-gray-200 hover:border-gray-300': !selected
        }
      )}
      onClick={onToggle}
    >
      <Checkbox
        checked={selected}
        onChange={onToggle}
        onClick={(e) => e.stopPropagation()}
      />
      
      <div className="flex-1 min-w-0">
        <div className="flex items-center space-x-2">
          <span className="font-medium text-gray-900">{member.name}</span>
          <span className="text-sm text-gray-500">({member.age}y)</span>
          {member.relationship !== 'Self' && (
            <span className="text-xs text-gray-400">{member.relationship}</span>
          )}
        </div>
        
        {/* Dietary Restrictions & Allergies */}
        <div className="mt-1 flex flex-wrap gap-1">
          {member.dietaryRestrictions.map((restriction) => (
            <span
              key={restriction}
              className="inline-block px-2 py-0.5 text-xs bg-blue-100 text-blue-700 rounded"
            >
              {restriction}
            </span>
          ))}
          {member.allergies.map((allergy) => (
            <span
              key={allergy}
              className="inline-block px-2 py-0.5 text-xs bg-red-100 text-red-700 rounded"
            >
              ?? {allergy}
            </span>
          ))}
        </div>
      </div>
    </div>
  );
};
```

## 3. Form Components

### SearchInput Component
```typescript
// components/ui/SearchInput/SearchInput.tsx
export interface SearchInputProps extends Omit<InputProps, 'leftIcon' | 'rightIcon'> {
  onSearch?: (query: string) => void;
  onClear?: () => void;
  suggestions?: string[];
  showSuggestions?: boolean;
  isLoading?: boolean;
  debounceMs?: number;
}

export const SearchInput: React.FC<SearchInputProps> = ({
  onSearch,
  onClear,
  suggestions = [],
  showSuggestions = false,
  isLoading = false,
  debounceMs = 300,
  className,
  ...props
}) => {
  const [query, setQuery] = useState(props.value || '');
  const [showDropdown, setShowDropdown] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);
  
  // Debounced search
  const debouncedSearch = useMemo(
    () => debounce((searchQuery: string) => {
      onSearch?.(searchQuery);
    }, debounceMs),
    [onSearch, debounceMs]
  );

  useEffect(() => {
    debouncedSearch(query);
  }, [query, debouncedSearch]);

  const handleInputChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setQuery(value);
    setShowDropdown(value.length > 0 && showSuggestions);
  };

  const handleClear = () => {
    setQuery('');
    setShowDropdown(false);
    onClear?.();
    inputRef.current?.focus();
  };

  const handleSuggestionClick = (suggestion: string) => {
    setQuery(suggestion);
    setShowDropdown(false);
    onSearch?.(suggestion);
  };

  return (
    <div className="relative">
      <Input
        ref={inputRef}
        {...props}
        value={query}
        onChange={handleInputChange}
        leftIcon={
          isLoading ? (
            <LoadingSpinner size="sm" />
          ) : (
            <MagnifyingGlassIcon className="w-4 h-4" />
          )
        }
        rightIcon={
          query && (
            <button
              onClick={handleClear}
              className="text-gray-400 hover:text-gray-600"
            >
              <XMarkIcon className="w-4 h-4" />
            </button>
          )
        }
        className={className}
      />

      {/* Suggestions Dropdown */}
      {showDropdown && suggestions.length > 0 && (
        <div className="absolute z-10 w-full mt-1 bg-white border border-gray-200 rounded-lg shadow-lg max-h-60 overflow-auto">
          {suggestions.map((suggestion, index) => (
            <button
              key={index}
              className="w-full px-4 py-2 text-left hover:bg-gray-50 focus:bg-gray-50 focus:outline-none first:rounded-t-lg last:rounded-b-lg"
              onClick={() => handleSuggestionClick(suggestion)}
            >
              <span className="text-gray-900">{suggestion}</span>
            </button>
          ))}
        </div>
      )}
    </div>
  );
};
```

## 4. Utility Components

### LoadingSpinner Component
```typescript
// components/ui/LoadingSpinner/LoadingSpinner.tsx
export interface LoadingSpinnerProps {
  size?: 'sm' | 'md' | 'lg';
  color?: 'primary' | 'secondary' | 'white';
  className?: string;
}

export const LoadingSpinner: React.FC<LoadingSpinnerProps> = ({
  size = 'md',
  color = 'primary',
  className
}) => {
  const sizeClasses = {
    sm: 'w-4 h-4',
    md: 'w-6 h-6',
    lg: 'w-8 h-8'
  };

  const colorClasses = {
    primary: 'text-primary-500',
    secondary: 'text-secondary-500',
    white: 'text-white'
  };

  return (
    <div
      className={cn(
        'animate-spin rounded-full border-2 border-current border-t-transparent',
        sizeClasses[size],
        colorClasses[color],
        className
      )}
      role="status"
      aria-label="Loading"
    >
      <span className="sr-only">Loading...</span>
    </div>
  );
};
```

### Toast Notification System
```typescript
// components/ui/Toast/Toast.tsx
export interface ToastProps {
  id: string;
  type: 'success' | 'error' | 'warning' | 'info';
  title: string;
  description?: string;
  duration?: number;
  onClose: (id: string) => void;
}

export const Toast: React.FC<ToastProps> = ({
  id,
  type,
  title,
  description,
  duration = 5000,
  onClose
}) => {
  useEffect(() => {
    const timer = setTimeout(() => {
      onClose(id);
    }, duration);

    return () => clearTimeout(timer);
  }, [id, duration, onClose]);

  const typeConfig = {
    success: {
      icon: CheckCircleIcon,
      bgColor: 'bg-green-50',
      borderColor: 'border-green-200',
      textColor: 'text-green-800',
      iconColor: 'text-green-400'
    },
    error: {
      icon: XCircleIcon,
      bgColor: 'bg-red-50',
      borderColor: 'border-red-200',
      textColor: 'text-red-800',
      iconColor: 'text-red-400'
    },
    warning: {
      icon: ExclamationTriangleIcon,
      bgColor: 'bg-yellow-50',
      borderColor: 'border-yellow-200',
      textColor: 'text-yellow-800',
      iconColor: 'text-yellow-400'
    },
    info: {
      icon: InformationCircleIcon,
      bgColor: 'bg-blue-50',
      borderColor: 'border-blue-200',
      textColor: 'text-blue-800',
      iconColor: 'text-blue-400'
    }
  };

  const config = typeConfig[type];
  const Icon = config.icon;

  return (
    <div
      className={cn(
        'relative flex items-start p-4 mb-4 border rounded-lg shadow-sm',
        config.bgColor,
        config.borderColor
      )}
    >
      <Icon className={cn('w-5 h-5 mr-3 mt-0.5 flex-shrink-0', config.iconColor)} />
      
      <div className="flex-1 min-w-0">
        <p className={cn('text-sm font-medium', config.textColor)}>
          {title}
        </p>
        {description && (
          <p className={cn('mt-1 text-sm', config.textColor, 'opacity-90')}>
            {description}
          </p>
        )}
      </div>
      
      <button
        onClick={() => onClose(id)}
        className={cn(
          'ml-3 flex-shrink-0 rounded-md p-1.5 hover:bg-black/5 focus:outline-none focus:ring-2 focus:ring-offset-2',
          config.textColor
        )}
      >
        <XMarkIcon className="w-4 h-4" />
      </button>
    </div>
  );
};

// Toast Provider and Hook
export const ToastProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [toasts, setToasts] = useState<ToastProps[]>([]);

  const addToast = useCallback((toast: Omit<ToastProps, 'id' | 'onClose'>) => {
    const id = Math.random().toString(36).substr(2, 9);
    setToasts(current => [...current, { ...toast, id, onClose: removeToast }]);
  }, []);

  const removeToast = useCallback((id: string) => {
    setToasts(current => current.filter(toast => toast.id !== id));
  }, []);

  return (
    <ToastContext.Provider value={{ addToast, removeToast }}>
      {children}
      
      {/* Toast Container */}
      <div className="fixed top-0 right-0 z-50 p-6 space-y-4 max-w-sm w-full">
        <AnimatePresence>
          {toasts.map(toast => (
            <motion.div
              key={toast.id}
              initial={{ opacity: 0, x: 100 }}
              animate={{ opacity: 1, x: 0 }}
              exit={{ opacity: 0, x: 100 }}
              transition={{ duration: 0.3 }}
            >
              <Toast {...toast} />
            </motion.div>
          ))}
        </AnimatePresence>
      </div>
    </ToastContext.Provider>
  );
};

export const useToast = () => {
  const context = useContext(ToastContext);
  if (!context) {
    throw new Error('useToast must be used within a ToastProvider');
  }
  return context;
};
```

## Component Documentation Standards

### Documentation Template
```typescript
/**
 * Component Name
 * 
 * Brief description of what the component does and when to use it.
 * 
 * @example
 * ```tsx
 * <ComponentName
 *   prop1="value"
 *   prop2={true}
 *   onAction={handleAction}
 * />
 * ```
 */
```

### Storybook Integration
```typescript
// Button.stories.tsx
export default {
  title: 'UI/Button',
  component: Button,
  parameters: {
    docs: {
      description: {
        component: 'A versatile button component with multiple variants and sizes.'
      }
    }
  },
  argTypes: {
    variant: {
      control: { type: 'select' },
      options: ['primary', 'secondary', 'outline', 'ghost', 'danger']
    },
    size: {
      control: { type: 'select' },
      options: ['sm', 'md', 'lg']
    }
  }
} as ComponentMeta<typeof Button>;

export const Primary: ComponentStory<typeof Button> = (args) => <Button {...args} />;
Primary.args = {
  children: 'Primary Button',
  variant: 'primary'
};

export const AllVariants: ComponentStory<typeof Button> = () => (
  <div className="space-x-4">
    <Button variant="primary">Primary</Button>
    <Button variant="secondary">Secondary</Button>
    <Button variant="outline">Outline</Button>
    <Button variant="ghost">Ghost</Button>
    <Button variant="danger">Danger</Button>
  </div>
);
```

This comprehensive React component library provides the foundation for building consistent, accessible, and maintainable user interfaces in the MealPrep application.

*This documentation should be updated as components evolve and new patterns emerge.*