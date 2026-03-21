# IDOR — Insecure Direct Object Reference
### Web Exploitation Write-Up | Week 3
**Author:** OBADA  
**Platform:** TryHackMe  
**Date:** 2026-03-19

---

## What is IDOR?

Insecure Direct Object Reference (IDOR) is a vulnerability where an application exposes internal object identifiers — such as user IDs, order numbers, or filenames — without verifying whether the requester is authorized to access them. By manipulating a parameter in a URL or API call, an attacker can view, modify, or delete data they should never have access to.

IDOR falls under the broader category of Broken Access Control, which OWASP has consistently ranked among the top web security risks for years. What makes IDOR particularly dangerous is its simplicity: no advanced exploitation techniques, no payload encoding, no privilege escalation chains required. A single modified parameter — often just a number — can expose private information belonging to another user.

---

## Where IDOR Appears

IDOR can surface in any system that references internal objects through user-controlled input. Common contexts include:

| Application Type | Object Example | Why IDOR Occurs |
|---|---|---|
| Account systems | User ID | Direct exposure of user identifiers |
| E-commerce apps | Order ID | Improper ownership validation |
| File management | File ID | Missing access control on files |
| Ticketing platforms | Ticket ID | Users accessing other users' tickets |
| Multi-tenant SaaS | Tenant ID | Data leakage across tenants |

Attackers typically enumerate predictable IDs, attempt sequential identifiers, or replay authenticated requests with modified parameters.

---

## How Attackers Exploit IDOR — Safe & Controlled Examples

The following examples illustrate common insecure patterns alongside their secure counterparts. These are not harmful exploits — they are learning models to help recognize unsafe code.

**Example 1 — Node.js / Express**

```javascript
// ❌ Unsafe: trusting the user-supplied ID
app.get('/api/user/profile', (req, res) => {
  const userId = req.query.id;
  db.users.findById(userId).then(user => res.json(user));
});

// ✅ Safe: enforce authorization from session
app.get('/api/user/profile', (req, res) => {
  const authenticatedId = req.user.id;
  db.users.findById(authenticatedId).then(user => {
    if (!user) return res.status(404).json({ error: "User not found" });
    res.json({ id: user.id, email: user.email, role: user.role });
  });
});
```

**Example 2 — Python / Flask**

```python
# ❌ Unsafe: no ownership verification
@app.get("/invoice")
def get_invoice():
    invoice_id = request.args.get("id")
    return get_invoice_by_id(invoice_id)

# ✅ Safe: permission check added
@app.get("/invoice")
def get_invoice_secure():
    invoice_id = request.args.get("id")
    user_id = session["user_id"]
    invoice = get_invoice_by_id(invoice_id)
    if invoice.owner_id != user_id:
        return {"error": "Unauthorized"}, 403
    return invoice
```

**Example 3 — Java (Spring Boot) + React**

```java
// ❌ Unsafe: missing authorization check
@GetMapping("/orders")
public Order getOrder(@RequestParam String orderId) {
    return orderRepository.findById(orderId);
}

// ✅ Safe: enforce ownership
@GetMapping("/orders")
public ResponseEntity getOrder(@RequestParam String orderId, Principal principal) {
    Order order = orderRepository.findById(orderId);
    if (!order.getUser().equals(principal.getName())) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
               .body(Collections.singletonMap("error", "Access denied"));
    }
    return ResponseEntity.ok(order);
}
```

```javascript
// React — safe fetch
async function fetchOrder(orderId) {
  const res = await fetch(`/orders?orderId=${orderId}`, {
    headers: { "Authorization": `Bearer ${localStorage.getItem("token")}` }
  });
  if (!res.ok) throw new Error("Unauthorized or not found");
  return res.json();
}
```

---

## Types of IDs Encountered

Not all IDOR vulnerabilities look the same. During this lab, I encountered four distinct identifier types, each requiring a different approach:

- **Numeric IDs** — The most obvious form. Sequential integers like `?id=42` are trivially enumerable. Increment by one, and you're accessing someone else's data.
- **Encoded IDs** — Values like `?id=NDI=` appear obfuscated but are simply Base64-encoded. Decoding reveals the underlying integer, which can then be modified and re-encoded.
- **Hashed IDs** — MD5 or SHA1 hashes of predictable values (e.g., a username or numeric ID). If the input space is small, they can be cracked or pre-computed.
- **UUIDs** — Universally unique identifiers look random and unpredictable, but they are not a security control. If the application doesn't enforce ownership server-side, knowing a UUID is enough to access the object.

> **The core lesson: obfuscation is not authorization.**

---

## The Finding — A Flag Hidden in Response Headers

During the IDOR lab, I discovered something that most people would miss: the flag wasn't in the response body — it was buried in the response headers.

This wasn't luck. It was the result of reading the full HTTP response rather than just the rendered output. When modifying the user ID parameter and sending the request through Burp Suite Repeater, I made a habit of scrolling through every part of the response — status line, headers, and body. The flag appeared as a custom header value that would have been completely invisible in a browser.

This matters beyond CTFs. In real applications, sensitive data leaks through headers more often than developers realize — internal stack traces, server versions, session tokens, or improperly placed debug values. The habit of reading the full response, not just what the application displays, is what separates thorough testing from surface-level clicking.

---

## Tooling — Burp Suite Repeater

The primary tool used throughout this lab was **Burp Suite Repeater**. The workflow was consistent across every test case:

1. Intercept the target request through the Proxy
2. Send it to Repeater (`Ctrl+R`)
3. Modify the object identifier in the request — URL parameter, request body, or header
4. Send and analyze the full response — status code, headers, and body
5. Compare responses across different ID values to confirm unauthorized data access

Repeater's value here isn't speed — it's control and visibility. Every request is deliberate, every response is fully readable, and nothing is automated away.

---

## Key Takeaway

IDOR is deceptively simple to exploit and deceptively easy to miss as a developer. The fix is never on the frontend — hiding or encoding an ID in the UI does nothing if the backend doesn't enforce ownership. Every object access must be validated server-side against the authenticated user's identity, every time.

And from a tester's perspective: always read the full response. The most interesting findings are rarely where you expect them.

---

*Written by XENOS — obadahamed.github.io*
