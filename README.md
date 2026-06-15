# ZeroMaxShop — Full-Stack E-Commerce Platform

🔗 **Live site:** [zeromaxshop.com](https://zeromaxshop.com)

> Note: This repository contains documentation only. The source code is 
> private as the platform handles real customer data and live payments.

## Overview

ZeroMaxShop is a production e-commerce platform I designed, built, and 
deployed independently — handling live Stripe payments, shipping, 
transactional email, and a custom buyer/seller offer-negotiation system.

## Tech Stack

Python, Flask, SQLAlchemy, MySQL, JavaScript, Stripe API, Shippo API, 
Resend API, Railway, Backblaze B2

## Key Features & Architecture

- **Payments:** Stripe Checkout integration with webhook handling for 
  live payment processing and automatic order creation
- **Database:** Relational MySQL schema (SQLAlchemy ORM) covering users, 
  products, variants, orders, carts, and a custom multi-state 
  offer-negotiation system
- **Shipping & Email:** Shippo API for shipping rates/labels/tracking; 
  Resend API for transactional emails (verification, order confirmation, 
  shipping updates)
- **Security:** CSRF protection, session security, and a crash-monitoring 
  system with real-time admin email alerts
- **Deployment:** CI/CD via GitHub → Railway, custom domain, HTTPS

## Code Sample

Stripe webhook handler — verifies signatures, prevents duplicate event 
processing, creates orders on successful payment, and alerts admins if 
a payment succeeds but order creation fails:
```
@app.route("/stripe/webhook", methods=["POST"])
@csrf.exempt
def stripe_webhook():
    payload = request.get_data()
    sig_header = request.headers.get("Stripe-Signature")

    try:
        stripe.Webhook.construct_event(payload, sig_header, STRIPE_WEBHOOK_SECRET)
    except ValueError:
        return "Invalid payload", 400
    except stripe.error.SignatureVerificationError:
        return "Invalid signature", 400

    event = json.loads(payload.decode("utf-8"))
    stripe_event_id = event.get("id")
    event_type = event.get("type")

    if not stripe_event_id or not event_type:
        return "Invalid event", 400

    with Session() as db_session:
        # Idempotency check — prevent duplicate processing
        if stripe_webhook_logic.already_processed_event(db_session, stripe_event_id):
            return jsonify({"status": "already_processed"}), 200

        stripe_webhook_logic.record_received_event(db_session, stripe_event_id, event_type)
        db_session.commit()

    with Session() as db_session:
        try:
            if event_type != "checkout.session.completed":
                stripe_webhook_logic.mark_event_skipped(db_session, stripe_event_id, f"Unhandled: {event_type}")
                db_session.commit()
                return jsonify({"status": "skipped", "event_type": event_type}), 200

            checkout_session = event["data"]["object"]
            order = order_logic.process_paid_checkout_session(db_session, checkout_session)

            if order:
                stripe_webhook_logic.mark_event_processed(db_session, stripe_event_id, order.id)
                email_logic.send_buyer_order_confirmation_email(db_session, order)
            else:
                # Paid but no order created — flag for admin recovery
                stripe_webhook_logic.mark_event_skipped(db_session, stripe_event_id, "Paid but no order created")
                admin_user = message_logic.get_admin_user(db_session)
                if admin_user:
                    notification_logic.create_notification(
                        db_session, user_id=admin_user.id,
                        notification_type="stripe_recovery_needed",
                        title="Paid checkout needs recovery",
                        message=f"Stripe checkout {checkout_session.get('id')} was paid but no order was created.",
                        link_url=url_for("admin_stripe_payment_tools")
                    )

            db_session.commit()
            return jsonify({"status": "success", "event_type": event_type}), 200

        except Exception as error:
            db_session.rollback()
            stripe_webhook_logic.mark_event_failed(db_session, stripe_event_id, str(error))
            db_session.commit()
            return jsonify({"status": "failed", "error": str(error)}), 500
