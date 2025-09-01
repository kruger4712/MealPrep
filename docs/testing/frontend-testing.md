# Frontend Testing Guide

## Overview
Comprehensive guide for testing React components and frontend functionality in the MealPrep application, covering unit testing, integration testing, visual testing, accessibility testing, and end-to-end testing strategies.

## Frontend Testing Strategy

### Testing Philosophy
- **User-Centric Testing**: Test from the user's perspective
- **Confidence Over Coverage**: Focus on critical user flows
- **Fast Feedback**: Prioritize tests that run quickly and provide immediate feedback
- **Maintainable Tests**: Write tests that are easy to understand and maintain
- **Real Browser Testing**: Test in actual browser environments when needed

### Testing Pyramid for Frontend
```
    E2E Tests (5%)
   ?? Visual Tests (10%)
  ?? Integration Tests (25%)
 ?? Unit Tests (60%)
```

### Testing Types
1. **Unit Tests**: Individual component testing in isolation
2. **Integration Tests**: Component interactions and data flow
3. **Visual Tests**: UI appearance and layout validation  
4. **Accessibility Tests**: WCAG compliance and screen reader compatibility
5. **End-to-End Tests**: Complete user journey testing

---

## Testing Framework Setup

### Core Testing Dependencies
```json
{
  "devDependencies": {
    "@testing-library/react": "^13.4.0",
    "@testing-library/jest-dom": "^5.16.5",
    "@testing-library/user-event": "^14.4.3",
    "@testing-library/react-hooks": "^8.0.1",
    "jest": "^27.5.1",
    "jest-environment-jsdom": "^27.5.1",
    "@types/jest": "^27.5.2",
    "msw": "^1.3.2",
    "jest-canvas-mock": "^2.4.0",
    "@storybook/testing-library": "^0.2.2",
    "axe-core": "^4.7.2",
    "jest-axe": "^7.0.1",
    "@percy/cli": "^1.15.0",
    "@percy/storybook": "^4.3.5"
  }
}
```

### Jest Configuration (jest.config.js)
```javascript
module.exports = {
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/src/setupTests.ts'],
  moduleNameMapping: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@components/(.*)$': '<rootDir>/src/components/$1',
    '^@pages/(.*)$': '<rootDir>/src/pages/$1',
    '^@services/(.*)$': '<rootDir>/src/services/$1',
    '^@utils/(.*)$': '<rootDir>/src/utils/$1',
    '^@types/(.*)$': '<rootDir>/src/types/$1',
    '^@hooks/(.*)$': '<rootDir>/src/hooks/$1',
    '^@contexts/(.*)$': '<rootDir>/src/contexts/$1',
    '\\.(css|less|scss|sass)$': 'identity-obj-proxy'
  },
  transform: {
    '^.+\\.(ts|tsx)$': 'ts-jest'
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/index.tsx',
    '!src/reportWebVitals.ts',
    '!src/**/*.stories.{ts,tsx}',
    '!src/**/*.test.{ts,tsx}'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    },
    './src/components/': {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90
    }
  },
  testMatch: [
    '<rootDir>/src/**/__tests__/**/*.{ts,tsx}',
    '<rootDir>/src/**/*.{test,spec}.{ts,tsx}'
  ]
};
```

### Test Setup (setupTests.ts)
```typescript
import '@testing-library/jest-dom';
import 'jest-canvas-mock';
import { server } from './mocks/server';
import { toHaveNoViolations } from 'jest-axe';

// Extend Jest matchers
expect.extend(toHaveNoViolations);

// Mock IntersectionObserver
global.IntersectionObserver = class IntersectionObserver {
  constructor() {}
  disconnect() {}
  observe() {}
  unobserve() {}
};

// Mock ResizeObserver
global.ResizeObserver = class ResizeObserver {
  constructor() {}
  disconnect() {}
  observe() {}
  unobserve() {}
};

// Setup MSW server
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// Mock window.matchMedia
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: jest.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: jest.fn(),
    removeListener: jest.fn(),
    addEventListener: jest.fn(),
    removeEventListener: jest.fn(),
    dispatchEvent: jest.fn(),
  })),
});

// Mock local storage
const localStorageMock = {
  getItem: jest.fn(),
  setItem: jest.fn(),
  removeItem: jest.fn(),
  clear: jest.fn(),
};
global.localStorage = localStorageMock;
```

---

## Component Unit Testing

### Testing Utilities

#### Custom Render Function
```typescript
// src/test-utils/test-utils.tsx
import React, { ReactElement } from 'react';
import { render, RenderOptions } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { BrowserRouter } from 'react-router-dom';
import { ThemeProvider } from '@mui/material/styles';
import { AuthProvider } from '@contexts/AuthContext';
import { NotificationProvider } from '@contexts/NotificationContext';
import { theme } from '@styles/theme';

interface AllTheProvidersProps {
  children: React.ReactNode;
}

const AllTheProviders: React.FC<AllTheProvidersProps> = ({ children }) => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
        cacheTime: 0,
      },
      mutations: {
        retry: false,
      },
    },
  });

  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <ThemeProvider theme={theme}>
          <AuthProvider>
            <NotificationProvider>
              {children}
            </NotificationProvider>
          </AuthProvider>
        </ThemeProvider>
      </BrowserRouter>
    </QueryClientProvider>
  );
};

const customRender = (
  ui: ReactElement,
  options?: Omit<RenderOptions, 'wrapper'>
) => render(ui, { wrapper: AllTheProviders, ...options });

// Re-export everything
export * from '@testing-library/react';
export { customRender as render };
export { userEvent } from '@testing-library/user-event';
```

### Recipe Card Component Testing

#### Recipe Card Component Tests
```typescript
// src/components/RecipeCard/RecipeCard.test.tsx
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@/test-utils/test-utils';
import userEvent from '@testing-library/user-event';
import { axe, toHaveNoViolations } from 'jest-axe';
import { RecipeCard } from './RecipeCard';
import { Recipe } from '@types/Recipe';

expect.extend(toHaveNoViolations);

const mockRecipe: Recipe = {
  id: 1,
  name: 'Chicken Parmesan',
  description: 'Crispy breaded chicken with marinara sauce and melted cheese',
  prepTimeMinutes: 20,
  cookTimeMinutes: 25,
  servingSize: 4,
  difficulty: 'Medium',
  cuisine: 'Italian',
  averageRating: 4.5,
  ratingCount: 12,
  imageUrl: 'https://example.com/chicken-parmesan.jpg',
  isAiGenerated: false,
  familyFitScore: 8,
  ingredients: [
    { id: 1, name: 'Chicken Breast', quantity: 4, unit: 'pieces' },
    { id: 2, name: 'Breadcrumbs', quantity: 1, unit: 'cup' }
  ],
  tags: ['dinner', 'italian', 'comfort-food']
};

describe('RecipeCard Component', () => {
  const mockOnFavorite = jest.fn();
  const mockOnRate = jest.fn();
  const mockOnClick = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders recipe information correctly', () => {
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    // Test basic recipe information
    expect(screen.getByText('Chicken Parmesan')).toBeInTheDocument();
    expect(screen.getByText('Crispy breaded chicken with marinara sauce and melted cheese')).toBeInTheDocument();
    expect(screen.getByText('20 min')).toBeInTheDocument();
    expect(screen.getByText('25 min')).toBeInTheDocument();
    expect(screen.getByText('Serves 4')).toBeInTheDocument();
    expect(screen.getByText('Medium')).toBeInTheDocument();
    expect(screen.getByText('Italian')).toBeInTheDocument();
  });

  it('displays rating information correctly', () => {
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    expect(screen.getByText('4.5')).toBeInTheDocument();
    expect(screen.getByText('(12 reviews)')).toBeInTheDocument();
  });

  it('shows family fit score when provided', () => {
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    expect(screen.getByText('Family Fit: 8/10')).toBeInTheDocument();
  });

  it('displays AI badge when recipe is AI generated', () => {
    const aiRecipe = { ...mockRecipe, isAiGenerated: true };
    
    render(
      <RecipeCard
        recipe={aiRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    expect(screen.getByText('AI Generated')).toBeInTheDocument();
    expect(screen.getByTestId('ai-badge')).toHaveClass('ai-generated-badge');
  });

  it('handles recipe image correctly', () => {
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    const image = screen.getByRole('img', { name: /chicken parmesan/i });
    expect(image).toHaveAttribute('src', mockRecipe.imageUrl);
    expect(image).toHaveAttribute('alt', 'Chicken Parmesan');
  });

  it('handles missing image gracefully', () => {
    const recipeWithoutImage = { ...mockRecipe, imageUrl: undefined };
    
    render(
      <RecipeCard
        recipe={recipeWithoutImage}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    const image = screen.getByRole('img');
    expect(image).toHaveAttribute('src', expect.stringContaining('placeholder'));
  });

  it('calls onClick when card is clicked', async () => {
    const user = userEvent.setup();
    
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    const card = screen.getByTestId('recipe-card');
    await user.click(card);

    expect(mockOnClick).toHaveBeenCalledWith(mockRecipe);
  });

  it('calls onFavorite when favorite button is clicked', async () => {
    const user = userEvent.setup();
    
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    const favoriteButton = screen.getByTestId('favorite-button');
    await user.click(favoriteButton);

    expect(mockOnFavorite).toHaveBeenCalledWith(mockRecipe.id);
    expect(mockOnClick).not.toHaveBeenCalled(); // Should not trigger card click
  });

  it('prevents event bubbling on favorite button click', async () => {
    const user = userEvent.setup();
    
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    const favoriteButton = screen.getByTestId('favorite-button');
    await user.click(favoriteButton);

    expect(mockOnFavorite).toHaveBeenCalledTimes(1);
    expect(mockOnClick).not.toHaveBeenCalled();
  });

  it('displays recipe tags correctly', () => {
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    expect(screen.getByText('dinner')).toBeInTheDocument();
    expect(screen.getByText('italian')).toBeInTheDocument();
    expect(screen.getByText('comfort-food')).toBeInTheDocument();
  });

  // Accessibility Tests
  it('has no accessibility violations', async () => {
    const { container } = render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('has proper ARIA labels', () => {
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    expect(screen.getByRole('button', { name: /add to favorites/i })).toBeInTheDocument();
    expect(screen.getByRole('img')).toHaveAccessibleName();
  });

  it('supports keyboard navigation', async () => {
    const user = userEvent.setup();
    
    render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    const favoriteButton = screen.getByTestId('favorite-button');
    favoriteButton.focus();
    expect(favoriteButton).toHaveFocus();

    await user.keyboard('{Enter}');
    expect(mockOnFavorite).toHaveBeenCalledWith(mockRecipe.id);
  });

  // Visual Regression Tests
  it('matches snapshot', () => {
    const { container } = render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    expect(container.firstChild).toMatchSnapshot();
  });

  it('matches snapshot for AI generated recipe', () => {
    const aiRecipe = { ...mockRecipe, isAiGenerated: true };
    
    const { container } = render(
      <RecipeCard
        recipe={aiRecipe}
        onFavorite={mockOnFavorite}
        onRate={mockOnRate}
        onClick={mockOnClick}
      />
    );

    expect(container.firstChild).toMatchSnapshot();
  });
});
```

### Recipe Form Component Testing

#### Recipe Form Integration Tests
```typescript
// src/components/RecipeForm/RecipeForm.test.tsx
import React from 'react';
import { render, screen, waitFor } from '@/test-utils/test-utils';
import userEvent from '@testing-library/user-event';
import { RecipeForm } from './RecipeForm';
import { CreateRecipeDto } from '@types/Recipe';

describe('RecipeForm Component', () => {
  const mockOnSubmit = jest.fn();
  const mockOnCancel = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders all form fields', () => {
    render(<RecipeForm onSubmit={mockOnSubmit} onCancel={mockOnCancel} />);

    expect(screen.getByLabelText(/recipe name/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/description/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/instructions/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/prep time/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/cook time/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/serving size/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/difficulty/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/cuisine/i)).toBeInTheDocument();
  });

  it('submits form with valid data', async () => {
    const user = userEvent.setup();
    
    render(<RecipeForm onSubmit={mockOnSubmit} onCancel={mockOnCancel} />);

    // Fill out the form
    await user.type(screen.getByLabelText(/recipe name/i), 'Test Recipe');
    await user.type(screen.getByLabelText(/description/i), 'A delicious test recipe');
    await user.type(screen.getByLabelText(/instructions/i), '1. Mix ingredients\n2. Cook\n3. Serve');
    await user.type(screen.getByLabelText(/prep time/i), '15');
    await user.type(screen.getByLabelText(/cook time/i), '30');
    await user.type(screen.getByLabelText(/serving size/i), '4');
    await user.selectOptions(screen.getByLabelText(/difficulty/i), 'Medium');
    await user.selectOptions(screen.getByLabelText(/cuisine/i), 'Italian');

    // Add an ingredient
    await user.click(screen.getByText(/add ingredient/i));
    await user.type(screen.getByLabelText(/ingredient name/i), 'Chicken');
    await user.type(screen.getByLabelText(/quantity/i), '2');
    await user.type(screen.getByLabelText(/unit/i), 'lbs');

    // Submit the form
    await user.click(screen.getByRole('button', { name: /create recipe/i }));

    await waitFor(() => {
      expect(mockOnSubmit).toHaveBeenCalledWith(
        expect.objectContaining({
          name: 'Test Recipe',
          description: 'A delicious test recipe',
          instructions: '1. Mix ingredients\n2. Cook\n3. Serve',
          prepTimeMinutes: 15,
          cookTimeMinutes: 30,
          servingSize: 4,
          difficultyLevel: 'Medium',
          cuisine: 'Italian',
          ingredients: expect.arrayContaining([
            expect.objectContaining({
              ingredientName: 'Chicken',
              quantity: 2,
              unit: 'lbs'
            })
          ])
        })
      );
    });
  });

  it('shows validation errors for invalid input', async () => {
    const user = userEvent.setup();
    
    render(<RecipeForm onSubmit={mockOnSubmit} onCancel={mockOnCancel} />);

    // Try to submit empty form
    await user.click(screen.getByRole('button', { name: /create recipe/i }));

    await waitFor(() => {
      expect(screen.getByText(/recipe name is required/i)).toBeInTheDocument();
      expect(screen.getByText(/instructions are required/i)).toBeInTheDocument();
    });

    expect(mockOnSubmit).not.toHaveBeenCalled();
  });

  it('validates numeric fields', async () => {
    const user = userEvent.setup();
    
    render(<RecipeForm onSubmit={mockOnSubmit} onCancel={mockOnCancel} />);

    // Enter invalid numeric values
    await user.type(screen.getByLabelText(/prep time/i), '-5');
    await user.type(screen.getByLabelText(/cook time/i), '0');
    await user.type(screen.getByLabelText(/serving size/i), '0');

    await user.click(screen.getByRole('button', { name: /create recipe/i }));

    await waitFor(() => {
      expect(screen.getByText(/prep time must be positive/i)).toBeInTheDocument();
      expect(screen.getByText(/cook time must be greater than 0/i)).toBeInTheDocument();
      expect(screen.getByText(/serving size must be at least 1/i)).toBeInTheDocument();
    });
  });

  it('manages ingredients list correctly', async () => {
    const user = userEvent.setup();
    
    render(<RecipeForm onSubmit={mockOnSubmit} onCancel={mockOnCancel} />);

    // Add first ingredient
    await user.click(screen.getByText(/add ingredient/i));
    const ingredientInputs = screen.getAllByLabelText(/ingredient name/i);
    await user.type(ingredientInputs[0], 'Chicken');

    // Add second ingredient
    await user.click(screen.getByText(/add ingredient/i));
    const updatedIngredientInputs = screen.getAllByLabelText(/ingredient name/i);
    expect(updatedIngredientInputs).toHaveLength(2);

    // Remove first ingredient
    const removeButtons = screen.getAllByText(/remove ingredient/i);
    await user.click(removeButtons[0]);

    // Should have one ingredient left
    const finalIngredientInputs = screen.getAllByLabelText(/ingredient name/i);
    expect(finalIngredientInputs).toHaveLength(1);
  });

  it('calls onCancel when cancel button is clicked', async () => {
    const user = userEvent.setup();
    
    render(<RecipeForm onSubmit={mockOnSubmit} onCancel={mockOnCancel} />);

    await user.click(screen.getByRole('button', { name: /cancel/i }));

    expect(mockOnCancel).toHaveBeenCalled();
  });

  it('populates form for editing existing recipe', () => {
    const existingRecipe = {
      id: 1,
      name: 'Existing Recipe',
      description: 'An existing recipe',
      instructions: 'Existing instructions',
      prepTimeMinutes: 20,
      cookTimeMinutes: 30,
      servingSize: 4,
      difficultyLevel: 'Easy',
      cuisine: 'American',
      ingredients: [
        { ingredientName: 'Ingredient 1', quantity: 1, unit: 'cup' }
      ]
    };

    render(
      <RecipeForm 
        initialRecipe={existingRecipe}
        onSubmit={mockOnSubmit} 
        onCancel={mockOnCancel} 
      />
    );

    expect(screen.getByDisplayValue('Existing Recipe')).toBeInTheDocument();
    expect(screen.getByDisplayValue('An existing recipe')).toBeInTheDocument();
    expect(screen.getByDisplayValue('20')).toBeInTheDocument();
    expect(screen.getByDisplayValue('30')).toBeInTheDocument();
  });
});
```

---

## Custom Hooks Testing

### useRecipes Hook Testing
```typescript
// src/hooks/useRecipes/useRecipes.test.tsx
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useRecipes } from './useRecipes';
import { recipeService } from '@services/recipeService';
import { Recipe } from '@types/Recipe';

// Mock the service
jest.mock('@services/recipeService');
const mockedRecipeService = recipeService as jest.Mocked<typeof recipeService>;

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false, cacheTime: 0 },
      mutations: { retry: false },
    },
  });
  
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
};

const mockRecipes: Recipe[] = [
  {
    id: 1,
    name: 'Test Recipe 1',
    description: 'A test recipe',
    prepTimeMinutes: 15,
    cookTimeMinutes: 30,
    servingSize: 4,
    difficulty: 'Easy',
    cuisine: 'Italian'
  },
  {
    id: 2,
    name: 'Test Recipe 2',
    description: 'Another test recipe',
    prepTimeMinutes: 20,
    cookTimeMinutes: 25,
    servingSize: 2,
    difficulty: 'Medium',
    cuisine: 'Mexican'
  }
];

describe('useRecipes Hook', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('loads recipes successfully', async () => {
    mockedRecipeService.searchRecipes.mockResolvedValue(mockRecipes);

    const { result } = renderHook(() => useRecipes(), {
      wrapper: createWrapper(),
    });

    expect(result.current.loading).toBe(true);
    expect(result.current.recipes).toEqual([]);

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.recipes).toEqual(mockRecipes);
    expect(result.current.error).toBeNull();
  });

  it('handles search with criteria', async () => {
    const searchCriteria = { searchTerm: 'pasta', cuisine: 'Italian' };
    mockedRecipeService.searchRecipes.mockResolvedValue([mockRecipes[0]]);

    const { result } = renderHook(() => useRecipes(searchCriteria), {
      wrapper: createWrapper(),
    });

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(mockedRecipeService.searchRecipes).toHaveBeenCalledWith(searchCriteria);
    expect(result.current.recipes).toEqual([mockRecipes[0]]);
  });

  it('handles errors correctly', async () => {
    const errorMessage = 'Failed to load recipes';
    mockedRecipeService.searchRecipes.mockRejectedValue(new Error(errorMessage));

    const { result } = renderHook(() => useRecipes(), {
      wrapper: createWrapper(),
    });

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.error).toBe(errorMessage);
    expect(result.current.recipes).toEqual([]);
  });

  it('refetches data when criteria changes', async () => {
    mockedRecipeService.searchRecipes
      .mockResolvedValueOnce(mockRecipes)
      .mockResolvedValueOnce([mockRecipes[0]]);

    const { result, rerender } = renderHook(
      ({ criteria }) => useRecipes(criteria),
      {
        wrapper: createWrapper(),
        initialProps: { criteria: undefined },
      }
    );

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.recipes).toEqual(mockRecipes);

    // Change search criteria
    rerender({ criteria: { searchTerm: 'pasta' } });

    await waitFor(() => {
      expect(mockedRecipeService.searchRecipes).toHaveBeenCalledTimes(2);
    });

    expect(mockedRecipeService.searchRecipes).toHaveBeenLastCalledWith({ searchTerm: 'pasta' });
  });
});
```

### useAuth Hook Testing
```typescript
// src/hooks/useAuth/useAuth.test.tsx
import { renderHook, act, waitFor } from '@testing-library/react';
import { useAuth } from './useAuth';
import { AuthProvider } from '@contexts/AuthContext';
import { authService } from '@services/authService';

jest.mock('@services/authService');
const mockedAuthService = authService as jest.Mocked<typeof authService>;

const createWrapper = ({ children }: { children: React.ReactNode }) => (
  <AuthProvider>{children}</AuthProvider>
);

describe('useAuth Hook', () => {
  beforeEach(() => {
    jest.clearAllMocks();
    localStorage.clear();
  });

  it('initializes with no user when not authenticated', () => {
    const { result } = renderHook(() => useAuth(), {
      wrapper: createWrapper,
    });

    expect(result.current.user).toBeNull();
    expect(result.current.isAuthenticated).toBe(false);
    expect(result.current.loading).toBe(false);
  });

  it('logs in user successfully', async () => {
    const mockUser = {
      id: 1,
      email: 'test@example.com',
      firstName: 'Test',
      lastName: 'User'
    };

    mockedAuthService.login.mockResolvedValue({
      user: mockUser,
      token: 'mock-jwt-token',
      expiresAt: new Date(Date.now() + 3600000).toISOString()
    });

    const { result } = renderHook(() => useAuth(), {
      wrapper: createWrapper,
    });

    await act(async () => {
      await result.current.login('test@example.com', 'password');
    });

    expect(result.current.user).toEqual(mockUser);
    expect(result.current.isAuthenticated).toBe(true);
    expect(mockedAuthService.login).toHaveBeenCalledWith('test@example.com', 'password');
  });

  it('handles login error', async () => {
    mockedAuthService.login.mockRejectedValue(new Error('Invalid credentials'));

    const { result } = renderHook(() => useAuth(), {
      wrapper: createWrapper,
    });

    await act(async () => {
      try {
        await result.current.login('test@example.com', 'wrong-password');
      } catch (error) {
        expect(error.message).toBe('Invalid credentials');
      }
    });

    expect(result.current.user).toBeNull();
    expect(result.current.isAuthenticated).toBe(false);
  });

  it('logs out user successfully', async () => {
    // First login
    const mockUser = {
      id: 1,
      email: 'test@example.com',
      firstName: 'Test',
      lastName: 'User'
    };

    mockedAuthService.login.mockResolvedValue({
      user: mockUser,
      token: 'mock-jwt-token',
      expiresAt: new Date(Date.now() + 3600000).toISOString()
    });

    const { result } = renderHook(() => useAuth(), {
      wrapper: createWrapper,
    });

    await act(async () => {
      await result.current.login('test@example.com', 'password');
    });

    expect(result.current.isAuthenticated).toBe(true);

    // Then logout
    await act(async () => {
      result.current.logout();
    });

    expect(result.current.user).toBeNull();
    expect(result.current.isAuthenticated).toBe(false);
  });
});
```

---

## Page Component Testing

### Recipe List Page Testing
```typescript
// src/pages/RecipeListPage/RecipeListPage.test.tsx
import React from 'react';
import { render, screen, waitFor } from '@/test-utils/test-utils';
import userEvent from '@testing-library/user-event';
import { RecipeListPage } from './RecipeListPage';
import { recipeService } from '@services/recipeService';

jest.mock('@services/recipeService');
const mockedRecipeService = recipeService as jest.Mocked<typeof recipeService>;

const mockRecipes = [
  {
    id: 1,
    name: 'Pasta Carbonara',
    description: 'Classic Italian pasta dish',
    prepTimeMinutes: 15,
    cookTimeMinutes: 20,
    servingSize: 4,
    difficulty: 'Medium',
    cuisine: 'Italian'
  },
  {
    id: 2,
    name: 'Chicken Curry',
    description: 'Spicy Indian curry',
    prepTimeMinutes: 30,
    cookTimeMinutes: 45,
    servingSize: 6,
    difficulty: 'Hard',
    cuisine: 'Indian'
  }
];

describe('RecipeListPage', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders recipe list page with recipes', async () => {
    mockedRecipeService.searchRecipes.mockResolvedValue(mockRecipes);

    render(<RecipeListPage />);

    expect(screen.getByText(/my recipes/i)).toBeInTheDocument();
    
    await waitFor(() => {
      expect(screen.getByText('Pasta Carbonara')).toBeInTheDocument();
      expect(screen.getByText('Chicken Curry')).toBeInTheDocument();
    });
  });

  it('shows loading state initially', () => {
    mockedRecipeService.searchRecipes.mockImplementation(
      () => new Promise(resolve => setTimeout(() => resolve(mockRecipes), 1000))
    );

    render(<RecipeListPage />);

    expect(screen.getByTestId('loading-spinner')).toBeInTheDocument();
  });

  it('shows empty state when no recipes found', async () => {
    mockedRecipeService.searchRecipes.mockResolvedValue([]);

    render(<RecipeListPage />);

    await waitFor(() => {
      expect(screen.getByText(/no recipes found/i)).toBeInTheDocument();
      expect(screen.getByText(/create your first recipe/i)).toBeInTheDocument();
    });
  });

  it('handles search functionality', async () => {
    const user = userEvent.setup();
    mockedRecipeService.searchRecipes
      .mockResolvedValueOnce(mockRecipes)
      .mockResolvedValueOnce([mockRecipes[0]]);

    render(<RecipeListPage />);

    await waitFor(() => {
      expect(screen.getByText('Pasta Carbonara')).toBeInTheDocument();
    });

    // Perform search
    const searchInput = screen.getByPlaceholderText(/search recipes/i);
    await user.type(searchInput, 'pasta');
    await user.click(screen.getByRole('button', { name: /search/i }));

    await waitFor(() => {
      expect(mockedRecipeService.searchRecipes).toHaveBeenCalledWith(
        expect.objectContaining({ searchTerm: 'pasta' })
      );
    });
  });

  it('handles error state', async () => {
    mockedRecipeService.searchRecipes.mockRejectedValue(new Error('Network error'));

    render(<RecipeListPage />);

    await waitFor(() => {
      expect(screen.getByText(/failed to load recipes/i)).toBeInTheDocument();
      expect(screen.getByRole('button', { name: /retry/i })).toBeInTheDocument();
    });
  });

  it('retries loading on retry button click', async () => {
    const user = userEvent.setup();
    mockedRecipeService.searchRecipes
      .mockRejectedValueOnce(new Error('Network error'))
      .mockResolvedValueOnce(mockRecipes);

    render(<RecipeListPage />);

    await waitFor(() => {
      expect(screen.getByText(/failed to load recipes/i)).toBeInTheDocument();
    });

    await user.click(screen.getByRole('button', { name: /retry/i }));

    await waitFor(() => {
      expect(screen.getByText('Pasta Carbonara')).toBeInTheDocument();
    });
  });

  it('navigates to recipe detail on recipe click', async () => {
    const user = userEvent.setup();
    mockedRecipeService.searchRecipes.mockResolvedValue(mockRecipes);

    render(<RecipeListPage />);

    await waitFor(() => {
      expect(screen.getByText('Pasta Carbonara')).toBeInTheDocument();
    });

    const recipeCard = screen.getByTestId('recipe-card-1');
    await user.click(recipeCard);

    // Check if navigation occurred (would be mocked in actual test)
    expect(window.location.pathname).toBe('/recipes/1');
  });
});
```

---

## Visual Testing with Storybook

### Storybook Component Stories
```typescript
// src/components/RecipeCard/RecipeCard.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { RecipeCard } from './RecipeCard';
import { Recipe } from '@types/Recipe';

const mockRecipe: Recipe = {
  id: 1,
  name: 'Chicken Parmesan',
  description: 'Crispy breaded chicken with marinara sauce and melted cheese',
  prepTimeMinutes: 20,
  cookTimeMinutes: 25,
  servingSize: 4,
  difficulty: 'Medium',
  cuisine: 'Italian',
  averageRating: 4.5,
  ratingCount: 12,
  imageUrl: 'https://images.unsplash.com/photo-1565299624946-b28f40a0ca4b',
  isAiGenerated: false,
  familyFitScore: 8,
  ingredients: [],
  tags: ['dinner', 'italian', 'comfort-food']
};

const meta: Meta<typeof RecipeCard> = {
  title: 'Components/RecipeCard',
  component: RecipeCard,
  parameters: {
    layout: 'centered',
    docs: {
      description: {
        component: 'Recipe card component for displaying recipe information in lists and grids.'
      }
    }
  },
  argTypes: {
    onFavorite: { action: 'favorited' },
    onRate: { action: 'rated' },
    onClick: { action: 'clicked' }
  }
};

export default meta;
type Story = StoryObj<typeof meta>;

export const Default: Story = {
  args: {
    recipe: mockRecipe
  }
};

export const AIGenerated: Story = {
  args: {
    recipe: {
      ...mockRecipe,
      isAiGenerated: true,
      name: 'AI Suggested Pasta Primavera'
    }
  }
};

export const NoImage: Story = {
  args: {
    recipe: {
      ...mockRecipe,
      imageUrl: undefined
    }
  }
};

export const LongDescription: Story = {
  args: {
    recipe: {
      ...mockRecipe,
      description: 'This is a very long description that might wrap to multiple lines and we need to see how it looks in the card layout. It should handle text overflow gracefully and maintain good visual hierarchy.'
    }
  }
};

export const HighDifficulty: Story = {
  args: {
    recipe: {
      ...mockRecipe,
      difficulty: 'Hard',
      prepTimeMinutes: 45,
      cookTimeMinutes: 90
    }
  }
};

export const Loading: Story = {
  args: {
    recipe: mockRecipe,
    loading: true
  }
};
```

### Visual Regression Testing with Percy
```javascript
// .percy.yml
version: 2
discovery:
  allowed-hostnames:
    - localhost
  network-idle-timeout: 750
static-snapshots:
  path: storybook-static
  base-url: /
  snapshot-files: '**/*.html'
  ignore-files: 'iframe.html'
```

```javascript
// scripts/visual-tests.js
const { execSync } = require('child_process');

// Build Storybook
console.log('Building Storybook...');
execSync('npm run build-storybook', { stdio: 'inherit' });

// Run Percy visual tests
console.log('Running visual regression tests...');
execSync('percy storybook storybook-static', { stdio: 'inherit' });
```

---

## Accessibility Testing

### Automated Accessibility Testing
```typescript
// src/components/RecipeCard/RecipeCard.a11y.test.tsx
import React from 'react';
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';
import { RecipeCard } from './RecipeCard';

expect.extend(toHaveNoViolations);

describe('RecipeCard Accessibility', () => {
  it('should not have any accessibility violations', async () => {
    const mockRecipe = {
      id: 1,
      name: 'Test Recipe',
      description: 'A test recipe',
      prepTimeMinutes: 15,
      cookTimeMinutes: 30,
      servingSize: 4,
      difficulty: 'Easy',
      cuisine: 'Italian'
    };

    const { container } = render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={() => {}}
        onRate={() => {}}
        onClick={() => {}}
      />
    );

    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('should have proper heading hierarchy', () => {
    const mockRecipe = {
      id: 1,
      name: 'Test Recipe',
      description: 'A test recipe',
      prepTimeMinutes: 15,
      cookTimeMinutes: 30,
      servingSize: 4,
      difficulty: 'Easy',
      cuisine: 'Italian'
    };

    const { container } = render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={() => {}}
        onRate={() => {}}
        onClick={() => {}}
      />
    );

    const headings = container.querySelectorAll('h1, h2, h3, h4, h5, h6');
    expect(headings).toHaveLength(1);
    expect(headings[0]).toHaveTextContent('Test Recipe');
  });

  it('should have proper color contrast', async () => {
    // This would typically use a library like axe-core
    // or custom contrast checking utilities
    const mockRecipe = {
      id: 1,
      name: 'Test Recipe',
      description: 'A test recipe',
      prepTimeMinutes: 15,
      cookTimeMinutes: 30,
      servingSize: 4,
      difficulty: 'Easy',
      cuisine: 'Italian'
    };

    const { container } = render(
      <RecipeCard
        recipe={mockRecipe}
        onFavorite={() => {}}
        onRate={() => {}}
        onClick={() => {}}
      />
    );

    const results = await axe(container, {
      rules: {
        'color-contrast': { enabled: true }
      }
    });

    expect(results).toHaveNoViolations();
  });
});
```

---

## Performance Testing

### Component Performance Testing
```typescript
// src/components/RecipeList/RecipeList.performance.test.tsx
import React from 'react';
import { render } from '@testing-library/react';
import { RecipeList } from './RecipeList';
import { Recipe } from '@types/Recipe';

const generateMockRecipes = (count: number): Recipe[] => {
  return Array.from({ length: count }, (_, index) => ({
    id: index + 1,
    name: `Recipe ${index + 1}`,
    description: `Description for recipe ${index + 1}`,
    prepTimeMinutes: Math.floor(Math.random() * 60) + 5,
    cookTimeMinutes: Math.floor(Math.random() * 120) + 10,
    servingSize: Math.floor(Math.random() * 8) + 1,
    difficulty: ['Easy', 'Medium', 'Hard'][Math.floor(Math.random() * 3)],
    cuisine: ['Italian', 'Mexican', 'Chinese', 'American'][Math.floor(Math.random() * 4)],
    averageRating: Math.random() * 5,
    ratingCount: Math.floor(Math.random() * 100),
    isAiGenerated: Math.random() > 0.5,
    familyFitScore: Math.floor(Math.random() * 10) + 1,
    ingredients: [],
    tags: []
  }));
};

describe('RecipeList Performance', () => {
  it('should render large lists efficiently', () => {
    const recipes = generateMockRecipes(1000);
    
    const startTime = performance.now();
    render(<RecipeList recipes={recipes} />);
    const endTime = performance.now();
    
    const renderTime = endTime - startTime;
    
    // Expect render time to be under 100ms for 1000 items
    expect(renderTime).toBeLessThan(100);
  });

  it('should handle rapid re-renders efficiently', () => {
    const recipes = generateMockRecipes(100);
    
    const { rerender } = render(<RecipeList recipes={recipes} />);
    
    const startTime = performance.now();
    
    // Simulate rapid re-renders
    for (let i = 0; i < 50; i++) {
      const updatedRecipes = generateMockRecipes(100);
      rerender(<RecipeList recipes={updatedRecipes} />);
    }
    
    const endTime = performance.now();
    const totalTime = endTime - startTime;
    
    // Expect 50 re-renders to complete in under 500ms
    expect(totalTime).toBeLessThan(500);
  });
});
```

### Memory Leak Testing
```typescript
// src/hooks/useRecipes/useRecipes.memory.test.tsx
import { renderHook } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useRecipes } from './useRecipes';

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false, cacheTime: 0 }
    }
  });
  
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
};

describe('useRecipes Memory Management', () => {
  it('should clean up properly on unmount', () => {
    const { unmount } = renderHook(() => useRecipes(), {
      wrapper: createWrapper()
    });

    // Record initial memory usage
    const initialMemory = performance.memory?.usedJSHeapSize || 0;

    // Unmount the hook
    unmount();

    // Force garbage collection if available
    if (global.gc) {
      global.gc();
    }

    // Check memory usage after cleanup
    const finalMemory = performance.memory?.usedJSHeapSize || 0;
    
    // Memory shouldn't increase significantly
    expect(finalMemory - initialMemory).toBeLessThan(1024 * 1024); // Less than 1MB
  });
});
```

---

## Test Organization and Best Practices

### Test File Organization
```
src/
??? components/
?   ??? RecipeCard/
?       ??? RecipeCard.tsx
?       ??? RecipeCard.test.tsx
?       ??? RecipeCard.stories.tsx
?       ??? RecipeCard.a11y.test.tsx
?       ??? __snapshots__/
?           ??? RecipeCard.test.tsx.snap
??? hooks/
?   ??? useRecipes/
?       ??? useRecipes.ts
?       ??? useRecipes.test.tsx
?       ??? useRecipes.memory.test.tsx
??? test-utils/
    ??? test-utils.tsx
    ??? mocks/
    ?   ??? server.ts
    ?   ??? handlers.ts
    ??? fixtures/
        ??? recipes.ts
```

### Testing Best Practices

#### 1. Test Structure and Naming
```typescript
describe('ComponentName', () => {
  describe('when condition', () => {
    it('should behavior', () => {
      // Test implementation
    });
  });
});
```

#### 2. Test Data Management
```typescript
// fixtures/recipes.ts
export const createMockRecipe = (overrides = {}): Recipe => ({
  id: 1,
  name: 'Default Recipe',
  description: 'Default description',
  prepTimeMinutes: 15,
  cookTimeMinutes: 30,
  servingSize: 4,
  difficulty: 'Easy',
  cuisine: 'Italian',
  ...overrides
});
```

#### 3. Async Testing Patterns
```typescript
// Wait for async operations
await waitFor(() => {
  expect(screen.getByText('Expected text')).toBeInTheDocument();
});

// Test loading states
expect(screen.getByTestId('loading-spinner')).toBeInTheDocument();

// Test error states
await waitFor(() => {
  expect(screen.getByText(/error message/i)).toBeInTheDocument();
});
```

#### 4. User Interaction Testing
```typescript
const user = userEvent.setup();

// Type in input
await user.type(screen.getByLabelText(/recipe name/i), 'New Recipe');

// Click button
await user.click(screen.getByRole('button', { name: /submit/i }));

// Select from dropdown
await user.selectOptions(screen.getByLabelText(/difficulty/i), 'Medium');
```

---

## Continuous Integration

### GitHub Actions Workflow
```yaml
name: Frontend Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run type checking
      run: npm run type-check
    
    - name: Run linting
      run: npm run lint
    
    - name: Run unit tests
      run: npm run test:coverage
    
    - name: Run accessibility tests
      run: npm run test:a11y
    
    - name: Upload coverage reports
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/lcov.info
    
    - name: Build Storybook
      run: npm run build-storybook
    
    - name: Run visual tests
      run: npm run test:visual
      env:
        PERCY_TOKEN: ${{ secrets.PERCY_TOKEN }}
```

---

*Last Updated: December 2024*  
*Frontend testing guide continuously updated with new testing patterns and best practices*