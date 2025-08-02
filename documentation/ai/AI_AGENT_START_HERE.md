# 🤖 AI Agent Start Here - Interactor Development Guide

## 🚀 Quick Start (Read This First!)

**Interactor** is a modular, event-driven system for interactive installations. You are here to develop **new modules** or **enhance existing features**.

### 📁 What You're Working With
- **Backend**: Node.js/TypeScript modules that handle inputs/outputs (sensors, lights, audio, etc.)
- **Frontend**: React visual editor for connecting modules
- **Architecture**: Simple base classes + manifest-driven modules

---

## 🎯 Your Main Tasks Will Be

### ✅ **Module Development** (Most Common)
1. **Extend base classes**: `InputModuleBase` or `OutputModuleBase` 
2. **Create manifest.json**: Describes your module's config and capabilities
3. **Follow the lifecycle**: `onInit()` → `onStart()` → `onStop()`

**📂 Template Location**: `backend/src/modules/input/time_input/` (great example)

### ✅ **Frontend Features** (Less Common)
- The frontend is **simplified architecture only**
- Main files: `App.tsx`, `NodeEditor.tsx`, `CustomEdge.tsx`
- **No singletons or complex state management**

---

## 🚨 **Critical Rules** (Read These!)

### ❌ **What NOT to Touch**
- **Don't modify core services**: `MessageRouter`, `StateManager`, `ModuleLoader`
- **Don't create new base classes**: Use existing `InputModuleBase`/`OutputModuleBase`
- **Don't add complex state management**: Keep it simple
- **Don't create documentation files**: Focus on code

### ✅ **What TO Do**
- **Read the existing module examples first**
- **Use shared types from `@interactor/shared`**
- **Follow the manifest schema exactly**
- **Test your module in isolation**
- **Keep modules focused and simple**
- **✨ Use InteractorError for all error handling**
- **✨ Add retry logic for network/file operations**
- **✨ Include helpful error messages and suggestions**

---

## 🚨 **Error Handling & Logging** (New System!)

### ✅ **Use InteractorError for All Errors**
The system now has **centralized error handling** with user-friendly messages and automatic retry capabilities.

```typescript
import { InteractorError } from '../../../core/ErrorHandler';

// Validation errors with helpful suggestions
if (!config.url) {
  throw InteractorError.validation(
    'URL is required for HTTP module',
    { providedConfig: config },
    [
      'Provide a valid URL (e.g., "http://localhost:3000/api")',
      'Include the protocol (http:// or https://)',
      'Check that the target server is accessible'
    ]
  );
}

// Network errors (automatically retryable)
throw InteractorError.networkError(
  `Cannot connect to ${this.url}`,
  originalError
);

// Module errors with operation context
throw InteractorError.moduleError(
  this.name, 
  'initialization', 
  error, 
  retryable: true
);
```

### ✅ **Use RetryHandler for Network Operations**
Automatic retry with exponential backoff for reliable operations:

```typescript
import { RetryHandler } from '../../../core/RetryHandler';

// Network requests with automatic retry
const { result, attempts } = await RetryHandler.withNetworkRetry(
  async () => await fetch(this.url, options),
  {
    maxAttempts: 3,
    onRetry: (error, attempt, delay) => {
      this.logger?.warn(`Retry ${attempt}/3 in ${delay}ms: ${error.message}`);
    }
  }
);

// File operations with retry
await RetryHandler.withFileRetry(async () => {
  return await fs.writeFile(path, data);
});
```

### ✅ **Enhanced Logging Patterns**
Use the built-in logger with proper context:

```typescript
export class MyModule extends InputModuleBase {
  protected async onStart(): Promise<void> {
    try {
      // Success logging
      this.logger?.info('Module started successfully', {
        url: this.url,
        timeout: this.timeout
      });
      
      // Debug information
      this.logger?.debug('Processing data', { 
        dataSize: data.length,
        timestamp: Date.now()
      });
      
    } catch (error) {
      // Error logging with context
      this.logger?.error('Startup failed', error, {
        moduleName: this.name,
        config: this.config
      });
      
      // Re-throw as InteractorError
      throw InteractorError.moduleError(
        this.name, 
        'startup', 
        error as Error, 
        retryable: true
      );
    }
  }
}
```

### ✅ **Error Types to Use**

| Use Case | Error Type | Example |
|----------|------------|---------|
| Invalid config | `InteractorError.validation()` | Missing URL, invalid timeout |
| Resource not found | `InteractorError.notFound()` | Module instance, file not found |
| Network failures | `InteractorError.networkError()` | Connection refused, timeout |
| File operations | `InteractorError.fileError()` | Permission denied, disk full |
| Module operations | `InteractorError.moduleError()` | Start/stop failures |
| Conflicts | `InteractorError.conflict()` | Already running, port in use |

### ❌ **Don't Use Generic Errors**
```typescript
// ❌ BAD - Generic error
throw new Error('Something went wrong');

// ✅ GOOD - Specific error with context
throw InteractorError.validation(
  'Timeout must be between 1000-30000ms',
  { provided: config.timeout, min: 1000, max: 30000 },
  ['Try 5000ms for most cases', 'Use 10000ms for slow networks']
);
```

---

## 🚨 **Frontend State Management & Debugging** (Critical Lessons!)

### ✅ **React Re-rendering Issues**
**Problem**: Module settings changes not updating UI
**Root Cause**: `useCallback` dependencies not watching `instance.config` changes
**Solution**: Add state variable to force re-renders when config changes

```typescript
// ❌ WRONG - React won't re-render when config changes
const renderConfig = useCallback(() => {
  // ...
}, [config.renderConfig, instance, props.data.onConfigChange]);

// ✅ CORRECT - Force re-render when config changes
const [configVersion, setConfigVersion] = useState(0);
const renderConfig = useCallback(() => {
  // ...
  const updateConfig = (key: string, value: any) => {
    // ... update logic ...
    setConfigVersion(prev => prev + 1); // Force re-render
  };
}, [config.renderConfig, instance, props.data.onConfigChange, configVersion]);
```

### ✅ **Module Creation Race Conditions**
**Problem**: Double module creation when dragging from sidebar
**Root Cause**: `onDrop` creating nodes directly AND through interactions
**Solution**: Single source of truth - only update interactions, let useEffect handle node creation

```typescript
// ❌ WRONG - Creates race condition
const onDrop = useCallback((event) => {
  // ... create node ...
  setNodes((nds) => [...nds, newNode]); // Direct manipulation
  onInteractionsChange(updatedInteractions); // Also triggers useEffect
}, [dependencies]);

// ✅ CORRECT - Single source of truth
const onDrop = useCallback((event) => {
  // ... create node ...
  onInteractionsChange(updatedInteractions); // Only update interactions
  // useEffect will handle node creation from interactions
}, [dependencies]);
```

### ✅ **Edge Duplication Prevention**
**Problem**: React warnings about duplicate edge keys
**Root Cause**: useEffect running multiple times creating duplicate edges
**Solution**: Check for existing edges before creating new ones

```typescript
// ✅ Add duplicate prevention
const edgeId = route.id || `edge-${interaction.id}-${route.source}-${route.target}`;
const existingEdge = newEdges.find(edge => edge.id === edgeId);
if (existingEdge) {
  console.log('Skipping duplicate edge:', edgeId);
  return;
}
newEdges.push({ id: edgeId, ... });
```

### ✅ **Debugging Frontend State Issues**
**Key Debugging Points**:
1. **Check useEffect dependencies** - ensure they're stable
2. **Add console.log to track state changes** - see when components re-render
3. **Use hash comparison** - prevent unnecessary updates
4. **Monitor React warnings** - duplicate keys indicate state issues

```typescript
// ✅ Debug state changes
console.log('Component: useEffect triggered with data:', data);
console.log('Component: Hash comparison:', currentHash === lastHash);
console.log('Component: State update:', newState);
```

### ✅ **Port Configuration Issues**
**Problem**: Frontend trying to connect to wrong backend port
**Root Cause**: Vite config proxy pointing to wrong port
**Solution**: Ensure frontend (3000) and backend (3001) ports match config

```typescript
// ✅ Correct Vite config
export default defineConfig({
  server: {
    port: 3000, // Frontend port
    proxy: {
      '/api': {
        target: 'http://localhost:3001', // Backend port
        changeOrigin: true,
      },
    },
  },
});
```

---

## 📚 **Essential Reading (In Order)**

1. **`MODULE_DEVELOPMENT.md`** - How to create modules
2. **`shared/src/types/index.ts`** - All available types  
3. **`backend/src/modules/input/time_input/`** - Complete example
4. **`backend/src/core/ErrorHandler.ts`** - Error handling system
5. **`backend/src/core/RetryHandler.ts`** - Retry mechanisms
6. **`API_GUIDE.md`** - REST endpoints and WebSocket events

---

## 🔧 **Development Workflow**

```bash
# 1. Create your module directory
mkdir backend/src/modules/input/my_new_module

# 2. Create required files
touch backend/src/modules/input/my_new_module/index.ts
touch backend/src/modules/input/my_new_module/manifest.json

# 3. Start development server (don't restart it yourself)
npm run dev  # (User handles this)

# 4. Test your module via API or frontend
```

---

## 🎨 **Current Architecture Status**

- ✅ **Backend**: Stable, don't modify core services
- ✅ **Frontend**: Recently simplified, stable patterns only
- ✅ **Shared Types**: Complete and consistent
- ❌ **No complex patterns**: Singletons, complex state, duplicate WebSocket logic

---

## 💡 **Quick Examples**

### Module Structure (with Error Handling)
```typescript
import { InteractorError } from '../../../core/ErrorHandler';
import { RetryHandler } from '../../../core/RetryHandler';

export class MyInputModule extends InputModuleBase {
  constructor(config: MyConfig) {
    // Validate config with helpful messages
    if (!config.interval || config.interval < 100) {
      throw InteractorError.validation(
        'Interval must be at least 100ms',
        { provided: config.interval },
        ['Try 1000ms for regular updates', 'Use 100-500ms for real-time data']
      );
    }
    
    super('my_input', config, manifest);
  }
  
  protected async onStart(): Promise<void> {
    try {
      // Use retry for network operations
      await RetryHandler.withNetworkRetry(async () => {
        await this.connectToExternalService();
      });
      
      this.emitTrigger('trigger', { value: 123 });
      this.logger?.info('Module started successfully');
      
    } catch (error) {
      this.logger?.error('Failed to start module', error);
      throw InteractorError.moduleError(
        this.name, 
        'start', 
        error as Error, 
        retryable: true
      );
    }
  }
}
```

### Manifest Structure
```json
{
  "name": "My Input",
  "type": "input",
  "version": "1.0.0",
  "description": "Does something useful",
  "configSchema": { "type": "object", "properties": {} },
  "events": [{ "name": "trigger", "type": "output" }]
}
```

---

## 🏁 **Success Criteria**

- Your module loads without errors
- Configuration UI appears in frontend  
- Events route correctly through the system
- No impact on existing modules
- Clean, focused code
- **✨ Proper error handling with InteractorError**
- **✨ User-friendly error messages with suggestions**
- **✨ Automatic retry for transient failures**

**Ready to start? Read `MODULE_DEVELOPMENT.md` next!**