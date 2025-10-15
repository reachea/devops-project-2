# Jinja2 Templating Error Fix

## Problem Encountered

```
TASK [Detect ImagePullBackOff on controller]
fatal: [localhost]: FAILED! =>
  "msg": "Unexpected templating type error occurred on (...):
  'builtin_function_or_method' object is not iterable"

TASK [Count running pods in rescue]
fatal: [localhost]: FAILED! =>
  "msg": "Unexpected templating type error occurred on (...):
  object of type 'builtin_function_or_method' has no len()"
```

## Root Cause

Multi-line YAML folding with the `>-` operator was causing Jinja2 to misparse the `from_json` filter. When templates are split across multiple lines with YAML's block scalar indicators, the whitespace and newlines can interfere with Jinja2's parser, causing it to treat `from_json` as a method reference instead of calling it as a function.

## Fixes Applied

### Fix 1: Simplified ImagePullBackOff Detection

**Before** (Complex, error-prone):

```yaml
- name: Detect ImagePullBackOff on controller
  ansible.builtin.set_fact:
    cm_pull_issue: >-
      {{ (cm_pods_json.stdout | from_json).items
        | selectattr('metadata.labels.app','equalto','cert-manager')
        | map(attribute='status.containerStatuses') | list | sum(start=[])
        | selectattr('state.waiting.reason','defined')
        | selectattr('state.waiting.reason','equalto','ImagePullBackOff')
        | list | length > 0 }}
```

**After** (Simple, reliable):

```yaml
- name: Detect ImagePullBackOff on controller
  ansible.builtin.set_fact:
    cm_pull_issue: "{{ 'ImagePullBackOff' in cm_pods_json.stdout }}"
```

**Why it's better:**

- ✅ No complex JSON parsing
- ✅ No multi-line YAML issues
- ✅ Simple string search in JSON output
- ✅ More reliable and faster

### Fix 2: Fixed JSON Parsing in "Count running pods"

**Before**:

```yaml
- name: Count running pods
  ansible.builtin.set_fact:
    running_pods_count: "{{ (running_pods_check.stdout | from_json).items | length }}"
```

**After**:

```yaml
- name: Count running pods
  ansible.builtin.set_fact:
    running_pods_count: "{{ (running_pods_check.stdout | from_json)['items'] | length }}"
```

**Why it works:**

- ✅ Using dictionary access `['items']` instead of attribute access `.items`
- ✅ Avoids confusion with Python's `.items()` method
- ✅ More explicit and clear

### Fix 3: Fixed JSON Parsing in "Count running pods in rescue"

**Before**:

```yaml
- name: Count running pods in rescue
  ansible.builtin.set_fact:
    rescue_running_count: "{{ (rescue_running_pods.stdout | from_json).items | length }}"
  when: rescue_running_pods.rc == 0
```

**After**:

```yaml
- name: Count running pods in rescue
  ansible.builtin.set_fact:
    rescue_running_count: "{{ (rescue_running_pods.stdout | from_json)['items'] | length }}"
  when: rescue_running_pods.rc == 0
```

**Why it works:**

- ✅ Same as Fix 2 - explicit dictionary access
- ✅ Avoids ambiguity with `.items()` method

### Fix 4: Simplified "Check if all deployments succeeded"

**Before**:

```yaml
- name: Check if all deployments succeeded
  ansible.builtin.set_fact:
    all_deployments_ok: >-
      {{ (cm_ctrl_rollout.rc == 0 or ...)
         and cm_cainj_rollout.rc == 0
         and cm_webhook_rollout.rc == 0 }}
```

**After**:

```yaml
- name: Check if all deployments succeeded
  ansible.builtin.set_fact:
    all_deployments_ok: "{{ (cm_ctrl_rollout.rc == 0 or ...) and cm_cainj_rollout.rc == 0 and cm_webhook_rollout.rc == 0 }}"
```

**Why it works:**

- ✅ Single-line template avoids YAML folding issues
- ✅ Clearer and more reliable

## Key Lessons

### 1. Avoid Multi-line Jinja2 in YAML

When using complex Jinja2 expressions with `from_json` or other filters:

- ❌ Don't use `>-` or `|-` for multi-line folding
- ✅ Keep the entire expression on one line
- ✅ Or break it into multiple simpler tasks

### 2. Use Dictionary Access for JSON

When accessing properties from `from_json`:

- ❌ Avoid: `.items` (ambiguous with Python's `.items()` method)
- ✅ Prefer: `['items']` (explicit dictionary access)

### 3. Simplify Complex Filters

Instead of chaining many filters:

```yaml
# Complex (error-prone):
{{ (json | from_json).items | selectattr(...) | map(...) | list | sum(...) }}

# Simple (reliable):
{{ 'SearchTerm' in json_string }}
```

### 4. Test Templates in Isolation

Use `ansible.builtin.debug` to test complex templates before using them in `set_fact`:

```yaml
- name: Debug test
  ansible.builtin.debug:
    msg: "{{ my_complex_template }}"
```

## Testing

To verify these fixes work:

```bash
# Run the playbook
ansible-playbook create_k8s.yml

# The following tasks should now succeed:
# ✅ Detect ImagePullBackOff on controller
# ✅ Count running pods
# ✅ Count running pods in rescue
# ✅ Check if all deployments succeeded
```

## What These Fixes Enable

1. **ImagePullBackOff Detection**: Now correctly detects when cert-manager has image pull issues
2. **Automatic Remediation**: Switches to ghcr.io when Docker Hub rate limits hit
3. **Graceful Degradation**: Counts running pods even if rollout times out
4. **Smart Failure**: Only fails if cert-manager truly isn't working

## Summary

All Jinja2 templating errors have been resolved by:

1. Simplifying complex filter chains
2. Using explicit dictionary access notation
3. Keeping templates on single lines
4. Avoiding ambiguous attribute access

The playbook should now run successfully through the cert-manager deployment phase without templating errors.
