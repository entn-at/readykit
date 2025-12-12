# Billing

Stripe integration for subscriptions and payments.

## Overview

ReadyKit uses Stripe's hosted pages for billing - no custom checkout UI to build or maintain. Users upgrade via Stripe Checkout and manage subscriptions through the Stripe Customer Portal.

## Plans

Out of the box, ReadyKit supports two plans:

| Plan | Features |
|------|----------|
| **Free** | Basic access, limited features |
| **Pro** | Full access, all features |

Plans are stored on the `Workspace` model, not the user. This means the entire team shares the same plan.

## Stripe Setup

### 1. Get Your Keys

From [Stripe Dashboard](https://dashboard.stripe.com):

1. **API Keys** → Copy your secret key and publishable key
2. **Products** → Create a product with a recurring price
3. **Webhooks** → Add endpoint (see below)

### 2. Configure Environment

```bash
# .env
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_PRO_PRICE_ID=price_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Display values (shown in UI)
PRO_PRICE_DISPLAY=$29
PRO_PRICE_INTERVAL=month
```

### 3. Set Up Webhook

In Stripe Dashboard → Webhooks → Add endpoint:

- **URL**: `https://yourdomain.com/api/webhooks/stripe/webhook`
- **Events to listen for**:
  - `checkout.session.completed`
  - `customer.subscription.deleted`
  - `invoice.payment_failed`

Copy the signing secret to `STRIPE_WEBHOOK_SECRET`.

## How Billing Works

### Upgrade Flow

```
User clicks "Upgrade"
    → Create Stripe Checkout session
    → Redirect to Stripe
    → User completes payment
    → Stripe redirects to success URL
    → Validate session_id
    → Upgrade workspace to Pro
```

```python
from enferno.services.billing import HostedBilling

@app.route("/workspace/<int:workspace_id>/upgrade/")
@require_workspace_access("admin")
def upgrade(workspace_id):
    session = HostedBilling.create_upgrade_session(
        workspace_id=workspace_id,
        user_email=current_user.email,
        base_url=request.host_url
    )
    return redirect(session.url)
```

### Success Callback

The success URL includes a `session_id` that's validated server-side:

```python
@app.route("/billing/success")
def billing_success():
    session_id = request.args.get("session_id")
    workspace_id = HostedBilling.handle_successful_payment(session_id)

    if workspace_id:
        flash("Welcome to Pro!")
        return redirect(url_for("portal.workspace_settings", workspace_id=workspace_id))

    flash("Payment processing failed")
    return redirect(url_for("portal.dashboard"))
```

::: info
The `session_id` is the security token. Always validate it via Stripe API before upgrading - never trust URL parameters directly.
:::

### Manage Billing (Customer Portal)

Existing Pro users can manage their subscription through Stripe's Customer Portal:

```python
@app.route("/workspace/<int:workspace_id>/billing/")
@require_workspace_access("admin")
def manage_billing(workspace_id):
    workspace = g.current_workspace

    if not workspace.stripe_customer_id:
        return redirect(url_for("portal.upgrade", workspace_id=workspace_id))

    session = HostedBilling.create_portal_session(
        customer_id=workspace.stripe_customer_id,
        workspace_id=workspace_id,
        base_url=request.host_url
    )
    return redirect(session.url)
```

## Webhook Handlers

Webhooks update workspace status automatically when billing changes:

| Event | Action |
|-------|--------|
| `checkout.session.completed` | Upgrade workspace to Pro, save customer_id |
| `customer.subscription.deleted` | Downgrade workspace to Free |
| `invoice.payment_failed` | Downgrade workspace to Free |

### Idempotency

Webhooks are idempotent - duplicate events are safely ignored using the `StripeEvent` model:

```python
# Duplicate events are caught by unique constraint
try:
    db.session.add(StripeEvent(event_id=event_id, event_type=event.type))
    db.session.commit()
except IntegrityError:
    db.session.rollback()
    return "OK", 200  # Already processed
```

## Gating Features

Use the `@requires_pro_plan` decorator to restrict features:

```python
from enferno.services.billing import requires_pro_plan

@app.route("/workspace/<int:workspace_id>/advanced-feature/")
@require_workspace_access("member")
@requires_pro_plan
def advanced_feature(workspace_id):
    # Only Pro workspaces can access this
    return render_template("advanced.html")
```

For API endpoints, it returns a `402 Payment Required` response:

```json
{"error": "Pro plan required"}
```

For web pages, it redirects to the upgrade page.

## Testing Locally

Use [Stripe CLI](https://stripe.com/docs/stripe-cli) to test webhooks locally:

```bash
# Install Stripe CLI
brew install stripe/stripe-cli/stripe

# Login
stripe login

# Forward webhooks to local server
stripe listen --forward-to localhost:5000/api/webhooks/stripe/webhook

# In another terminal, trigger test events
stripe trigger checkout.session.completed
stripe trigger customer.subscription.deleted
```

## Checking Plan Status

```python
# In routes (after @require_workspace_access)
workspace = g.current_workspace
if workspace.is_pro:
    # Pro features
    pass
```

```html
<!-- In templates -->
{% if get_current_workspace().is_pro %}
  <span class="badge">Pro</span>
{% else %}
  <a href="{{ url_for('portal.upgrade', workspace_id=workspace.id) }}">
    Upgrade to Pro
  </a>
{% endif %}
```

## Workspace Model Fields

```python
class Workspace(db.Model):
    # Billing fields
    plan = db.Column(db.String(20), default="free")  # "free" or "pro"
    stripe_customer_id = db.Column(db.String(255))    # Stripe customer ID
    upgraded_at = db.Column(db.DateTime)              # When they upgraded

    @property
    def is_pro(self):
        return self.plan == "pro"
```
