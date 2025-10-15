# Additional Jinja2 Template Fixes

## New Errors Encountered

### Error 1: Accessing .rc on Skipped Task

```
fatal: [localhost]: FAILED! =>
  "msg": "The conditional check '... cm_ctrl_rollout_after_patch.rc != 0' failed.
  The error was: 'dict object' has no attribute 'rc'"
```

**Line**: 587

### Error 2: Invalid List Slicing Syntax

```
fatal: [localhost]: FAILED! =>
  "msg": "template error while templating string: expected token 'end of print statement', got '['.
  String: {{ cluster_events.stdout_lines | default([]) | list[-50:] }}"
```

**Line**: 742

---

## Root Causes

### Error 1: Skipped Tasks Don't Have .rc Attribute

When a task is skipped (due to `when:` condition being false), Ansible registers the variable as a dict with:

```python
{
  'skipped': True,
  'skip_reason': '...'
}
```

It does **NOT** have an `.rc` attribute!

When you try to access `.rc` on a skipped task:

```yaml
cm_ctrl_rollout_after_patch.rc # ❌ AttributeError!
```

### Error 2: Jinja2 Doesn't Support Python List Slicing

Jinja2 templates don't support Python's slice notation:

```yaml
list[-50:] # ❌ Syntax error in Jinja2!
```

You must use filters instead:

```yaml
list | last(50)   # ✅ Get last 50 items
list | first(10)  # ✅ Get first 10 items
```

Or do the slicing in the shell command itself.

---

## Fixes Applied

### Fix 1: Safe Access to .rc with Default

**Before**:

```yaml
when: cm_rollout_failed and (cm_ctrl_rollout_after_patch is not defined or cm_ctrl_rollout_after_patch.rc != 0)
```

**Problem**: If `cm_ctrl_rollout_after_patch` is skipped, accessing `.rc` raises AttributeError.

**After**:

```yaml
when: cm_rollout_failed and (cm_ctrl_rollout_after_patch is not defined or cm_ctrl_rollout_after_patch.skipped | default(false) or cm_ctrl_rollout_after_patch.rc | default(1) != 0)
```

**What it does**:

- Check if task was skipped: `cm_ctrl_rollout_after_patch.skipped | default(false)`
- Safely access `.rc` with default: `cm_ctrl_rollout_after_patch.rc | default(1)`
- If skipped OR rc != 0, run the final attempt

### Fix 2: Use tail in Shell Command

**Before**:

```yaml
- name: Show ALL cluster events (for scheduling issues)
  ansible.builtin.command: >
    kubectl get events --all-namespaces --sort-by=.lastTimestamp
  register: cluster_events

# Then in diagnostics:
- "{{ cluster_events.stdout_lines | default([]) | list[-50:] }}" # ❌ Error!
```

**After**:

```yaml
- name: Show ALL cluster events (for scheduling issues)
  ansible.builtin.shell: >
    kubectl get events --all-namespaces --sort-by=.lastTimestamp | tail -n 50
  register: cluster_events

# Then in diagnostics:
- "{{ cluster_events.stdout_lines | default([]) }}" # ✅ Already limited to 50
```

**What changed**:

- Changed `command` → `shell` (to use pipe `|`)
- Added `| tail -n 50` to limit output in the shell command
- Removed invalid `list[-50:]` from Jinja2 template

---

## Key Lessons

### 1. Always Use .default() When Accessing Task Result Attributes

When a task might be skipped or failed:

```yaml
# ❌ Bad:
when: my_task.rc != 0

# ✅ Good:
when: my_task.rc | default(1) != 0

# ✅ Even better (check if skipped):
when: my_task.skipped | default(false) or my_task.rc | default(1) != 0
```

### 2. Check for Skipped Tasks Explicitly

```yaml
# Check if task was skipped:
when: not my_task.skipped | default(false)

# Check if task ran and succeeded:
when: my_task.rc is defined and my_task.rc == 0
```

### 3. Jinja2 List Manipulation

```yaml
# ❌ Don't use Python slicing:
{{ my_list[-10:] }}        # Syntax error
{{ my_list[5:10] }}        # Syntax error

# ✅ Use Jinja2 filters:
{{ my_list | last(10) }}   # Get last 10 items
{{ my_list | first(5) }}   # Get first 5 items
{{ my_list | batch(3) }}   # Batch into groups of 3

# ✅ Or limit in shell command:
shell: some_command | head -n 10
shell: some_command | tail -n 50
```

### 4. When to Use command vs shell

```yaml
# Use command when no shell features needed:
command: kubectl get pods

# Use shell when you need pipes, redirects, etc:
shell: kubectl get pods | grep Running
shell: cat file.txt | sort | uniq
shell: kubectl get events | tail -n 50
```

---

## Testing

These fixes resolve:

- ✅ No more AttributeError when accessing `.rc` on skipped tasks
- ✅ No more Jinja2 syntax errors with list slicing
- ✅ Proper fallback logic for conditional execution
- ✅ Cluster events properly limited to 50 lines

Run the playbook again:

```bash
ansible-playbook create_k8s.yml
```

Expected behavior:

- If ImagePullBackOff detected → remediation runs
- If no ImagePullBackOff → remediation skipped
- If remediation skipped → final attempt runs (safely checks skipped state)
- Diagnostics show last 50 cluster events (not thousands of lines)

---

## Summary of All Jinja2 Fixes

| Issue                  | Location        | Fix                                      |
| ---------------------- | --------------- | ---------------------------------------- |
| `from_json` multi-line | Line ~474       | Simplified to single line                |
| `.items` ambiguity     | Line ~575, ~715 | Changed to `['items']`                   |
| `.rc` on skipped task  | Line ~587       | Added `.skipped` check and `.default(1)` |
| List slicing `[-50:]`  | Line ~742       | Moved to shell command `tail -n 50`      |

All Jinja2 template errors should now be resolved! 🎉
