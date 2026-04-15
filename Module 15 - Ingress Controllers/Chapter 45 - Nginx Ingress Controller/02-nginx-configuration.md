# NGINX Configuration

## Why Configuration Matters

Once NGINX is installed, you will want to tune how it behaves.

Maybe your app needs longer timeouts.

Maybe uploads are larger than the default limit.

Maybe you want the controller to redirect HTTP to HTTPS.

That is where configuration comes in.

---

## The ConfigMap Pattern

NGINX Ingress Controller usually reads settings from a ConfigMap.

That keeps the behavior in Kubernetes YAML instead of inside the app.

### Example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
	name: nginx-configuration
	namespace: ingress-nginx
data:
	proxy-read-timeout: "60"
	proxy-send-timeout: "60"
	client-max-body-size: "20m"
```

---

## What These Settings Mean

| Setting | Meaning | Why It Helps |
|--------|---------|--------------|
| proxy-read-timeout | How long NGINX waits for backend response | Prevents premature failures |
| proxy-send-timeout | How long NGINX waits while sending data | Helps with slow clients |
| client-max-body-size | Maximum upload size | Useful for file uploads |

### Story Example

Your app lets users upload product images.

The default body size is too small.

You raise the limit in the ConfigMap.

Now the app works without changing code.

---

## Ingress Annotations vs ConfigMap

| Feature | Annotation | ConfigMap |
|---------|------------|-----------|
| Scope | One Ingress | Whole controller |
| Best for | Special rules on one app | Shared defaults |
| Example | Rewrite target | Timeout values |

Use annotations when only one app needs the setting.

Use ConfigMap when every Ingress should follow the same rule.

---

## Example: Timeouts for Slow Apps

Some apps are slow for a reason.

Maybe they talk to a database.

Maybe they generate reports.

Maybe they wait on third-party APIs.

If NGINX times out too early, users see errors even though the app is still working.

### Config Example

```yaml
data:
	proxy-read-timeout: "120"
	proxy-send-timeout: "120"
```

That gives the app more breathing room.

---

## Example: Bigger Uploads

```yaml
data:
	client-max-body-size: "50m"
```

That can help when your app accepts:

- Images
- CSV files
- Documents

Without that setting, uploads may fail for no useful reason.

---

## How Teams Use This in Practice

Platform teams usually set safe defaults.

App teams then add only the special changes they need.

That keeps the cluster consistent and avoids random tuning per app.

---

## Troubleshooting NGINX Configuration

### Problem 1: Setting Not Applied

Check whether the ConfigMap name matches what the controller expects.

### Problem 2: App Still Times Out

Check backend latency and service health too.

### Problem 3: Upload Still Fails

Make sure the Ingress annotation and controller config do not conflict.

---

## AWS EKS Angle

In EKS, NGINX often sits in front of services that need a shared ingress layer.

The ConfigMap becomes the place where platform defaults live.

That is a clean way to keep the behavior repeatable across teams.

---

## Summary

NGINX configuration lets you tune the controller without changing application code. ConfigMaps are the usual way to set shared behavior like timeouts and upload limits, while annotations handle one-off Ingress rules.

Next, Chapter 45 moves to TLS with NGINX Ingress, where the controller starts serving HTTPS instead of plain HTTP.
