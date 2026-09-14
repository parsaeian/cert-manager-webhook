# نصب وبهوک در کلاستر

اگر repository قبلاً اضافه نشده، ابتدا آن را اضافه کنید:

```bash
helm repo add sotoon https://admin.registry.platform.ske.sotoon.ir/repository/helm-hosted/

helm repo update
```

### نکته

آدرس `https://admin.registry.platform.ske.sotoon.ir/` فقط از داخل ماشین‌های کلاستر قابل دسترسی است.
اگر از بیرون کلاستر هستید، برای دسترسی باید از bastion استفاده کنید.

برای نصب helm chart مربوط به webhook از دستور زیر استفاده کنید:

```bash
helm install cert-manager-webhook https://admin.registry.platform.ske.sotoon.ir/repository/helm-hosted/cert-manager-webhook \
  --namespace cert-manager \
  --version 1.3.20

```

---

## ساخت ClusterIssuer یا Issuer

بعد از نصب، باید یک `ClusterIssuer` یا `Issuer` با مشخصات زیر ایجاد کنید:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: <name>
spec:
  acme:
    email: <their-own-email>
    preferredChain: ""
    privateKeySecretRef:
      name: <name of secret>
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - dns01:
          webhook:
            groupName: acme.sotoon.ir
            solverName: sotoon
            config:
              apiTokenSecretRef:
                key: <key>
                name: <apiTokenSecretRef>
              endpoint: https://api.sotoon.ir/delivery/v1
              inCluster: false
              namespace: <namespace in cdn>

```

---

## توضیح مقادیر

- **metadata.name**نام issuer است و می‌توانید هر نام دلخواهی انتخاب کنید.
- **spec.acme.email**ایمیل شما برای ثبت در Let's Encrypt است. برای دریافت نوتیفیکیشن‌های مربوط به certificate استفاده می‌شود.
- **spec.acme.privateKeySecretRef.name**
  نام سکرتی که کلید خصوصی ACME داخل آن ذخیره می‌شود. اگر وجود نداشته باشد، توسط cert-manager ساخته می‌شود.

---

## تنظیمات دسترسی به DNS (Webhook)

برای اینکه webhook بتواند به تنظیمات دامنه شما دسترسی داشته باشد:

1. وارد [بخش توکن‌ها در پروفایل کاربری](https://ocean.sotoon.ir/profile/tokens) خود در پنل اوشن شوید.
2. توکن را در یک Kubernetes Secret ذخیره کنید و مقادیر زیر را در issuer قرار دهید:

```yaml
apiTokenSecretRef:
  key: <<key>>
  name: <<apiTokenSecretRef>>

```

- **key**: کلیدی که داخل secret توکن را با آن ذخیره کرده‌اید (مثلاً `token`)
- **name**: نام secret ساخته‌شده در Kubernetes

---

## تنظیم namespace

برای مقدار `config.namespace`:

1. از [بخش فضاهای کاری در پروفایل کاربری](https://ocean.sotoon.ir/profile/workspaces).
2. نام  فضای کاری که قصد دارید روی آن کار کنید را کپی کرده و در این فیلد قرار دهید:

```yaml
config:
  namespace: <<namespace in cdn>>

```

---

با توجه به اختلالاتی که روی شبکه‌ی کشور مشاهده میشه امکان داره که nameserverهایی که ست شدن به درستی پاسخگو نباشن که در این صورت کافیست nameserver مناسبی رو پیدا کرد و اون رو به لیست nameserver ها در helm-chart اضافه کرد.

---

## تنظیمات محیط‌های بدون دسترسی به اینترنت (Air-Gapped)

اگر کلاستر شما دسترسی به اینترنت ندارد و تنها یک یا چند Resolver DNS خاص را می‌تواند استفاده کند، می‌توانید از گزینه `usePublicResolver` استفاده کنید.

### نحوه فعال‌کردن

1. در تنظیمات `ClusterIssuer` یا `Issuer`، فیلد `usePublicResolver` را به `true` تغییر دهید:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: <name>
spec:
  acme:
    email: <their-own-email>
    preferredChain: ""
    privateKeySecretRef:
      name: <name of secret>
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - dns01:
          webhook:
            groupName: acme.sotoon.ir
            solverName: sotoon
            config:
              apiTokenSecretRef:
                key: <key>
                name: <apiTokenSecretRef>
              endpoint: https://api.sotoon.ir/delivery/v1
              inCluster: false
              namespace: <namespace in cdn>
              usePublicResolver: true  # فعال‌کردن

```

2. سپس Helm values را برای تنظیم nameServers دستی‌کاری کنید:

**برای یک Resolver:**

```yaml
nameServers:
  - 10.0.0.5
```

**برای چند Resolver:**

```yaml
nameServers:
  - 10.0.0.5
  - 10.0.0.6
  - 10.0.0.7
```

**نمونه Helm install:**

```bash
helm install cert-manager-webhook \
  https://admin.registry.platform.ske.sotoon.ir/repository/helm-hosted/cert-manager-webhook \
  --namespace cert-manager \
  --version 1.3.20 \
  --set "nameServers[0]=10.0.0.5" \
  --set "nameServers[1]=10.0.0.6"
```

یا از فایل values.yaml:

```bash
helm install cert-manager-webhook \
  https://admin.registry.platform.ske.sotoon.ir/repository/helm-hosted/cert-manager-webhook \
  --namespace cert-manager \
  --version 1.3.20 \
  -f values.yaml
```

---

## پسوندهای DNS سرورهای پلتفرم ستون (Authoritative DNS Suffixes)

لیست پسوندهای DNS سرورهای پلتفرم ستون به‌صورت پیش‌فرض داخل ایمیج قرار دارد. برای تغییر آن بدون ساخت ایمیج جدید:

```yaml
authoritativeDnsSuffixes:
  - newns.sotoon.ir
```

مقدار ست‌شده **جایگزین** لیست پیش‌فرض می‌شود؛ ورودی نامعتبر باعث خطا در هنگام راه‌اندازی وبهوک می‌شود.
