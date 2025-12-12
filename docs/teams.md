# Teams

Member management and collaboration.

## Overview

ReadyKit supports team collaboration within workspaces. Users can invite team members, assign roles, and manage access - all while maintaining data isolation.

## Roles

Each workspace member has one of two roles:

| Role | Permissions |
|------|-------------|
| **Admin** | Full access: billing, member management, settings, all data |
| **Member** | Standard access: view and edit workspace data |

The workspace **owner** is always an admin and cannot be removed or demoted.

## Adding Team Members

### Using WorkspaceService

```python
from enferno.services.workspace import WorkspaceService
from enferno.user.models import User
from enferno.extensions import db

# Find or create user
user = db.session.execute(
    db.select(User).where(User.email == "teammate@example.com")
).scalar_one_or_none()

if user:
    # Add existing user to workspace
    WorkspaceService.add_member(workspace_id, user, role="member")
    db.session.commit()
```

### Route Example

```python
@app.route("/workspace/<int:workspace_id>/members/add/", methods=["POST"])
@require_workspace_access("admin")  # Only admins can add members
def add_member(workspace_id):
    email = request.form.get("email")
    role = request.form.get("role", "member")

    user = db.session.execute(
        db.select(User).where(User.email == email)
    ).scalar_one_or_none()

    if not user:
        flash("User not found. They must register first.")
        return redirect(url_for("portal.workspace_members", workspace_id=workspace_id))

    try:
        WorkspaceService.add_member(workspace_id, user, role=role)
        db.session.commit()
        flash(f"Added {email} as {role}")
    except ValueError as e:
        flash(str(e))

    return redirect(url_for("portal.workspace_members", workspace_id=workspace_id))
```

## Removing Members

```python
@app.route("/workspace/<int:workspace_id>/members/<int:user_id>/remove/", methods=["POST"])
@require_workspace_access("admin")
def remove_member(workspace_id, user_id):
    try:
        WorkspaceService.remove_member(workspace_id, user_id)
        flash("Member removed")
    except ValueError as e:
        flash(str(e))  # "Cannot remove workspace owner"

    return redirect(url_for("portal.workspace_members", workspace_id=workspace_id))
```

::: warning
The workspace owner cannot be removed. Attempting to do so raises a `ValueError`.
:::

## Changing Roles

```python
@app.route("/workspace/<int:workspace_id>/members/<int:user_id>/role/", methods=["POST"])
@require_workspace_access("admin")
def change_role(workspace_id, user_id):
    new_role = request.form.get("role")

    try:
        WorkspaceService.update_member_role(workspace_id, user_id, new_role)
        flash("Role updated")
    except ValueError as e:
        flash(str(e))  # "Cannot change workspace owner's role"

    return redirect(url_for("portal.workspace_members", workspace_id=workspace_id))
```

## Listing Members

```python
@app.route("/workspace/<int:workspace_id>/members/")
@require_workspace_access("admin")
def workspace_members(workspace_id):
    workspace = g.current_workspace

    # Get all memberships with user data
    members = db.session.execute(
        db.select(Membership, User)
        .join(User, Membership.user_id == User.id)
        .where(Membership.workspace_id == workspace_id)
    ).all()

    return render_template(
        "workspace/members.html",
        workspace=workspace,
        members=members
    )
```

## Role-Based Route Protection

Use the `required_role` parameter in the decorator:

```python
# Any member can access
@app.route("/workspace/<int:workspace_id>/projects/")
@require_workspace_access("member")
def list_projects(workspace_id):
    pass

# Only admins can access
@app.route("/workspace/<int:workspace_id>/settings/")
@require_workspace_access("admin")
def workspace_settings(workspace_id):
    pass
```

## Checking User's Role

```python
# In routes (after decorator)
if g.user_workspace_role == "admin":
    # Show admin controls
    pass

# Via User model
role = current_user.get_workspace_role(workspace_id)
```

In templates:

```html
{% if g.user_workspace_role == 'admin' %}
  <a href="{{ url_for('portal.workspace_settings', workspace_id=workspace.id) }}">
    Settings
  </a>
{% endif %}
```

## Owner Protection

The workspace owner has special protections:

```python
# These will raise ValueError:
WorkspaceService.remove_member(workspace_id, owner.id)
# → "Cannot remove workspace owner"

WorkspaceService.update_member_role(workspace_id, owner.id, "member")
# → "Cannot change workspace owner's role"
```

## Template Example

```html
<h2>Team Members</h2>

<table>
  <thead>
    <tr>
      <th>Email</th>
      <th>Role</th>
      <th>Actions</th>
    </tr>
  </thead>
  <tbody>
    {% for membership, user in members %}
    <tr>
      <td>
        {{ user.email }}
        {% if user.id == workspace.owner_id %}
          <span class="badge">Owner</span>
        {% endif %}
      </td>
      <td>{{ membership.role }}</td>
      <td>
        {% if user.id != workspace.owner_id and g.user_workspace_role == 'admin' %}
          <form method="POST" action="{{ url_for('portal.remove_member', workspace_id=workspace.id, user_id=user.id) }}">
            <button type="submit">Remove</button>
          </form>
        {% endif %}
      </td>
    </tr>
    {% endfor %}
  </tbody>
</table>
```
