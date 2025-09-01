# Accessibility Guidelines

## Overview
Comprehensive accessibility implementation guide for the MealPrep React application, ensuring WCAG 2.1 AA compliance and inclusive design for users with disabilities, following best practices for semantic HTML, ARIA, keyboard navigation, and assistive technologies.

## Accessibility Standards

### WCAG 2.1 AA Compliance Requirements
```typescript
// Accessibility standards and targets
export const ACCESSIBILITY_STANDARDS = {
  // WCAG 2.1 AA Requirements
  contrast: {
    normal: 4.5, // Minimum contrast ratio for normal text
    large: 3.0,  // Minimum contrast ratio for large text (18pt+ or 14pt+ bold)
  },
  
  // Timing requirements
  timing: {
    noTimeLimit: true, // No time limits on user interactions
    pauseExtend: true, // Users can pause/extend time limits
  },
  
  // Navigation requirements
  navigation: {
    skipLinks: true,     // Skip navigation links present
    headingStructure: true, // Proper heading hierarchy
    focusManagement: true,  // Proper focus management
  },
  
  // Content requirements
  content: {
    altText: true,       // All images have alt text
    formLabels: true,    // All form controls have labels
    errorMessages: true, // Clear error messages
  },
} as const;
```

## Semantic HTML Foundation

### Document Structure
```typescript
// components/layout/AccessibleLayout.tsx
export const AccessibleLayout: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  return (
    <div className="min-h-screen bg-background">
      {/* Skip Links */}
      <a
        href="#main-content"
        className="sr-only focus:not-sr-only focus:absolute focus:top-4 focus:left-4 z-50 bg-primary-600 text-white px-4 py-2 rounded"
      >
        Skip to main content
      </a>
      
      <a
        href="#navigation"
        className="sr-only focus:not-sr-only focus:absolute focus:top-4 focus:left-32 z-50 bg-primary-600 text-white px-4 py-2 rounded"
      >
        Skip to navigation
      </a>

      {/* Main Navigation */}
      <nav
        id="navigation"
        role="navigation"
        aria-label="Main navigation"
        className="bg-white border-b border-gray-200"
      >
        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
          <div className="flex justify-between h-16">
            {/* Logo and main nav items */}
            <div className="flex items-center">
              <Link
                to="/"
                className="flex items-center text-xl font-bold text-primary-600 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:ring-offset-2 rounded"
              >
                <img
                  src="/logo.svg"
                  alt="MealPrep - AI-powered family meal planning"
                  className="w-8 h-8 mr-2"
                />
                MealPrep
              </Link>
            </div>
          </div>
        </div>
      </nav>

      {/* Main Content */}
      <main
        id="main-content"
        role="main"
        className="flex-1"
        tabIndex={-1} // Allow programmatic focus
      >
        {children}
      </main>

      {/* Footer */}
      <footer
        role="contentinfo"
        className="bg-gray-50 border-t border-gray-200"
      >
        <div className="max-w-7xl mx-auto py-12 px-4 sm:px-6 lg:px-8">
          {/* Footer content */}
        </div>
      </footer>
    </div>
  );
};
```

### Heading Hierarchy
```typescript
// components/ui/AccessibleHeading.tsx
interface HeadingProps {
  level: 1 | 2 | 3 | 4 | 5 | 6;
  children: React.ReactNode;
  className?: string;
  id?: string;
}

export const Heading: React.FC<HeadingProps> = ({ 
  level, 
  children, 
  className,
  id 
}) => {
  const Component = `h${level}` as keyof JSX.IntrinsicElements;
  
  // Ensure proper heading hierarchy
  const headingClasses = {
    1: 'text-4xl font-bold text-gray-900',
    2: 'text-3xl font-semibold text-gray-900',
    3: 'text-2xl font-semibold text-gray-900',
    4: 'text-xl font-medium text-gray-900',
    5: 'text-lg font-medium text-gray-900',
    6: 'text-base font-medium text-gray-900',
  };

  return (
    <Component
      id={id}
      className={cn(headingClasses[level], className)}
    >
      {children}
    </Component>
  );
};

// Page structure example
export const RecipesPage: React.FC = () => {
  return (
    <div>
      <Heading level={1} id="page-title">
        Recipe Collection
      </Heading>
      
      <section aria-labelledby="filters-heading">
        <Heading level={2} id="filters-heading">
          Filter Recipes
        </Heading>
        {/* Filter components */}
      </section>
      
      <section aria-labelledby="results-heading">
        <Heading level={2} id="results-heading">
          Recipe Results
        </Heading>
        {/* Recipe list */}
      </section>
    </div>
  );
};
```

## ARIA Implementation

### ARIA Labels and Descriptions
```typescript
// components/ui/AccessibleForm.tsx
export const AccessibleFormField: React.FC<{
  label: string;
  description?: string;
  error?: string;
  required?: boolean;
  children: React.ReactElement;
}> = ({ label, description, error, required, children }) => {
  const fieldId = useId();
  const descriptionId = description ? `${fieldId}-description` : undefined;
  const errorId = error ? `${fieldId}-error` : undefined;

  // Clone child element with accessibility props
  const field = React.cloneElement(children, {
    id: fieldId,
    'aria-labelledby': `${fieldId}-label`,
    'aria-describedby': [descriptionId, errorId].filter(Boolean).join(' ') || undefined,
    'aria-invalid': error ? 'true' : undefined,
    'aria-required': required,
  });

  return (
    <div className="space-y-2">
      <label
        id={`${fieldId}-label`}
        htmlFor={fieldId}
        className={cn(
          'block text-sm font-medium text-gray-700',
          required && "after:content-['*'] after:text-red-500 after:ml-1"
        )}
      >
        {label}
      </label>
      
      {description && (
        <p
          id={descriptionId}
          className="text-sm text-gray-600"
        >
          {description}
        </p>
      )}
      
      {field}
      
      {error && (
        <p
          id={errorId}
          role="alert"
          className="text-sm text-red-600"
        >
          {error}
        </p>
      )}
    </div>
  );
};

// Complex component with ARIA
export const RecipeCard: React.FC<{ recipe: Recipe }> = ({ recipe }) => {
  const [expanded, setExpanded] = useState(false);
  const cardId = useId();
  const contentId = `${cardId}-content`;

  return (
    <article
      className="bg-white rounded-lg shadow border border-gray-200"
      aria-labelledby={`${cardId}-title`}
    >
      <div className="p-6">
        {/* Recipe Image */}
        <div className="mb-4">
          <img
            src={recipe.imageUrl}
            alt={`${recipe.name} - ${recipe.description.substring(0, 100)}`}
            className="w-full h-48 object-cover rounded-md"
          />
        </div>

        {/* Recipe Header */}
        <header>
          <h3
            id={`${cardId}-title`}
            className="text-xl font-semibold text-gray-900 mb-2"
          >
            {recipe.name}
          </h3>
          
          <div
            className="flex items-center text-sm text-gray-600 space-x-4"
            aria-label="Recipe metadata"
          >
            <span aria-label={`Preparation time: ${recipe.prepTime} minutes`}>
              <ClockIcon className="w-4 h-4 inline mr-1" aria-hidden="true" />
              {recipe.prepTime}m prep
            </span>
            
            <span aria-label={`Cooking time: ${recipe.cookTime} minutes`}>
              <FireIcon className="w-4 h-4 inline mr-1" aria-hidden="true" />
              {recipe.cookTime}m cook
            </span>
            
            <span aria-label={`Serves ${recipe.servings} people`}>
              <UsersIcon className="w-4 h-4 inline mr-1" aria-hidden="true" />
              {recipe.servings} servings
            </span>
          </div>
        </header>

        {/* Expandable Content */}
        <button
          onClick={() => setExpanded(!expanded)}
          aria-expanded={expanded}
          aria-controls={contentId}
          className="mt-4 text-primary-600 hover:text-primary-700 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:ring-offset-2 rounded"
        >
          {expanded ? 'Hide' : 'Show'} recipe details
          <ChevronDownIcon
            className={cn(
              'w-4 h-4 inline ml-1 transition-transform',
              expanded ? 'rotate-180' : ''
            )}
            aria-hidden="true"
          />
        </button>

        {/* Collapsible Content */}
        <div
          id={contentId}
          className={cn(
            'mt-4 space-y-4',
            expanded ? 'block' : 'hidden'
          )}
          aria-hidden={!expanded}
        >
          <section aria-labelledby={`${cardId}-ingredients`}>
            <h4 id={`${cardId}-ingredients`} className="font-medium text-gray-900">
              Ingredients
            </h4>
            <ul className="mt-2 space-y-1">
              {recipe.ingredients.map((ingredient, index) => (
                <li key={index} className="text-sm text-gray-600">
                  {ingredient.quantity} {ingredient.unit} {ingredient.name}
                </li>
              ))}
            </ul>
          </section>

          <section aria-labelledby={`${cardId}-instructions`}>
            <h4 id={`${cardId}-instructions`} className="font-medium text-gray-900">
              Instructions
            </h4>
            <ol className="mt-2 space-y-2 list-decimal list-inside">
              {recipe.instructions.map((instruction, index) => (
                <li key={index} className="text-sm text-gray-600">
                  {instruction}
                </li>
              ))}
            </ol>
          </section>
        </div>
      </div>
    </article>
  );
};
```

### Live Regions and Announcements
```typescript
// components/ui/LiveRegion.tsx
export const LiveRegion: React.FC<{
  message: string;
  priority?: 'polite' | 'assertive';
  atomic?: boolean;
}> = ({ message, priority = 'polite', atomic = true }) => {
  return (
    <div
      role="status"
      aria-live={priority}
      aria-atomic={atomic}
      className="sr-only"
    >
      {message}
    </div>
  );
};

// Hook for announcements
export const useAnnouncer = () => {
  const [message, setMessage] = useState('');
  const [priority, setPriority] = useState<'polite' | 'assertive'>('polite');

  const announce = useCallback((
    text: string, 
    urgency: 'polite' | 'assertive' = 'polite'
  ) => {
    setPriority(urgency);
    setMessage(text);
    
    // Clear message after announcement
    setTimeout(() => setMessage(''), 1000);
  }, []);

  return {
    announce,
    LiveRegion: () => (
      <LiveRegion message={message} priority={priority} />
    ),
  };
};

// Usage in components
export const SearchResults: React.FC<{ results: Recipe[] }> = ({ results }) => {
  const { announce, LiveRegion } = useAnnouncer();

  useEffect(() => {
    if (results.length === 0) {
      announce('No recipes found. Try adjusting your search criteria.');
    } else {
      announce(`Found ${results.length} recipe${results.length === 1 ? '' : 's'}.`);
    }
  }, [results, announce]);

  return (
    <div>
      <LiveRegion />
      {/* Search results content */}
    </div>
  );
};
```

## Keyboard Navigation

### Focus Management
```typescript
// hooks/useFocusManagement.ts
export const useFocusManagement = () => {
  const focusableElementsSelector = [
    'a[href]',
    'button:not([disabled])',
    'input:not([disabled])',
    'select:not([disabled])',
    'textarea:not([disabled])',
    '[tabindex]:not([tabindex="-1"])',
    '[contenteditable="true"]',
  ].join(', ');

  const trapFocus = useCallback((container: HTMLElement) => {
    const focusableElements = container.querySelectorAll(focusableElementsSelector);
    const firstElement = focusableElements[0] as HTMLElement;
    const lastElement = focusableElements[focusableElements.length - 1] as HTMLElement;

    const handleKeyDown = (event: KeyboardEvent) => {
      if (event.key === 'Tab') {
        if (event.shiftKey) {
          // Shift + Tab
          if (document.activeElement === firstElement) {
            event.preventDefault();
            lastElement.focus();
          }
        } else {
          // Tab
          if (document.activeElement === lastElement) {
            event.preventDefault();
            firstElement.focus();
          }
        }
      }
    };

    container.addEventListener('keydown', handleKeyDown);
    return () => container.removeEventListener('keydown', handleKeyDown);
  }, [focusableElementsSelector]);

  const moveFocusToElement = useCallback((element: HTMLElement | null) => {
    if (element) {
      element.focus();
      element.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
    }
  }, []);

  const moveFocusToFirstError = useCallback((formElement: HTMLElement) => {
    const errorElement = formElement.querySelector('[aria-invalid="true"]') as HTMLElement;
    if (errorElement) {
      moveFocusToElement(errorElement);
    }
  }, [moveFocusToElement]);

  return {
    trapFocus,
    moveFocusToElement,
    moveFocusToFirstError,
  };
};

// Modal with focus trap
export const AccessibleModal: React.FC<{
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
}> = ({ isOpen, onClose, title, children }) => {
  const modalRef = useRef<HTMLDivElement>(null);
  const { trapFocus, moveFocusToElement } = useFocusManagement();
  const titleId = useId();

  // Focus management
  useEffect(() => {
    if (!isOpen || !modalRef.current) return;

    const previousActiveElement = document.activeElement as HTMLElement;
    
    // Focus modal on open
    moveFocusToElement(modalRef.current);
    
    // Trap focus within modal
    const cleanup = trapFocus(modalRef.current);

    // Restore focus on close
    return () => {
      cleanup();
      if (previousActiveElement) {
        moveFocusToElement(previousActiveElement);
      }
    };
  }, [isOpen, trapFocus, moveFocusToElement]);

  // Handle escape key
  useEffect(() => {
    const handleEscape = (event: KeyboardEvent) => {
      if (event.key === 'Escape') {
        onClose();
      }
    };

    if (isOpen) {
      document.addEventListener('keydown', handleEscape);
      return () => document.removeEventListener('keydown', handleEscape);
    }
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return (
    <div
      className="fixed inset-0 z-50 flex items-center justify-center"
      role="dialog"
      aria-modal="true"
      aria-labelledby={titleId}
    >
      {/* Backdrop */}
      <div
        className="fixed inset-0 bg-black bg-opacity-50"
        onClick={onClose}
        aria-hidden="true"
      />
      
      {/* Modal Content */}
      <div
        ref={modalRef}
        className="relative bg-white rounded-lg shadow-xl max-w-md w-full mx-4 p-6"
        tabIndex={-1}
      >
        <header className="flex items-center justify-between mb-4">
          <h2 id={titleId} className="text-xl font-semibold text-gray-900">
            {title}
          </h2>
          
          <button
            onClick={onClose}
            className="text-gray-400 hover:text-gray-600 focus:outline-none focus:ring-2 focus:ring-primary-500 rounded"
            aria-label="Close modal"
          >
            <XMarkIcon className="w-6 h-6" aria-hidden="true" />
          </button>
        </header>
        
        <div>{children}</div>
      </div>
    </div>
  );
};
```

### Keyboard Shortcuts
```typescript
// hooks/useKeyboardShortcuts.ts
export const useKeyboardShortcuts = () => {
  useEffect(() => {
    const handleKeyDown = (event: KeyboardEvent) => {
      // Skip if user is typing in an input
      const target = event.target as HTMLElement;
      if (target.tagName === 'INPUT' || target.tagName === 'TEXTAREA' || target.contentEditable === 'true') {
        return;
      }

      // Global shortcuts
      if (event.metaKey || event.ctrlKey) {
        switch (event.key) {
          case 'k': // Cmd/Ctrl + K - Open search
            event.preventDefault();
            // Trigger search modal
            break;
            
          case '/': // Cmd/Ctrl + / - Show keyboard shortcuts
            event.preventDefault();
            // Show shortcuts modal
            break;
        }
      }

      // Navigation shortcuts
      switch (event.key) {
        case '/': // Focus search
          if (!event.metaKey && !event.ctrlKey) {
            event.preventDefault();
            const searchInput = document.querySelector('[role="searchbox"]') as HTMLElement;
            if (searchInput) searchInput.focus();
          }
          break;
          
        case 'h': // Go home
          if (!event.metaKey && !event.ctrlKey) {
            window.location.href = '/';
          }
          break;
          
        case 'r': // Go to recipes
          if (!event.metaKey && !event.ctrlKey) {
            window.location.href = '/recipes';
          }
          break;
          
        case 'm': // Go to menu planning
          if (!event.metaKey && !event.ctrlKey) {
            window.location.href = '/menu-planning';
          }
          break;
      }
    };

    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, []);
};

// Keyboard shortcuts help modal
export const KeyboardShortcutsModal: React.FC<{
  isOpen: boolean;
  onClose: () => void;
}> = ({ isOpen, onClose }) => {
  const shortcuts = [
    { key: 'Ctrl/Cmd + K', action: 'Open search' },
    { key: 'Ctrl/Cmd + /', action: 'Show keyboard shortcuts' },
    { key: '/', action: 'Focus search' },
    { key: 'H', action: 'Go to home' },
    { key: 'R', action: 'Go to recipes' },
    { key: 'M', action: 'Go to menu planning' },
    { key: 'Esc', action: 'Close modal/cancel action' },
    { key: 'Tab', action: 'Navigate to next element' },
    { key: 'Shift + Tab', action: 'Navigate to previous element' },
  ];

  return (
    <AccessibleModal
      isOpen={isOpen}
      onClose={onClose}
      title="Keyboard Shortcuts"
    >
      <div className="space-y-4">
        <p className="text-sm text-gray-600">
          Use these keyboard shortcuts to navigate MealPrep more efficiently.
        </p>
        
        <div className="space-y-2">
          {shortcuts.map((shortcut, index) => (
            <div key={index} className="flex justify-between items-center">
              <kbd className="px-2 py-1 text-xs font-mono bg-gray-100 rounded border">
                {shortcut.key}
              </kbd>
              <span className="text-sm text-gray-700">{shortcut.action}</span>
            </div>
          ))}
        </div>
      </div>
    </AccessibleModal>
  );
};
```

## Screen Reader Support

### Screen Reader Optimizations
```typescript
// components/ui/ScreenReaderContent.tsx
export const ScreenReaderOnly: React.FC<{ children: React.ReactNode }> = ({ children }) => (
  <span className="sr-only">{children}</span>
);

// Enhanced recipe card for screen readers
export const ScreenReaderOptimizedRecipeCard: React.FC<{ recipe: Recipe }> = ({ recipe }) => {
  const totalTime = recipe.prepTime + recipe.cookTime;
  
  return (
    <article className="bg-white rounded-lg shadow border border-gray-200">
      <div className="p-6">
        {/* Image with descriptive alt text */}
        <img
          src={recipe.imageUrl}
          alt={`A delicious ${recipe.name} featuring ${recipe.ingredients.slice(0, 3).map(i => i.name).join(', ')}`}
          className="w-full h-48 object-cover rounded-md mb-4"
        />

        {/* Recipe title and metadata */}
        <h3 className="text-xl font-semibold text-gray-900 mb-2">
          {recipe.name}
          <ScreenReaderOnly>
            - Takes {totalTime} minutes total, serves {recipe.servings} people
          </ScreenReaderOnly>
        </h3>

        {/* Visual metadata with screen reader alternatives */}
        <div className="flex items-center text-sm text-gray-600 space-x-4 mb-3">
          <span>
            <ClockIcon className="w-4 h-4 inline mr-1" aria-hidden="true" />
            {recipe.prepTime}m prep
            <ScreenReaderOnly>, {recipe.cookTime} minutes cooking time</ScreenReaderOnly>
          </span>
          
          <span>
            <UsersIcon className="w-4 h-4 inline mr-1" aria-hidden="true" />
            {recipe.servings} servings
          </span>
          
          <span>
            <DollarSignIcon className="w-4 h-4 inline mr-1" aria-hidden="true" />
            ${recipe.estimatedCost.toFixed(2)}
            <ScreenReaderOnly> estimated cost</ScreenReaderOnly>
          </span>
        </div>

        {/* Recipe description */}
        <p className="text-gray-600 mb-4">{recipe.description}</p>

        {/* Dietary information for screen readers */}
        {recipe.dietaryRestrictions.length > 0 && (
          <div>
            <ScreenReaderOnly>
              Dietary information: This recipe is {recipe.dietaryRestrictions.join(', ')}.
            </ScreenReaderOnly>
            <div className="flex flex-wrap gap-2 mb-4" role="list" aria-label="Dietary restrictions">
              {recipe.dietaryRestrictions.map((restriction) => (
                <span
                  key={restriction}
                  role="listitem"
                  className="px-2 py-1 text-xs bg-green-100 text-green-800 rounded"
                >
                  {restriction}
                </span>
              ))}
            </div>
          </div>
        )}

        {/* Actions */}
        <div className="flex space-x-3">
          <button
            className="flex-1 bg-primary-600 text-white px-4 py-2 rounded hover:bg-primary-700 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:ring-offset-2"
            aria-describedby={`recipe-${recipe.id}-details`}
          >
            View Recipe
          </button>
          
          <button
            className="flex-1 border border-gray-300 text-gray-700 px-4 py-2 rounded hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:ring-offset-2"
            aria-label={`Add ${recipe.name} to meal plan`}
          >
            Add to Plan
          </button>
        </div>

        {/* Hidden details for screen readers */}
        <div id={`recipe-${recipe.id}-details`} className="sr-only">
          This recipe contains {recipe.ingredients.length} ingredients and has {recipe.instructions.length} preparation steps.
          Difficulty level: {recipe.difficulty}.
        </div>
      </div>
    </article>
  );
};
```

## Testing and Validation

### Accessibility Testing Tools
```typescript
// utils/accessibilityTesting.ts
export const runAccessibilityAudit = async (element?: HTMLElement) => {
  if (process.env.NODE_ENV !== 'development') return;

  try {
    // Use axe-core for automated testing
    const axe = await import('axe-core');
    
    const results = await axe.run(element || document.body, {
      tags: ['wcag2a', 'wcag2aa', 'wcag21aa'],
    });

    if (results.violations.length > 0) {
      console.group('?? Accessibility Violations');
      results.violations.forEach(violation => {
        console.error(violation.description);
        console.log('Impact:', violation.impact);
        console.log('Help:', violation.helpUrl);
        console.log('Nodes:', violation.nodes);
      });
      console.groupEnd();
    }

    if (results.passes.length > 0) {
      console.log('? Accessibility checks passed:', results.passes.length);
    }

    return results;
  } catch (error) {
    console.warn('Accessibility audit failed:', error);
  }
};

// React component for testing
export const AccessibilityTester: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (process.env.NODE_ENV === 'development' && containerRef.current) {
      // Run audit after component mounts and updates
      const timeoutId = setTimeout(() => {
        runAccessibilityAudit(containerRef.current!);
      }, 100);

      return () => clearTimeout(timeoutId);
    }
  });

  return <div ref={containerRef}>{children}</div>;
};
```

### Manual Testing Checklist
```typescript
// Testing checklist component
export const AccessibilityChecklist: React.FC = () => {
  const [checkedItems, setCheckedItems] = useState<Set<string>>(new Set());

  const checklistItems = [
    {
      id: 'keyboard-nav',
      category: 'Keyboard Navigation',
      items: [
        'All interactive elements are reachable via keyboard',
        'Tab order is logical and intuitive',
        'Focus indicators are clearly visible',
        'No keyboard traps exist',
        'Skip links work properly',
      ],
    },
    {
      id: 'screen-reader',
      category: 'Screen Reader Support',
      items: [
        'All images have appropriate alt text',
        'Form controls have associated labels',
        'Headings create a logical structure',
        'ARIA labels are meaningful',
        'Status messages are announced',
      ],
    },
    {
      id: 'visual-design',
      category: 'Visual Design',
      items: [
        'Color contrast meets WCAG AA standards',
        'Text can be zoomed to 200% without horizontal scrolling',
        'Content reflows properly at different viewport sizes',
        'Information is not conveyed by color alone',
        'Focus indicators have sufficient contrast',
      ],
    },
    {
      id: 'content',
      category: 'Content',
      items: [
        'Error messages are clear and helpful',
        'Instructions are provided where needed',
        'Language is simple and clear',
        'Acronyms are defined on first use',
        'Page titles are descriptive',
      ],
    },
  ];

  const toggleCheck = (itemId: string) => {
    const newChecked = new Set(checkedItems);
    if (newChecked.has(itemId)) {
      newChecked.delete(itemId);
    } else {
      newChecked.add(itemId);
    }
    setCheckedItems(newChecked);
  };

  return (
    <div className="max-w-4xl mx-auto p-6">
      <h1 className="text-3xl font-bold text-gray-900 mb-6">
        Accessibility Testing Checklist
      </h1>
      
      {checklistItems.map((category) => (
        <section key={category.id} className="mb-8">
          <h2 className="text-2xl font-semibold text-gray-900 mb-4">
            {category.category}
          </h2>
          
          <ul className="space-y-3">
            {category.items.map((item, index) => {
              const itemId = `${category.id}-${index}`;
              const isChecked = checkedItems.has(itemId);
              
              return (
                <li key={itemId} className="flex items-start">
                  <input
                    type="checkbox"
                    id={itemId}
                    checked={isChecked}
                    onChange={() => toggleCheck(itemId)}
                    className="mt-1 h-4 w-4 text-primary-600 focus:ring-primary-500 border-gray-300 rounded"
                  />
                  <label
                    htmlFor={itemId}
                    className={cn(
                      'ml-3 text-sm',
                      isChecked ? 'text-gray-500 line-through' : 'text-gray-700'
                    )}
                  >
                    {item}
                  </label>
                </li>
              );
            })}
          </ul>
        </section>
      ))}
      
      <div className="mt-8 p-4 bg-blue-50 rounded-lg">
        <h3 className="text-lg font-medium text-blue-900 mb-2">
          Testing Progress
        </h3>
        <p className="text-blue-700">
          {checkedItems.size} of {checklistItems.reduce((sum, cat) => sum + cat.items.length, 0)} items completed
        </p>
      </div>
    </div>
  );
};
```

This comprehensive accessibility guide ensures the MealPrep application is usable by all users, including those with disabilities, and meets modern web accessibility standards.

*This accessibility guide should be updated regularly as WCAG standards evolve and new accessibility best practices emerge.*