---
description: How CIPP decides a user's effective permissions when they hold multiple roles
---

# How CIPP Evaluates Roles

CIPP assigns access through **built-in roles** and **custom roles**, and a single user can end up holding several of them at once — especially when roles are mapped to Entra ID groups and a user is a member of more than one group. This page explains, in plain terms, how those roles combine into the permissions a user actually gets.

For instructions on *creating* roles and mapping them to Entra groups, see [Adding Users and Managing Roles](../self-hosting-guide/roles.md).

## The mental model

> **Built-in roles are the ceiling. Custom roles are the filter.**

* A **built-in role** (`readonly`, `editor`, `admin`, `superadmin`) sets the **maximum** a user can ever do.
* A **custom role** does not grant a ceiling of its own. It **narrows down** whatever the built-in role already allows.

Everything below follows from those two sentences.

## Built-in roles: highest wins

Built-in roles rank from least to most privileged:

`readonly` → `editor` → `admin` → `superadmin`

If a user is a member of several Entra groups that map to **different built-in roles, they do not add together — the single highest role wins.** A user who is both `editor` and `readonly` is simply an `editor`; the `readonly` membership contributes nothing.

| Built-in role | Effective access |
| ------------- | ---------------- |
| `readonly`    | Read/list only. No settings, no admin. |
| `editor`      | Read **and** write, except system settings and Standards. |
| `admin`       | Everything except a small set of super-admin-only settings. |
| `superadmin`  | Everything, including high-privilege settings. |

{% hint style="warning" %}
**`admin` and `superadmin` ignore custom roles entirely.** If *any* group grants a user `admin` or `superadmin`, they receive the full built-in permission set and every custom role on that user is disregarded. You cannot use a custom role to restrict an admin — assign `editor` or `readonly` instead and filter from there.
{% endhint %}

## Custom roles: they filter, they don't grant a ceiling

A custom role only means something when the user **also** holds the `editor` or `readonly` built-in role. On its own — with no base role — a custom role grants no access.

When paired with `editor` or `readonly`:

* The built-in role decides the **maximum** (the ceiling).
* The custom role **removes** everything the user is not explicitly granted, and can further scope access by **tenant** (Allowed/Blocked Tenants) and by **endpoint** (Blocked Endpoints).
* A custom role can never raise a user **above** the ceiling. Granting *Read/Write* in a custom role to a `readonly` user still results in read-only access, because `readonly` is the ceiling.

{% hint style="info" %}
Any permission category you leave **undefined** in a custom role is treated as `None`. If you want to keep a capability the base role would otherwise allow, you must explicitly set it to `Read` or `Read/Write` in the custom role.
{% endhint %}

### Multiple custom roles on one user

If a user holds more than one custom role, their granted permissions are **combined (a union — the most permissive wins)**, and the `editor`/`readonly` ceiling still caps the result.

{% hint style="warning" %}
Tenant scope is evaluated **per custom role**, not pooled across them. For any single action, one custom role must grant **both** the required permission **and** access to the target tenant. A user cannot borrow the permission from one custom role and the tenant access from another.
{% endhint %}

## Worked examples

### Example 1 — Built-in roles only

Jane is a member of two Entra groups:

* `CIPP-Editors` → mapped to `editor`
* `CIPP-Read` → mapped to `readonly`

**Result:** the higher role wins, so Jane is an **editor**. The `readonly` membership changes nothing.

### Example 2 — Built-in role + custom role

Mark is a member of two Entra groups:

* `CIPP-Editors` → mapped to `editor`
* `CIPP-Helpdesk` → mapped to a custom role that grants **Identity: Read/Write** only, with **Allowed Tenants = Contoso**

**Result:** `editor` sets the ceiling (read/write everything except settings and Standards). The custom role filters that down to **only Identity read/write**, and only for **Contoso**. Net effect: Mark can manage users in Contoso and nothing else.

If Mark were removed from `CIPP-Editors`, the custom role alone would grant him **no access**, because there is no base role for it to filter.

### Example 3 — Admin plus a restrictive custom role

Sara is a member of two Entra groups:

* `CIPP-Admins` → mapped to `admin`
* `CIPP-Helpdesk` → the restrictive custom role from Example 2

**Result:** `admin` wins outright and the custom role is **ignored**. Sara has full admin access. To actually limit her, remove the `admin` mapping and give her `editor` (or `readonly`) plus the custom role.

## Quick reference

| The user holds…​ | What they get |
| ----------------- | ------------- |
| Several built-in roles | The single **highest** role. |
| `admin` or `superadmin` (+ anything) | Full built-in access; **custom roles ignored**. |
| `editor`/`readonly` + one custom role | Built-in ceiling **filtered down** by the custom role's permissions, tenants, and blocked endpoints. |
| `editor`/`readonly` + several custom roles | **Union** of the custom roles' grants, still capped by the built-in ceiling; tenant scope checked per role. |
| A custom role with **no** base role | **No access.** |

{% hint style="info" %}
Because `admin`/`superadmin` bypass custom roles, and because a lone custom role grants nothing, the recommended pattern is: map a base role (`editor` or `readonly`) to one Entra group and the custom role to another, then add users to **both**. See [Custom Roles](../self-hosting-guide/roles.md#custom-roles) for the full setup steps.
{% endhint %}

***

{% include "../../.gitbook/includes/feature-request.md" %}
