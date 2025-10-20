# TypeScript, React & MUI Stack: Good Practices & Implementation Guide

## Current Project Architecture Overview

**Stack:** React 17 + TypeScript 4.9 + MUI 5.10 + Redux Toolkit + React Query + Vite

**Key Files:**

- Configuration: `tsconfig.json`, `package.json`
- Main App: `src/App.tsx`, `src/index.tsx`
- State Management: `src/store.ts`, `src/reducers/shared.ts`
- Type Definitions: `src/types.ts`

##  Good Practices

### TypeScript

```typescript
// 1. Comprehensive Type Definitions
export interface User {
  type: UserType.Standard;
  id: number;
  countryCode: string;
  email: string;
  status: UserStatus;
  // ... well-defined properties
}

// 2. Union Types for Better Type Safety
export type TaskStatus = 'pending' | 'in-progress' | 'completed';
export type ProcessType = 'automated' | 'manual';

// 3. Proper Enum Usage
export enum SubscriptionTier {
  FREE = 'free',
  PREMIUM = 'premium',
  ENTERPRISE = 'enterprise',
}
```

### React Component Patterns

#### 1. **Functional Components with TypeScript Interfaces**

```typescript
export interface TagProps {
  interactive: boolean;
  color: string;
  onClick?: () => void;
  className?: string;
  children?: React.ReactNode;
}

export const Tag: React.FC<TagProps> = ({
  interactive,
  color,
  onClick = () => {},
  children,
  className,
}) => (
  <span
    className={classNames('Tag', { 'Tag--interactive': interactive }, className)}
    style={{ background: color }}
    onClick={(e) => {
      if (interactive) e.stopPropagation();
      onClick();
    }}
  >
    {children}
  </span>
);
```

#### 2. **Custom Hooks for Logic Separation**

```typescript
// Simple custom hook
export const useQuery = () => {
  const { search } = useLocation();
  return React.useMemo(() => new URLSearchParams(search), [search]);
};

// Complex state management hook
export const useTaskProcessor = (
  task: Task,
  onComplete: (value: Task) => void,
  onError: (e: string) => void
): UseTaskProcessorResult => {
  const [status, setStatus] = useState<TaskStatus>('pending');
  const [result, setResult] = useState<TaskResult | null>(null);
  
  const processTask = useCallback(async () => {
    try {
      setStatus('in-progress');
      const processed = await api.processTask(task.id);
      setResult(processed);
      setStatus('completed');
      onComplete(processed);
    } catch (error) {
      setStatus('failed');
      onError(error.message);
    }
  }, [task.id, onComplete, onError]);
  
  return { status, result, processTask };
};
```

#### 3. **Rules of Hooks - CRITICAL**

React Hooks must follow these essential rules to avoid runtime errors and linting violations:

```typescript
// ✅ CORRECT: Always call hooks at the top level
const MyComponent = ({ isEditable }: { isEditable: boolean }) => {
  // Always call hooks in the same order every render
  const editableForm = useEditableForm(data);
  const [state, setState] = useState(initialState);
  
  // Use conditions INSIDE the component logic, not for hook calls
  if (!isEditable) {
    return <ReadOnlyView />;
  }
  
  return <EditableView form={editableForm} />;
};

// ❌ WRONG: Conditional hook calls violate Rules of Hooks
const BadComponent = ({ isEditable }: { isEditable: boolean }) => {
  // This will cause react-hooks/rules-of-hooks ESLint error
  const editableForm = isEditable ? useEditableForm(data) : null;
  
  if (someCondition) {
    // Never call hooks inside conditions, loops, or nested functions
    const [state, setState] = useState(initialState); // ❌ WRONG
  }
  
  return <div />;
};

// ✅ CORRECT: Custom hooks also follow the same rules
export const useEditableForm = (data: FormData) => {
  // Always call hooks in the same order
  const [formState, setFormState] = useState(initialValues);
  const validation = useMemo(() => validateForm(formState), [formState]);
  const setName = useCallback((value: string) => {
    setFormState(prev => ({ ...prev, name: value }));
  }, []);
  
  return { formState, validation, setName };
};
```

**Key Rules:**

1. **Only call hooks at the top level** - Never inside loops, conditions, or nested functions
2. **Always call hooks in the same order** - React relies on call order to track hook state
3. **Use conditions for logic, not hook calls** - Conditional rendering should happen after hooks
4. **Custom hooks follow the same rules** - They must also call hooks unconditionally

### State Management Architecture

#### 1. **Redux Toolkit with Proper Type Safety**

```typescript
export const sagaMiddleware = createSagaMiddleware();
const store = configureStore({
  reducer: {
    user: userReducer,
    tasks: taskReducer,
    shared: sharedReducer,
  },
  middleware: (getDefaultMiddleware) => [
    ...getDefaultMiddleware({ serializableCheck: false }),
    sagaMiddleware,
    commitToLocalStorageMiddleware([setUserPreferences]),
  ],
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

#### 2. **Modern Redux Slice Pattern**

```typescript
const sharedSlice = createSlice({
  name: 'shared',
  initialState: {
    errorPopups: [],
    config: loadFromLocalStorage(),
    isSettingsDropdownOpen: false,
    currentLocation: null,
  } as SharedState,
  reducers: {
    setCurrentLocation: (state, action: PayloadAction<Location>) => {
      state.currentLocation = action.payload;
    },
    showErrorPopup: (state, action: PayloadAction<string>) => {
      state.errorPopups.push({ id: Date.now(), message: action.payload });
    },
    // ... more reducers
  },
});
```

#### 3. **React Query for Server State**

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      refetchOnWindowFocus: (query: Query) => {
        if (query.state.status === 'success') {
          const minutesSinceUpdate = millisecondsToMinutes(
            getTime(new Date()) - query.state.dataUpdatedAt
          );
          return minutesSinceUpdate > 15 && query.state.fetchStatus !== 'fetching';
        }
        return true;
      },
      retry: false,
    },
  },
});
```

### MUI Integration Patterns

#### 1. **Strategic MUI Component Usage**

```typescript
import {
  List,
  ListItem,
  Dialog,
  Box,
  Stack,
  Typography,
  Button,
} from '@mui/material';

// Usage with sx prop
<Dialog
  fullWidth
  disablePortal
  disableEnforceFocus
  sx={{ backdropFilter: 'blur(0.1rem)' }}
  open={openModal}
  onClose={closeModal}
>
  <Box sx={{ p: 3 }}>
    <Typography variant="h6">Modal Title</Typography>
    <Stack spacing={2}>
      <Typography>Modal content goes here</Typography>
      <Button variant="contained">Action</Button>
    </Stack>
  </Box>
</Dialog>
```

#### 2. **MUI Icons Integration**

```typescript
import PlayCircleIcon from '@mui/icons-material/PlayCircle';
import PauseCircleIcon from '@mui/icons-material/PauseCircle';
import CircularProgress from '@mui/material/CircularProgress';

// Dynamic icon rendering
const getIcon = (isLoading: boolean, isActive: boolean, isEnabled: boolean): ReactNode => {
  if (isLoading) return <CircularProgress size={32} color="inherit" />;
  if (isActive) return <PauseCircleIcon sx={{ fontSize: '2.5rem' }} />;
  if (isEnabled) return <PlayCircleIcon sx={{ fontSize: '2.5rem' }} />;
  return <PlayCircleIcon sx={{ fontSize: '2.5rem' }} />;
};
```

## Styling Strategy

### Current Hybrid Approach

1. **SCSS for Component-specific Styles**

   - Each component has its own `.scss` file
   - Example: `Button.scss`, `Modal.scss`, `Card.scss`
2. **MUI Components for Layout & Interaction**

   - `Stack`, `Box`, `Grid` for layout
   - `Button`, `Dialog`, `Typography` for UI elements
3. **Inline Styles for Dynamic Properties**

   ```typescript
   // Dynamic color styling
   style={{ background: color }}

   // Multiple dynamic properties
   <div
     className="Card__header"
     style={{ borderBottomColor: color, color }}
   >
   ```

## Implementation Patterns

### 1. **Error Handling Pattern**

```typescript
// Global error state
export const showErrorPopup = createAction<string>('shared/showErrorPopup');

// Component-level error handling
const dispatch = useDispatch();
const onError = (err: string) => dispatch(showErrorPopup(err));

// Async operation error handling
try {
  await api.processData(userId, data);
  setIsProcessed(true);
} catch (e: any) {
  onError(`something went wrong: ${e.message}`);
} finally {
  setIsLoading(false);
}
```

### 2. **Permission-Based Rendering**

```typescript
<WithPermission mode="hide" location={user.location} permission="edit-users">
  {user.isActive && (
    <div className="px-3 py-2 border-top">
      <Button variant="contained" onClick={handleEdit}>
        Edit User
      </Button>
    </div>
  )}
</WithPermission>
```

### 3. **Complex State Management with Custom Hooks**

```typescript
export const useTaskProcessor = (
  task: Task,
  onComplete: (value: Task) => void,
  onError: (e: string) => void
): UseTaskProcessorResult => {
  const [state, setState] = useState<ProcessorState>(initialState);
  const [currentStep, setCurrentStep] = useState<StepType>('validation');
  
  // Mutation synchronization
  const useSyncMutation = useMutationSynchroniser(() => {
    onError('Wait for previous operation to complete');
  });
  
  // Complex async operations with proper cleanup
  const handleValidation = useSyncMutation({
    mutationFn: async (taskId: string) => {
      cancelPendingOperations();
      updateStep('validation', { status: Status.LOADING, error: null });
      return api.validateTask(taskId);
    },
    onSuccess: (result) => {
      setCurrentStep('processing');
      setState(result);
      onComplete(result);
    },
    onError: (error: AxiosError) => {
      updateStep('validation', { error: error.message, status: Status.ERROR });
    },
  });
  
  return { 
    state, 
    currentStep,
    handler: currentStep === 'validation' ? handleValidation : handleProcessing,
  };
};
```

## Implementation Guidelines

### Component Creation

1. Always define TypeScript interfaces for props
2. Use functional components with proper typing
3. Co-locate styles (`.scss` files) with components
4. Extract complex logic into custom hooks
5. Follow the existing naming conventions

### State Management

1. Use Redux Toolkit for global state
2. Use React Query for server state
3. Create typed selectors for state access
4. Handle errors through the global error popup system

### MUI Usage

1. Prefer MUI components for layout (`Stack`, `Box`, `Grid`)
2. Use MUI components for interactive elements (`Button`, `Dialog`)
3. Apply custom styling through `sx` prop or SCSS when needed
4. Import MUI icons from `@mui/icons-material`
