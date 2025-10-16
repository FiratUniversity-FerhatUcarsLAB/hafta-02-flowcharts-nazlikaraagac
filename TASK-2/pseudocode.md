CONST RESERVATION_TTL = 15 minutes

FUNCTION AddToCart(user_or_session, product_id, qty):
    IF qty <= 0:
        RETURN Error("Geçersiz adet")
    product = DB.GetProduct(product_id)
    IF product == null OR !product.active:
        RETURN Error("Ürün mevcut değil")
    cart = GetOrCreateCart(user_or_session)

    // Basit kontrol: anlık görünüm. Gerçek garanti rezervasyon gerektirir.
    inventory = DB.GetInventory(product_id)
    IF inventory.available < qty:
        RETURN Error("Yeterli stok yok - mevcut: " + inventory.available)

    DB.UpsertCartItem(cart.id, product_id, product.price_cents, qty)
    // Opsiyonel: kısa süreli rezervasyon
    success = TryCreateReservation(cart.id, product_id, qty)
    IF NOT success:
        // Rezervasyon başarısızsa, yine de ekle ama kullanıcıyı uyar
        RETURN Warning("Sepete eklendi, ancak stok hemen doğrulanamadı")

    RETURN Success(GetCartSummary(cart.id))

FUNCTION TryCreateReservation(cart_id, product_id, qty):
    // Atomik update ile rezervasyon
    affected = DB.Execute(
      "UPDATE inventory SET available = available - :qty, reserved = reserved + :qty WHERE product_id = :pid AND available >= :qty",
      {qty: qty, pid: product_id}
    )
    IF affected == 1:
        reservation = {
          product_id: product_id,
          qty: qty,
          cart_id: cart_id,
          expires_at: NOW() + RESERVATION_TTL,
          status: "active"
        }
        DB.Insert("reservations", reservation)
        RETURN true
    ELSE:
        RETURN false

FUNCTION ApplyCoupon(cart_id, coupon_code, user_id):
    coupon = DB.GetCouponByCode(coupon_code)
    IF coupon == null OR NOT coupon.isActive():
        RETURN Error("Kupon geçersiz")
    cart = DB.GetCart(cart_id)
    subtotal = cart.CalculateSubtotal()

    IF coupon.min_cart_value_cents > subtotal:
        RETURN Error("Sepet tutarı kupon için yetersiz")
    IF coupon.usage_limit_reached():
        RETURN Error("Kupon kullanım limiti dolmuş")

    discount = CalculateDiscount(coupon, cart)
    // snapshot discount to cart
    DB.UpdateCart(cart_id, {coupon_code: coupon_code, discount_cents: discount})
    RETURN Success({new_total: cart.RecalculateTotal()})

FUNCTION Checkout(cart_id, user_or_session, billing_address, shipping_address, shipping_method, payment_method, idempotency_key):
    // Do not trust client totals; server-side recalc mandatory
    cart = DB.GetCart(cart_id)
    RecalculatePrices(cart)  // prices, coupon validity, taxes, shipping cost

    IF cart.has_price_changes_from_client:
        RETURN Error("Fiyat veya stok değişti, lütfen tekrar kontrol et")

    BEGIN TRANSACTION
    // Reserve all items atomically
    FOR item IN cart.items:
        ok = ReserveOrFail(item.product_id, item.quantity, cart.id)
        IF NOT ok:
            ROLLBACK
            RETURN Error("Stok yetersiz: " + item.product_id)

    order = DB.Insert("orders", {
        user_id: user_or_session.user_id,
        status: "pending",
        total_cents: cart.total_cents,
        currency: cart.currency,
        created_at: NOW()
    })
    FOR item IN cart.items:
        DB.Insert("order_items", {...snapshot of item...})

    payment = DB.Insert("payments", {
        order_id: order.id,
        provider: payment_method.provider,
        status: "initiated",
        idempotency_key: idempotency_key,
        amount_cents: order.total_cents
    })
    COMMIT TRANSACTION

    // Create payment intent with provider (example)
    provider_response = PaymentProvider.CreatePaymentIntent(amount=order.total_cents,
                                                           currency=order.currency,
                                                           idempotency_key=idempotency_key,
                                                           return_url=Callbacks.PaymentReturnUrl(order.id))
    IF provider_response.needs_redirect:
        RETURN Redirect(provider_response.redirect_url)
    ELSE IF provider_response.status == "succeeded":
        HandlePaymentSuccess(provider_response, order.id, payment.id)
        RETURN Success("Ödeme tamamlandı")
    ELSE:
        RETURN Error("Ödeme başlatılamadı")

FUNCTION ReserveOrFail(product_id, qty, cart_id):
    affected = DB.Execute("UPDATE inventory SET available = available - :qty, reserved = reserved + :qty WHERE product_id = :pid AND available >= :qty", {...})
    IF affected == 1:
        DB.Insert("reservations", {product_id, qty, cart_id, expires_at: NOW()+RESERVATION_TTL, status: "active"})
        RETURN true
    ELSE:
        RETURN false

HTTP ENDPOINT /webhook/payment:
    body = Request.body
    IF NOT VerifySignature(body, headers):
        RETURN 400

    event = ParseEvent(body)
    idempotency_key = event.metadata.idempotency_key OR event.id
    payment_rec = DB.GetPaymentByProviderId(event.payment_id) OR DB.GetPaymentByIdempotencyKey(idempotency_key)

    IF payment_rec != null AND payment_rec.status == "paid":
        RETURN 200 // idempotent response

    IF event.status == "succeeded":
        BEGIN TRANSACTION
        UpdatePayment(payment_rec.id, status="paid", provider_payment_id=event.payment_id)
        UpdateOrder(payment_rec.order_id, status="paid")
        // finalize reservations -> deduct reserved, clear reservation rows
        items = DB.GetOrderItems(payment_rec.order_id)
        FOR item IN items:
           DB.Execute("UPDATE inventory SET reserved = reserved - :qty WHERE product_id = :pid", {...})
           // note: available was already decreased at reservation time
        CreateInvoice(payment_rec.order_id)
        EnqueueShipment(payment_rec.order_id)
        COMMIT
        RETURN 200
    ELSE IF event.status IN ["failed", "cancelled"]:
        BEGIN TRANSACTION
        UpdatePayment(payment_rec.id, status="failed")
        UpdateOrder(payment_rec.order_id, status="failed")
        // release reservations
        items = DB.GetOrderItems(payment_rec.order_id)
        FOR item IN items:
           DB.Execute("UPDATE inventory SET reserved = reserved - :qty, available = available + :qty WHERE product_id = :pid", {...})
        COMMIT
        NotifyUser(order.user_id, "Ödeme başarısız")
        RETURN 200
