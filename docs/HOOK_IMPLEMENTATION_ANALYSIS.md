# Frida Gum ARM64 Hook 实现机制分析

## 概述

Frida Gum 在 ARM64 架构下通过**函数入口替换 + Trampoline（跳板）机制**实现函数 hook，支持在函数执行前后（on_enter 和 on_exit）插入回调，并能够调用原函数。

## 核心组件

### 1. GumFunctionContext（函数上下文）

每个被 hook 的函数都有一个对应的 `GumFunctionContext`，包含：

```c
struct _GumFunctionContext {
  gpointer function_address;           // 原函数地址
  gpointer on_enter_trampoline;        // 进入时的跳板地址
  gpointer on_invoke_trampoline;       // 调用原函数的跳板地址
  gpointer on_leave_trampoline;        // 离开时的跳板地址
  guint8 overwritten_prologue[32];     // 保存的原函数开头代码
  guint overwritten_prologue_len;      // 被覆盖的代码长度
  gpointer replacement_function;       // 替换函数（如果使用 replace）
  volatile GPtrArray * listener_entries; // 监听器列表
  // ...
};
```

### 2. Thunk（粘合代码）

系统会创建两个全局的 thunk：
- **enter_thunk**: 处理函数进入逻辑
- **leave_thunk**: 处理函数离开逻辑

## Hook 建立流程

### 步骤 1: 创建 Trampoline（跳板代码）

在 `_gum_interceptor_backend_create_trampoline()` 中：

1. **分配代码内存**：为新代码分配可执行内存（`trampoline_slice`）

2. **生成 on_enter_trampoline**：
   ```c
   // 加载 function_context 到 X17
   gum_arm64_writer_put_ldr_reg_address (aw, ARM64_REG_X17, GUM_ADDRESS (ctx));
   // 加载 enter_thunk 地址到 X16
   gum_arm64_writer_put_ldr_reg_address (aw, ARM64_REG_X16,
       GUM_ADDRESS (gum_sign_code_pointer (self->enter_thunk)));
   // 跳转到 enter_thunk
   gum_arm64_writer_put_br_reg (aw, ARM64_REG_X16);
   ```

3. **生成 on_leave_trampoline**：
   ```c
   // 类似地，加载 function_context 和 leave_thunk
   gum_arm64_writer_put_ldr_reg_address (aw, ARM64_REG_X17, GUM_ADDRESS (ctx));
   gum_arm64_writer_put_ldr_reg_address (aw, ARM64_REG_X16,
       GUM_ADDRESS (gum_sign_code_pointer (self->leave_thunk)));
   gum_arm64_writer_put_br_reg (aw, ARM64_REG_X16);
   ```

4. **生成 on_invoke_trampoline（原函数跳板）**：
   - 使用 `GumArm64Relocator` 将原函数开头的指令重定位到新位置
   - 如果原函数开头被部分覆盖，添加跳转指令回到原函数剩余部分

5. **保存原函数开头**：
   ```c
   gum_memcpy (ctx->overwritten_prologue, function_address, reloc_bytes);
   ```

### 步骤 2: 激活 Hook（修改原函数）

在 `_gum_interceptor_backend_activate_trampoline()` 中：

修改原函数开头，跳转到 `on_enter_trampoline`：

```c
// 根据可用空间，生成不同的跳转指令
switch (data->redirect_code_size) {
  case 4:
    gum_arm64_writer_put_b_imm (aw, on_enter);  // 直接跳转（26位偏移）
    break;
  case 8:
    gum_arm64_writer_put_adrp_reg_address (aw, data->scratch_reg, on_enter);
    gum_arm64_writer_put_br_reg_no_auth (aw, data->scratch_reg);  // 使用寄存器跳转
    break;
  case 16:
    gum_arm64_writer_put_ldr_reg_address (aw, data->scratch_reg, on_enter);
    gum_arm64_writer_put_br_reg (aw, data->scratch_reg);  // 加载地址后跳转
    break;
}
```

## 执行流程

### 函数调用时的执行路径

```
原函数调用
  ↓
[原函数开头被替换] → 跳转到 on_enter_trampoline
  ↓
on_enter_trampoline → 跳转到 enter_thunk
  ↓
enter_thunk (汇编代码)
  ├─ 保存所有寄存器到栈（gum_emit_prolog）
  ├─ 调用 _gum_function_context_begin_invocation()
  │   ├─ 执行所有 on_enter 回调
  │   ├─ 决定下一步跳转地址（next_hop）
  │   └─ 如果需要捕获返回，修改 caller_ret_addr
  ├─ 恢复寄存器（gum_emit_epilog）
  └─ 跳转到 next_hop
  ↓
[next_hop 可能是]
  ├─ on_invoke_trampoline → 执行原函数
  └─ replacement_function → 执行替换函数
  ↓
[函数返回时]
如果设置了 on_leave 或 replacement，LR 已被修改为 on_leave_trampoline
  ↓
on_leave_trampoline → 跳转到 leave_thunk
  ↓
leave_thunk (汇编代码)
  ├─ 保存所有寄存器到栈
  ├─ 调用 _gum_function_context_end_invocation()
  │   ├─ 执行所有 on_leave 回调
  │   └─ 返回原始调用者地址
  ├─ 恢复寄存器
  └─ 返回到原始调用者
```

### Enter Thunk 实现

在 `guminterceptor-arm64-glue.S` 中的 `_gum_interceptor_begin_invocation`:

```assembly
gum_emit_prolog:
  # 保存所有寄存器（X0-X30, Q0-Q31, FP, LR, SP, NZCV）
  sub sp, sp, GUM_FRAME_SIZE - (2 * 8)
  # ... 保存寄存器代码 ...

# 准备参数
add x1, sp, GUM_FRAME_OFFSET_CPU_CONTEXT      # cpu_context
add x2, sp, GUM_FRAME_OFFSET_CPU_CONTEXT + GUM_CPU_CONTEXT_OFFSET_LR  # caller_ret_addr
add x3, sp, GUM_FRAME_OFFSET_NEXT_HOP         # next_hop
# function_context 已在 X17 中

# 调用 C 函数
bl _gum_function_context_begin_invocation

gum_emit_epilog:
  # 恢复所有寄存器
  # ...
  # 从 next_hop 获取跳转地址（存储在栈上）
  ldr x16, [sp, GUM_FRAME_SIZE - (2 * 8)]
  add sp, sp, GUM_FRAME_SIZE
  br x16  # 跳转到 next_hop
```

### Begin Invocation 核心逻辑

在 `_gum_function_context_begin_invocation()` 中：

1. **检查是否应该跳过 hook**：
   - 防止递归调用（通过 TLS guard）
   - 检查线程过滤

2. **调用所有 on_enter 回调**：
   ```c
   for (i = 0; i != listener_entries->len; i++) {
     ListenerEntry * listener_entry = g_ptr_array_index (listener_entries, i);
     if (listener_entry->listener_interface->on_enter != NULL) {
       listener_entry->listener_interface->on_enter (
           listener_entry->listener_instance, invocation_ctx);
     }
   }
   ```

3. **决定下一步跳转地址**：
   - 如果有 `replacement_function`：跳转到替换函数
   - 否则：跳转到 `on_invoke_trampoline`（原函数）

4. **设置返回捕获**：
   - 如果需要执行 on_leave 或替换函数，修改 `caller_ret_addr` 为 `on_leave_trampoline`
   - 这样函数返回时会先执行 leave 逻辑

### Leave Thunk 实现

在 `guminterceptor-arm64-glue.S` 中的 `_gum_interceptor_end_invocation`:

```assembly
gum_emit_prolog:
  # 保存所有寄存器
  # ...

# 准备参数
add x1, sp, GUM_FRAME_OFFSET_CPU_CONTEXT      # cpu_context
add x2, sp, GUM_FRAME_OFFSET_NEXT_HOP         # next_hop（存储返回地址）
# function_context 已在 X17 中

# 调用 C 函数
bl _gum_function_context_end_invocation

gum_emit_epilog:
  # 恢复所有寄存器
  # ...
  # 从 next_hop 获取原始返回地址
  ldr x16, [sp, GUM_FRAME_SIZE - (2 * 8)]
  add sp, sp, GUM_FRAME_SIZE
  br x16  # 返回到原始调用者
```

### End Invocation 核心逻辑

在 `_gum_function_context_end_invocation()` 中：

1. **调用所有 on_leave 回调**：
   ```c
   for (i = 0; i != listener_entries->len; i++) {
     ListenerEntry * listener_entry = g_ptr_array_index (listener_entries, i);
     if (listener_entry->listener_interface->on_leave != NULL) {
       listener_entry->listener_interface->on_leave (
           listener_entry->listener_instance, invocation_ctx);
     }
   }
   ```

2. **返回到原始调用者**：
   - 从调用栈中恢复原始的 `caller_ret_addr`
   - 通过 `next_hop` 返回到调用者

## 关键技术点

### 1. LR（Link Register）替换机制

为了实现 on_leave 回调，Gum 会修改函数的返回地址（LR）：

- 在 `begin_invocation` 中，如果检测到需要捕获返回，会将 `caller_ret_addr` 修改为 `on_leave_trampoline`
- 原函数的 LR 被保存到调用栈中
- 函数返回时，会先执行 leave thunk，然后再返回到真正的调用者

### 2. 指令重定位（Relocation）

使用 `GumArm64Relocator` 将原函数开头的指令重定位到 trampoline：

- 分析原函数开头的指令
- 确定需要覆盖的最小指令数（至少 4 字节，通常是 16 字节）
- 将指令重定位到新的内存位置，处理相对跳转等特殊情况

### 3. 调用栈管理

使用 `GumInvocationStack` 维护函数调用的嵌套关系：

- 每个函数调用创建一个 `GumInvocationStackEntry`
- 保存函数上下文、原始返回地址、CPU 上下文等
- 支持深度嵌套的 hook 调用

### 4. 寄存器完整保存

在 thunk 中保存和恢复所有 ARM64 寄存器：

- 通用寄存器：X0-X30
- 浮点/向量寄存器：Q0-Q31
- 状态寄存器：NZCV（条件标志）
- 栈指针和帧指针：SP, FP
- 链接寄存器：LR

确保 hook 回调不会影响原函数的执行环境。

## 总结

Frida Gum 的 hook 机制通过以下方式实现：

1. **函数入口替换**：修改原函数开头，跳转到自定义代码
2. **Trampoline 机制**：创建跳板代码，保存原函数代码并提供跳转入口
3. **Thunk 粘合**：使用汇编代码进行寄存器保存/恢复和 C 函数调用
4. **LR 替换**：通过修改返回地址实现函数返回拦截
5. **调用栈管理**：维护函数调用关系，支持嵌套 hook

这种设计既保证了功能完整性，又最小化了对原函数的影响，是一个成熟稳定的 hook 实现方案。

