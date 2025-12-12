# Workspaces

Multi-tenant architecture for data isolation.

## Overview

ReadyKit uses workspaces as the foundation for multi-tenancy. Every piece of business data belongs to a workspace, ensuring complete isolation between customers.

## How It Works

```
User → Membership → Workspace
         ↓
    role: admin | member
```

- Users can belong to multiple workspaces
- Each workspace has one owner (the creator)
- Members have either `admin` or `member` role
- All business data is scoped to a workspace

## Automatic Workspace Creation

When users sign up (email or OAuth), a workspace is automatically created for them:

```python
# Happens automatically on registration/OAuth
workspace = WorkspaceService.create_workspace(
    name="",           # Ignored when auto_name=True
    owner_user=user,
    auto_name=True     # Generates "John's Workspace" from user data
)
```

The auto-generated name uses:
1. User's name if available → `"John's Workspace"`
2. Email prefix as fallback → `"john's Workspace"`
3. Default → `"My Workspace"`

## Invisible Workspaces for Solo Users

Solo users never see workspace UI. When a user has only one workspace, they're automatically redirected to it:

```python
workspaces = current_user.get_workspaces()
if len(workspaces) == 1:
    return redirect(url_for("portal.switch_workspace", workspace_id=workspaces[0].id))
```

Team features (member list, billing, settings) only appear when users invite team members.

## Route Protection

**Always** use the `@require_workspace_access()` decorator on workspace routes:

```python
from enferno.services.workspace import require_workspace_access
from flask import g

@app.get("/workspace/<int:workspace_id>/projects/")
@require_workspace_access("member")  # or "admin" for admin-only routes
def list_projects(workspace_id):
    # Security checks already performed:
    # ✓ User is authenticated
    # ✓ Workspace exists
    # ✓ User is a member
    # ✓ User has required role

    # Access workspace context:
    workspace = g.current_workspace
    role = g.user_workspace_role

    return render_template("projects.html", workspace=workspace)
```

### What the Decorator Does

1. Verifies user is authenticated
2. Fetches workspace from URL parameter
3. Checks user has membership in workspace
4. Validates role requirement (`admin` or `member`)
5. Sets session and context:
   - `session["current_workspace_id"]`
   - `g.current_workspace`
   - `g.user_workspace_role`

## Creating Workspace-Scoped Models

All business data should inherit from `WorkspaceScoped` mixin:

```python
from enferno.services.workspace import WorkspaceScoped
from enferno.extensions import db

class Project(db.Model, WorkspaceScoped):
    id = db.Column(db.Integer, primary_key=True)
    workspace_id = db.Column(db.Integer, db.ForeignKey('workspace.id'), nullable=False)
    name = db.Column(db.String(100), nullable=False)
    description = db.Column(db.Text)

    # Relationship to workspace
    workspace = db.relationship('Workspace', backref='projects')
```

::: warning
Always include `workspace_id` as a non-nullable foreign key. This ensures data isolation at the database level.
:::

## Querying Workspace Data

The `WorkspaceScoped` mixin provides convenient query methods:

```python
# Get all records for current workspace
projects = Project.for_current_workspace()

# Get specific record (workspace-scoped)
project = Project.get_by_id(project_id)  # Returns None if not in current workspace
```

For custom queries, use the helper function:

```python
from enferno.services.workspace import workspace_query

# Build workspace-scoped query
stmt = workspace_query(Project).where(Project.status == "active")
active_projects = db.session.execute(stmt).scalars().all()
```

## Workspace Service Methods

```python
from enferno.services.workspace import WorkspaceService

# Create workspace
ws = WorkspaceService.create_workspace(
    name="Acme Corp",
    owner_user=user,
    auto_name=False
)

# Add member
WorkspaceService.add_member(workspace_id, user, role="member")
db.session.commit()  # Caller must commit

# Remove member (cannot remove owner)
WorkspaceService.remove_member(workspace_id, user_id)

# Change role (cannot change owner's role)
WorkspaceService.update_member_role(workspace_id, user_id, "admin")
```

## User Model Methods

```python
# Get all workspaces user belongs to
workspaces = current_user.get_workspaces()

# Get user's role in a specific workspace
role = current_user.get_workspace_role(workspace_id)  # Returns "admin" or "member"
```

## Template Context

`get_current_workspace()` is globally available in all templates:

```html
{% if get_current_workspace() %}
  <h1>{{ get_current_workspace().name }}</h1>
  <p>Plan: {{ get_current_workspace().plan }}</p>
{% endif %}
```

## Security Best Practices

::: details Always use the decorator
Never access workspace data without `@require_workspace_access()`. The decorator handles all security checks.
:::

::: details Don't trust session alone
`session["current_workspace_id"]` can be stale. The decorator re-validates membership on every request.
:::

::: details Include workspace_id in queries
Even with the mixin, always scope queries to workspace_id. Defense in depth.
:::

::: details Validate ownership for destructive actions
For delete/modify operations, verify the record's workspace_id matches the current workspace.
:::
