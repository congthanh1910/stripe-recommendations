# Cách Tôi Giữ Bình Tĩnh Khi Tích Hợp Stripe

> [!NOTE]  
> **Cập nhật (2025-02-07)**  
> Stripe đã mời tôi phát biểu với CEO của họ tại buổi all-hands toàn công ty. Họ rất cởi mở với những feedback của tôi, và tôi thấy một tương lai tươi sáng nơi tất cả những thứ này không còn cần thiết nữa. Cho đến lúc đó, tôi vẫn cho rằng đây là cách tốt nhất để setup thanh toán trong các SaaS app của bạn.

Tôi đã setup Stripe quá nhiều lần. Tôi chưa bao giờ thích nó. Tôi đã nói chuyện với team Stripe về những hạn chế và họ nói sẽ fix...đến một lúc nào đó.

Cho đến lúc đó, đây là cách tôi khuyến nghị setup Stripe. Tôi không đề cập đến mọi thứ - hãy xem [những thứ vẫn là vấn đề của bạn](#những-thứ-vẫn-là-vấn-đề-của-bạn) để rõ hơn về những gì tôi KHÔNG hỗ trợ.

> Nếu bạn muốn giữ bình tĩnh khi tích hợp file upload, hãy xem sản phẩm của tôi [UploadThing](https://uploadthing.com/).

### Pre-requirements

- TypeScript
- Một loại JS backend nào đó
- Auth đang hoạt động (được verify trên JS backend của bạn)
- Một KV store (tôi dùng Redis, thường là [Upstash](https://upstash.com/?utm_source=theo), nhưng bất kỳ KV nào cũng được)

### General philosophy (Triết lý chung)

Theo tôi, vấn đề lớn nhất với Stripe là tình trạng "split brain" mà nó tất yếu đưa vào codebase của bạn. Khi customer checkout, "trạng thái của giao dịch" nằm trong Stripe. Sau đó bạn phải tự track giao dịch trong database của mình thông qua webhook.

Có [hơn 258 loại event](https://docs.stripe.com/api/events/types). Chúng đều có lượng data khác nhau. Thứ tự nhận chúng không được đảm bảo. Không cái nào nên được tin tưởng tuyệt đối. Rất dễ xảy ra trường hợp một payment thất bại trong Stripe nhưng lại hiển thị là "đã subscribe" trong app của bạn.

Những partial update và race condition này thực sự phiền toái. Tôi khuyến nghị tránh chúng hoàn toàn. Giải pháp của tôi rất đơn giản: _một hàm `syncStripeDataToKV(customerId: string)` duy nhất để sync toàn bộ data của một Stripe customer vào KV_.

Dưới đây là cách tôi (hầu như) tránh được những trạng thái xấu của Stripe.

## The Flow

Đây là tổng quan nhanh về "flow" tôi khuyến nghị. Chi tiết hơn ở bên dưới. Dù bạn không copy cách implement cụ thể của tôi, bạn vẫn nên đọc phần này. _Tôi hứa tất cả các bước này đều cần thiết. Bỏ qua bất kỳ bước nào sẽ khiến cuộc sống của bạn khó khăn hơn không cần thiết_

1. **FRONTEND:** Nút "Subscribe" nên gọi endpoint `"generate-stripe-checkout"` khi onClick
1. **NGƯỜI DÙNG:** Nhấn nút "subscribe" trong app của bạn
1. **BACKEND:** Tạo Stripe customer
1. **BACKEND:** Lưu liên kết giữa `customerId` của Stripe và `userId` của app
1. **BACKEND:** Tạo "checkout session" cho người dùng
   - Với return URL được đặt thành một route `/success` riêng trong app của bạn
1. **NGƯỜI DÙNG:** Thanh toán, subscribe, redirect về `/success`
1. **FRONTEND:** Khi load trang, trigger hàm `syncAfterSuccess` trên backend (gọi API, server action, rsc on load, tùy bạn)
1. **BACKEND:** Dùng `userId` để lấy Stripe `customerId` từ KV
1. **BACKEND:** Gọi `syncStripeDataToKV` với `customerId`
1. **FRONTEND:** Sau khi sync thành công, redirect người dùng đến nơi bạn muốn :)
1. **BACKEND:** Với [_tất cả các event liên quan_](#các-event-tôi-theo-dõi), gọi `syncStripeDataToKV` với `customerId`

Nghe có vẻ nhiều. Vì nó đúng là nhiều. Nhưng đây cũng là setup Stripe đơn giản nhất tôi từng thấy hoạt động được.

Hãy đi vào chi tiết các phần quan trọng.

### Checkout Flow

Điểm mấu chốt là đảm bảo **bạn luôn có customer được định nghĩa TRƯỚC KHI BẮT ĐẦU CHECKOUT**. Tính tạm thời của "customer" là một design flaw hoàn toàn và tôi không hiểu tại sao họ xây dựng Stripe như vậy.

Đây là ví dụ được điều chỉnh từ cách chúng tôi đang làm trong [T3 Chat](https://t3.chat).

```ts
export async function GET(req: Request) {
  const user = auth(req);

  // Lấy stripeCustomerId từ KV store của bạn
  let stripeCustomerId = await kv.get(`stripe:user:${user.id}`);

  // Tạo Stripe customer mới nếu user này chưa có
  if (!stripeCustomerId) {
    const newCustomer = await stripe.customers.create({
      email: user.email,
      metadata: {
        userId: user.id, // ĐỪNG QUÊN CÁI NÀY
      },
    });

    // Lưu quan hệ giữa userId và stripeCustomerId vào KV
    await kv.set(`stripe:user:${user.id}`, newCustomer.id);
    stripeCustomerId = newCustomer.id;
  }

  // LUÔN tạo checkout với stripeCustomerId. Họ nên bắt buộc điều này.
  const checkout = await stripe.checkout.sessions.create({
    customer: stripeCustomerId,
    success_url: "https://t3.chat/success",
    ...
  });
```

### syncStripeDataToKV

Đây là hàm sync toàn bộ data của một Stripe customer vào KV. Nó sẽ được dùng cả trong endpoint `/success` lẫn trong webhook handler `/api/stripe`.

Stripe API trả về rất nhiều data, phần lớn không thể serialize sang JSON. Tôi đã chọn ra phần "có khả năng cần dùng nhất" để bạn sử dụng, và có [type definition ở phần sau](#custom-stripe-subscription-type).

Cách implement của bạn sẽ khác nhau tùy thuộc vào việc bạn dùng subscription hay one-time purchase. Ví dụ dưới đây là với subscription (cũng từ [T3 Chat](https://t3.chat)).

```ts
// Nội dung của hàm này nên được bọc trong try/catch
export async function syncStripeDataToKV(customerId: string) {
  // Fetch subscription data mới nhất từ Stripe
  const subscriptions = await stripe.subscriptions.list({
    customer: customerId,
    limit: 1,
    status: "all",
    expand: ["data.default_payment_method"],
  });

  if (subscriptions.data.length === 0) {
    const subData = { status: "none" };
    await kv.set(`stripe:customer:${customerId}`, subData);
    return subData;
  }

  // Nếu user có thể có nhiều subscription, đó là vấn đề của bạn
  const subscription = subscriptions.data[0];

  // Lưu trạng thái subscription đầy đủ
  const subData = {
    subscriptionId: subscription.id,
    status: subscription.status,
    priceId: subscription.items.data[0].price.id,
    currentPeriodEnd: subscription.current_period_end,
    currentPeriodStart: subscription.current_period_start,
    cancelAtPeriodEnd: subscription.cancel_at_period_end,
    paymentMethod:
      subscription.default_payment_method &&
      typeof subscription.default_payment_method !== "string"
        ? {
            brand: subscription.default_payment_method.card?.brand ?? null,
            last4: subscription.default_payment_method.card?.last4 ?? null,
          }
        : null,
  };

  // Lưu data vào KV
  await kv.set(`stripe:customer:${customerId}`, subData);
  return subData;
}
```

### Endpoint `/success`

> [!NOTE]
> Dù điều này không "bắt buộc", rất có thể user sẽ quay lại site của bạn trước khi webhook đến nơi. Đây là một race condition khó chịu cần xử lý. Gọi `syncStripeDataToKV` chủ động sẽ ngăn chặn mọi trạng thái kỳ lạ mà bạn có thể gặp phải.

Đây là trang user được redirect đến sau khi hoàn tất checkout. Để đơn giản, tôi sẽ implement nó như một `get` route có redirect. Trong app của tôi, tôi làm điều này với server component và Suspense, nhưng tôi sẽ không dành thời gian giải thích tất cả ở đây.

```ts
export async function GET(req: Request) {
  const user = auth(req);
  const stripeCustomerId = await kv.get(`stripe:user:${user.id}`);
  if (!stripeCustomerId) {
    return redirect("/");
  }

  await syncStripeDataToKV(stripeCustomerId);
  return redirect("/");
}
```

Bạn có để ý tôi không dùng bất kỳ thứ gì liên quan đến `CHECKOUT_SESSION_ID` không? Vì nó tệ và nó khuyến khích bạn implement 12 cách khác nhau để lấy Stripe state. Hãy bỏ qua những tiếng gọi mời đó. Chỉ cần có MỘT hàm `syncStripeDataToKV`. Nó sẽ giúp cuộc sống của bạn dễ dàng hơn.

### `/api/stripe` (Webhook)

Đây là phần mọi người ghét nhất. Tôi sẽ chỉ dump code ra và giải thích sau.

```ts
export async function POST(req: Request) {
  const body = await req.text();
  const signature = (await headers()).get("Stripe-Signature");

  if (!signature) return NextResponse.json({}, { status: 400 });

  async function doEventProcessing() {
    if (typeof signature !== "string") {
      throw new Error("[STRIPE HOOK] Header isn't a string???");
    }

    const event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!,
    );

    waitUntil(processEvent(event));
  }

  const { error } = await tryCatch(doEventProcessing());

  if (error) {
    console.error("[STRIPE HOOK] Error processing event", error);
  }

  return NextResponse.json({ received: true });
}
```

> [!NOTE]
> Nếu bạn đang dùng Next.js Pages Router, hãy đảm bảo bật option này. Stripe yêu cầu body phải "nguyên vẹn" để có thể verify signature.
>
> ```ts
> export const config = {
>   api: {
>     bodyParser: false,
>   },
> };
> ```

### `processEvent`

Đây là hàm được gọi trong endpoint, thực sự nhận Stripe event và update KV.

```ts
async function processEvent(event: Stripe.Event) {
  // Bỏ qua nếu event không nằm trong danh sách tôi track (danh sách đầy đủ bên dưới)
  if (!allowedEvents.includes(event.type)) return;

  // Tất cả các event tôi track đều có customerId
  const { customer: customerId } = event?.data?.object as {
    customer: string; // Rất tiếc TypeScript không biết điều này
  };

  // Điều này giúp type-safe và cũng cho tôi biết nếu assumption của tôi sai
  if (typeof customerId !== "string") {
    throw new Error(
      `[STRIPE HOOK][CANCER] ID isn't string.\nEvent type: ${event.type}`,
    );
  }

  return await syncStripeDataToKV(customerId);
}
```

### Các Event Tôi Theo Dõi

Nếu có thêm event nào tôi nên track, hãy tạo PR. Nếu chúng không ảnh hưởng đến subscription state, tôi không quan tâm.

```ts
const allowedEvents: Stripe.Event.Type[] = [
  "checkout.session.completed",
  "customer.subscription.created",
  "customer.subscription.updated",
  "customer.subscription.deleted",
  "customer.subscription.paused",
  "customer.subscription.resumed",
  "customer.subscription.pending_update_applied",
  "customer.subscription.pending_update_expired",
  "customer.subscription.trial_will_end",
  "invoice.paid",
  "invoice.payment_failed",
  "invoice.payment_action_required",
  "invoice.upcoming",
  "invoice.marked_uncollectible",
  "invoice.payment_succeeded",
  "payment_intent.succeeded",
  "payment_intent.payment_failed",
  "payment_intent.canceled",
];
```

### Custom Stripe Subscription Type

```ts
export type STRIPE_SUB_CACHE =
  | {
      subscriptionId: string | null;
      status: Stripe.Subscription.Status;
      priceId: string | null;
      currentPeriodStart: number | null;
      currentPeriodEnd: number | null;
      cancelAtPeriodEnd: boolean;
      paymentMethod: {
        brand: string | null; // ví dụ: "visa", "mastercard"
        last4: string | null; // ví dụ: "4242"
      } | null;
    }
  | {
      status: "none";
    };
```

## Thêm Pro Tips

Sẽ dần thêm vào đây khi tôi nhớ ra.

### TẮT "CASH APP PAY".

Tôi tin chắc đây chỉ được dùng bởi kẻ lừa đảo. Hơn 90% cancelled transaction của tôi đến từ Cash App Pay.
![image](https://github.com/user-attachments/assets/c7271fa6-493c-4b1c-96cd-18904c2376ee)

### BẬT "Limit customers to one subscription"

Đây là một hidden setting thực sự hữu ích đã giúp tôi tránh được rất nhiều đau đầu và race condition. Thú vị là: đây là CÁCH DUY NHẤT để ngăn ai đó có thể checkout hai lần nếu họ mở hai checkout session cùng lúc 🙃 Thêm thông tin [trong Stripe docs tại đây](https://docs.stripe.com/payments/checkout/limit-subscriptions)

## Những Thứ Vẫn Là Vấn Đề Của Bạn

Dù tôi đã giải quyết được nhiều thứ ở đây, đặc biệt là subscription flow, vẫn còn một số thứ vẫn là vấn đề của bạn. Bao gồm...

- Quản lý các env var `STRIPE_SECRET_KEY` và `STRIPE_PUBLISHABLE_KEY` cho cả môi trường testing lẫn production
- Quản lý `STRIPE_PRICE_ID` cho tất cả các subscription tier cho dev và prod (Tôi không thể tin đây vẫn còn là một vấn đề)
- Expose subscription data từ KV đến user (một endpoint đơn giản chắc cũng ổn)
- Tracking "usage" (ví dụ: user nhận được 100 message mỗi tháng)
- Quản lý free trial
  ...danh sách còn dài

Dù sao, tôi hy vọng bạn tìm thấy giá trị nào đó trong tài liệu này.
