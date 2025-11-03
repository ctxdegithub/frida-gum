# 在 begin_invocation 中直接修改 next_hop 的方案

## 概述

通过修改 `_gum_function_context_begin_invocation` 函数，在决定 `next_hop` 时检查外部标志。如果 `on_enter` 设置了某个标志，直接将 `*next_hop` 设置为 `on_invoke_trampoline`（原函数），而不是 `replacement_function`（替换函数）。这样可以实现：

- **默认情况**：跳转到 `replacement_function`（空函数）
- **on_enter 设置了标志**：跳转到 `on_invoke_trampoline`（原函数）

## 优势

相比在 `replacement_function` 中做判断的方案，这个方案有以下优势：

1. **性能更好**：减少一次函数调用（不需要进入 replacement_function 再判断）
2. **逻辑更清晰**：跳转决策在 `begin_invocation` 中统一处理
3. **避免函数调用开销**：直接跳转，无需经过替换函数的栈帧

## 实现方案

### 方案一：通过 replacement_data 传递状态标志

#### 1. 定义状态结构

```c
// 函数 hook 状态结构
typedef struct _FunctionHookState FunctionHookState;
struct _FunctionHookState {
  gpointer function_address;        // 被 hook 的函数地址
  volatile gboolean should_call_original;  // 是否应该调用原函数（原子标志）
  // 可以使用原子操作或者加锁来保证线程安全
};
```

#### 2. 修改 begin_invocation 逻辑

**文件**: `guminterceptor.c:1464-1478`

```c
  // 在执行完所有 on_enter 回调后，决定 next_hop
  if (function_ctx->replacement_function != NULL)
  {
    gboolean should_call_original = FALSE;
    
    // 检查 replacement_data 中的标志
    if (function_ctx->replacement_data != NULL)
    {
      FunctionHookState * state = (FunctionHookState *) function_ctx->replacement_data;
      
      // 使用原子操作或内存屏障读取标志
      // 注意：这里需要确保在 on_enter 中设置的标志能及时看到
      should_call_original = g_atomic_int_get ((gint *) &state->should_call_original);
    }
    
    if (should_call_original)
    {
      // 如果标志为 TRUE，直接跳转到原函数
      *next_hop = function_ctx->on_invoke_trampoline;
      
      // 注意：虽然跳转到原函数，但我们需要确保：
      // 1. stack_entry 正确设置（如果需要 on_leave）
      // 2. calling_replacement 标志的处理
    }
    else
    {
      // 默认行为：跳转到替换函数（空函数）
      stack_entry->calling_replacement = TRUE;
      stack_entry->cpu_context = *cpu_context;
      stack_entry->original_system_error = system_error;
      invocation_ctx->cpu_context = &stack_entry->cpu_context;
      invocation_ctx->backend = &interceptor_ctx->replacement_backend;
      invocation_ctx->backend->data = function_ctx->replacement_data;

      *next_hop = function_ctx->replacement_function;
    }
  }
  else
  {
    *next_hop = function_ctx->on_invoke_trampoline;
  }
```

#### 3. 完整修改代码

**文件**: `guminterceptor.c`

```c
gboolean
_gum_function_context_begin_invocation (GumFunctionContext * function_ctx,
                                        GumCpuContext * cpu_context,
                                        gpointer * caller_ret_addr,
                                        gpointer * next_hop)
{
  // ... 前面的代码保持不变 ...

  if (invoke_listeners)
  {
    GPtrArray * listener_entries;
    guint i;

    invocation_ctx->cpu_context = cpu_context;
    invocation_ctx->backend = &interceptor_ctx->listener_backend;

    listener_entries =
        (GPtrArray *) g_atomic_pointer_get (&function_ctx->listener_entries);
    for (i = 0; i != listener_entries->len; i++)
    {
      ListenerEntry * listener_entry;
      ListenerInvocationState state;

      listener_entry = g_ptr_array_index (listener_entries, i);
      if (listener_entry == NULL)
        continue;

      if (only_invoke_unignorable_listeners && !listener_entry->unignorable)
        continue;

      state.point_cut = GUM_POINT_ENTER;
      state.entry = listener_entry;
      state.interceptor_ctx = interceptor_ctx;
      state.invocation_data = stack_entry->listener_invocation_data[i];
      invocation_ctx->backend->data = &state;

      if (listener_entry->listener_interface->on_enter != NULL)
      {
        listener_entry->listener_interface->on_enter (
            listener_entry->listener_instance, invocation_ctx);
      }
    }

    system_error = invocation_ctx->system_error;
  }

  // ... 中间的代码保持不变 ...

  if (function_ctx->replacement_function != NULL)
  {
    gboolean should_call_original = FALSE;
    
    // 检查 replacement_data 中的标志
    if (function_ctx->replacement_data != NULL)
    {
      // 假设 replacement_data 指向 FunctionHookState 结构
      // 使用 volatile 读取或原子操作确保可见性
      FunctionHookState * state = (FunctionHookState *) function_ctx->replacement_data;
      
      // 使用内存屏障确保看到 on_enter 中的修改
      // 注意：需要确保数据结构定义时使用了 volatile 或原子类型
      should_call_original = g_atomic_int_get ((volatile gint *) &state->should_call_original);
    }
    
    if (should_call_original)
    {
      // 直接跳转到原函数，不执行替换函数
      // 注意：仍然需要设置必要的状态，以便 on_leave 能正常工作
      if (will_trap_on_leave)
      {
        // 如果设置了 on_leave，需要标记这是"替换函数"调用
        // 但实际上调用的是原函数，所以这里的处理需要仔细考虑
        stack_entry->calling_replacement = FALSE;  // 或者保持为 TRUE，取决于你的需求
        stack_entry->cpu_context = *cpu_context;
        stack_entry->original_system_error = system_error;
      }
      
      *next_hop = function_ctx->on_invoke_trampoline;
    }
    else
    {
      // 默认：跳转到替换函数（空函数）
      stack_entry->calling_replacement = TRUE;
      stack_entry->cpu_context = *cpu_context;
      stack_entry->original_system_error = system_error;
      invocation_ctx->cpu_context = &stack_entry->cpu_context;
      invocation_ctx->backend = &interceptor_ctx->replacement_backend;
      invocation_ctx->backend->data = function_ctx->replacement_data;

      *next_hop = function_ctx->replacement_function;
    }
  }
  else
  {
    *next_hop = function_ctx->on_invoke_trampoline;
  }

  // ... 后面的代码保持不变 ...
}
```

### 方案二：使用 GumInvocationContext 扩展字段（更灵活）

如果不想依赖 `replacement_data` 的结构，可以通过 `GumInvocationContext` 传递标志。

#### 1. 在 GumInvocationContext 中添加标志字段（需要修改源码）

或者，使用 `replacement_data` 作为一个简单的标志指针，但这需要确保线程安全。

#### 2. 使用 GHashTable 存储状态（推荐）

```c
// 全局状态表（线程安全）
static GHashTable * function_states = NULL;
static GMutex function_states_mutex = G_MUTEX_INIT;

// 获取状态标志
static gboolean
get_should_call_original (gpointer function_address)
{
  FunctionHookState * state;
  gboolean result = FALSE;
  
  g_mutex_lock (&function_states_mutex);
  
  if (function_states != NULL)
  {
    state = g_hash_table_lookup (function_states, function_address);
    if (state != NULL)
    {
      result = state->should_call_original;
    }
  }
  
  g_mutex_unlock (&function_states_mutex);
  
  return result;
}
```

#### 3. 在 begin_invocation 中使用

```c
  if (function_ctx->replacement_function != NULL)
  {
    gboolean should_call_original = FALSE;
    
    // 从全局状态表中获取标志
    should_call_original = get_should_call_original (function_ctx->function_address);
    
    if (should_call_original)
    {
      *next_hop = function_ctx->on_invoke_trampoline;
    }
    else
    {
      // 跳转到替换函数
      stack_entry->calling_replacement = TRUE;
      // ... 其他设置 ...
      *next_hop = function_ctx->replacement_function;
    }
  }
```

### 方案三：最小修改方案（最推荐）

直接在现有代码基础上添加检查，使用 `replacement_data` 作为标志指针：

```c
  if (function_ctx->replacement_function != NULL)
  {
    // 检查 replacement_data 是否指向一个标志
    // 约定：如果 replacement_data 指向一个 gboolean 值，
    //       且该值为 TRUE，则调用原函数
    gboolean should_call_original = FALSE;
    
    if (function_ctx->replacement_data != NULL)
    {
      // 使用原子操作或 volatile 读取
      // 注意：需要确保在 on_enter 中的修改对这里可见
      should_call_original = g_atomic_int_get (
          (volatile gint *) function_ctx->replacement_data);
    }
    
    if (should_call_original)
    {
      // 直接跳转到原函数
      *next_hop = function_ctx->on_invoke_trampoline;
    }
    else
    {
      // 默认：跳转到替换函数（空函数）
      stack_entry->calling_replacement = TRUE;
      stack_entry->cpu_context = *cpu_context;
      stack_entry->original_system_error = system_error;
      invocation_ctx->cpu_context = &stack_entry->cpu_context;
      invocation_ctx->backend = &interceptor_ctx->replacement_backend;
      invocation_ctx->backend->data = function_ctx->replacement_data;

      *next_hop = function_ctx->replacement_function;
    }
  }
  else
  {
    *next_hop = function_ctx->on_invoke_trampoline;
  }
```

## 完整使用示例

### 1. 定义状态结构

```c
// 每个函数的状态
typedef struct _FunctionHookState FunctionHookState;
struct _FunctionHookState {
  volatile gint should_call_original;  // 使用 gint 以便使用原子操作
  gpointer original_function;          // 保存原函数地址（可选）
};

// 初始化状态
static FunctionHookState *
create_hook_state (void)
{
  FunctionHookState * state = g_new0 (FunctionHookState, 1);
  state->should_call_original = FALSE;  // 默认不调用原函数
  return state;
}
```

### 2. on_enter 回调设置标志

```c
static void
on_enter_callback (GumInvocationContext * ic)
{
  gpointer function_address = gum_strip_code_pointer (
      GUM_FUNCPTR_TO_POINTER (ic->function));
  
  // 获取状态（通过 replacement_data 或全局表）
  FunctionHookState * state = (FunctionHookState *) 
      gum_invocation_context_get_replacement_data (ic);
  
  if (state != NULL)
  {
    // 根据条件设置标志
    gboolean should_call = should_call_original_function (ic);
    
    // 使用原子操作设置标志，确保对 begin_invocation 可见
    g_atomic_int_set (&state->should_call_original, should_call ? 1 : 0);
  }
}

static gboolean
should_call_original_function (GumInvocationContext * ic)
{
  // 示例：检查第一个参数
  gpointer arg1 = gum_invocation_context_get_nth_argument (ic, 0);
  if (arg1 != NULL)
  {
    return TRUE;
  }
  
  // 默认不调用
  return FALSE;
}
```

### 3. 设置 Hook

```c
void setup_hook (void)
{
  GumInterceptor * interceptor = gum_interceptor_obtain ();
  gpointer target = (gpointer) target_function;
  gpointer original = NULL;
  FunctionHookState * state = create_hook_state ();
  GumInvocationListener * listener;
  
  // 注册替换函数（可以是空函数，因为不会真正调用）
  gum_interceptor_replace (
      interceptor,
      target,
      (gpointer) dummy_replacement_function,  // 空函数，不会被调用
      state,  // 通过 replacement_data 传递状态
      &original
  );
  
  state->original_function = original;
  
  // 注册 on_enter 回调
  listener = gum_make_call_listener (
      on_enter_callback,
      NULL,  // on_leave
      NULL, NULL
  );
  
  gum_interceptor_attach (
      interceptor,
      target,
      listener,
      NULL,
      GUM_ATTACH_FLAGS_NONE
  );
  
  g_object_unref (listener);
  g_object_unref (interceptor);
}

// 空替换函数（实际上不会被调用，但需要注册）
static void
dummy_replacement_function (void)
{
  // 这个函数理论上不会被调用，因为 begin_invocation 会直接跳转到原函数
  // 但为了安全，可以保留一个简单的实现
  return;
}
```

## 注意事项

### 1. 内存可见性（Memory Visibility）

关键问题：确保 `on_enter` 中设置的标志对 `begin_invocation` 后续的检查可见。

**解决方案**：
- 使用原子操作（`g_atomic_int_get/set`）
- 使用 `volatile` 关键字
- 使用内存屏障

```c
// 在 on_enter 中设置
g_atomic_int_set (&state->should_call_original, 1);

// 在 begin_invocation 中读取
gboolean should_call = g_atomic_int_get ((volatile gint *) &state->should_call_original) != 0;
```

### 2. 线程安全

如果多个线程可能同时访问状态标志，需要：
- 使用原子操作（推荐）
- 或者使用互斥锁（性能较差）

### 3. calling_replacement 标志的处理

当直接跳转到原函数时，需要考虑 `stack_entry->calling_replacement` 的设置：

- **选项 A**：设置为 `FALSE`（因为实际调用的是原函数）
- **选项 B**：设置为 `TRUE`（保持一致性，因为是通过 replacement 路径）

这取决于你的 `on_leave` 处理逻辑。

### 4. will_trap_on_leave 的处理

如果 `will_trap_on_leave` 为 `TRUE`（即需要执行 `on_leave`），即使跳转到原函数，也需要：
- 正确设置 `stack_entry` 的状态
- 确保 `on_leave` 能正确执行

### 5. 替换函数的必要性

如果使用这个方案，`replacement_function` 实际上可能不会被调用，但它仍然需要：
- 匹配原函数的签名
- 在 `gum_interceptor_replace` 中注册

可以考虑使用一个最小的空函数实现。

## 性能对比

### 方案 A：在 replacement_function 中判断

```
调用流程：
  begin_invocation → replacement_function → 判断 → 原函数
  开销：2 次函数调用
```

### 方案 B：在 begin_invocation 中判断（本方案）

```
调用流程：
  begin_invocation → 判断 → 原函数
  开销：1 次函数调用（节省一次）
```

**性能提升**：减少了约 50% 的函数调用开销（在实际场景中，这个提升可能不明显，因为还有其他开销）。

## 完整代码示例

```c
// ========== 状态结构 ==========
typedef struct _FunctionHookState FunctionHookState;
struct _FunctionHookState {
  volatile gint should_call_original;  // 原子标志
  gpointer original_function;
};

// ========== 修改 guminterceptor.c ==========
// 在 _gum_function_context_begin_invocation 函数中，1464-1478 行：

  if (function_ctx->replacement_function != NULL)
  {
    gboolean should_call_original = FALSE;
    
    // 检查标志（通过 replacement_data）
    if (function_ctx->replacement_data != NULL)
    {
      FunctionHookState * state = (FunctionHookState *) function_ctx->replacement_data;
      should_call_original = g_atomic_int_get (
          (volatile gint *) &state->should_call_original) != 0;
    }
    
    if (should_call_original)
    {
      // 直接跳转到原函数
      *next_hop = function_ctx->on_invoke_trampoline;
      
      // 如果需要 on_leave，设置必要的状态
      if (will_trap_on_leave && stack_entry != NULL)
      {
        stack_entry->calling_replacement = FALSE;
        stack_entry->cpu_context = *cpu_context;
        stack_entry->original_system_error = system_error;
      }
    }
    else
    {
      // 默认：跳转到替换函数（空函数）
      stack_entry->calling_replacement = TRUE;
      stack_entry->cpu_context = *cpu_context;
      stack_entry->original_system_error = system_error;
      invocation_ctx->cpu_context = &stack_entry->cpu_context;
      invocation_ctx->backend = &interceptor_ctx->replacement_backend;
      invocation_ctx->backend->data = function_ctx->replacement_data;

      *next_hop = function_ctx->replacement_function;
    }
  }
  else
  {
    *next_hop = function_ctx->on_invoke_trampoline;
  }
```

## 总结

这个方案的优势：
- ✅ **性能更好**：减少一次函数调用
- ✅ **逻辑更直接**：在 `begin_invocation` 中统一处理跳转决策
- ✅ **实现简单**：只需修改一处代码

需要注意的点：
- ⚠️ **内存可见性**：必须使用原子操作或 volatile
- ⚠️ **线程安全**：多线程环境下需要正确同步
- ⚠️ **状态一致性**：确保 `calling_replacement` 等标志正确设置

这是一个很好的优化方案，特别适合对性能有要求的场景。

