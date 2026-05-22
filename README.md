# scoopet.codes

A public-safe project overview for a privacy-first Epic Games code checker that keeps end-user submissions private in transit and keeps operational details out of the public repository.

This repository is intentionally minimal. It describes the product shape, privacy model, and encrypted request/response flow without publishing upstream implementation details, deployment secrets, private integration points, or infrastructure-specific behavior.

## What It Does

Users paste one or more Epic Games redemption codes into a simple web interface. The browser prepares the request, encrypts the submitted codes, and sends only ciphertext to the checker backend. The backend decrypts the payload in memory, performs the Epic Games validity check through a private integration layer, encrypts the result, and returns the encrypted response to the browser.

The browser decrypts the response locally and shows a small result table:

- `valid`
- `invalid`
- `used`
- `expired`
- `error`

No Epic account login is required for the user-facing flow.

## Privacy Goals

- Codes are encrypted before leaving the browser.
- Decrypted codes are handled only in backend memory for the duration of a check.
- Results are encrypted before being sent back to the browser.
- The public repository does not expose private checker mechanics.
- The frontend never needs access to backend credentials or upstream operational details.

## High-Level Flow

```text
Browser
  -> normalize pasted codes
  -> create per-request encryption material
  -> encrypt request payload
  -> POST encrypted payload

Backend
  -> decrypt payload in memory
  -> check codes with Epic Games through private adapter
  -> encrypt result payload
  -> return encrypted result

Browser
  -> decrypt response
  -> render results table
```

## Frontend Shape

The frontend can be implemented as a small React or Next.js page:

- Textarea for pasted codes
- Client-side normalization and dedupe
- `Check Codes` button
- Encrypted network request
- Results table rendered only after local decryption

Example normalization:

```ts
export function parseCodes(input: string): string[] {
  const seen = new Set<string>();

  return input
    .split(/[\s,]+/)
    .map((value) => value.trim().toUpperCase())
    .filter(Boolean)
    .filter((value) => {
      const compact = value.replaceAll("-", "");
      if (seen.has(compact)) return false;
      seen.add(compact);
      return true;
    });
}
```

Example encrypted submit:

```ts
type EncryptedPayload = {
  keyId: string;
  iv: string;
  ciphertext: string;
};

export async function submitEncryptedCodes(codes: string[]): Promise<EncryptedPayload> {
  const payload = await encryptForChecker({ codes });

  const response = await fetch(CHECKER_REQUEST_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(payload),
  });

  if (!response.ok) {
    throw new Error("Check failed");
  }

  return response.json() as Promise<EncryptedPayload>;
}
```

## Encryption Model

Use authenticated encryption for request and response bodies. The exact key-management strategy depends on the deployment, but a safe public pattern is:

- Browser encrypts with a short-lived public session key.
- Backend decrypts with the matching private session material.
- Backend encrypts the response for the browser.
- Keys are rotated regularly and never committed to the repository.

Example browser-side AES-GCM helper:

```ts
function toBase64(bytes: ArrayBuffer): string {
  return btoa(String.fromCharCode(...new Uint8Array(bytes)));
}

export async function encryptJson(
  key: CryptoKey,
  value: unknown,
): Promise<{ iv: string; ciphertext: string }> {
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const encoded = new TextEncoder().encode(JSON.stringify(value));

  const ciphertext = await crypto.subtle.encrypt(
    { name: "AES-GCM", iv },
    key,
    encoded,
  );

  return {
    iv: toBase64(iv),
    ciphertext: toBase64(ciphertext),
  };
}
```

Example decrypted response handling:

```ts
type CodeResult = {
  code: string;
  status: "valid" | "invalid" | "used" | "expired" | "error";
  itemName?: string;
};

export async function readEncryptedResults(
  key: CryptoKey,
  encrypted: EncryptedPayload,
): Promise<CodeResult[]> {
  const decrypted = await decryptFromChecker(key, encrypted);
  return decrypted.results;
}
```

## Backend Shape

The backend handler can stay very small:

```ts
async function handleEncryptedCheck(request: EncryptedPayload): Promise<EncryptedPayload> {
  const encryptedRequest = request;

  const { codes, replyKey } = await decryptIncomingPayload(encryptedRequest);

  const results = await checkCodesWithEpicGames(codes);

  return encryptOutgoingPayload(replyKey, {
    results,
    checkedAt: new Date().toISOString(),
  });
}
```

The Epic Games checker is represented as a private adapter:

```ts
type EpicCodeStatus = "valid" | "invalid" | "used" | "expired" | "error";

type EpicCodeResult = {
  code: string;
  status: EpicCodeStatus;
  itemName?: string;
};

async function checkCodesWithEpicGames(codes: string[]): Promise<EpicCodeResult[]> {
  // Private production integration lives outside this public overview.
  // Keep upstream request details, operational safeguards, and credentials out of source control.
  return codes.map((code) => ({
    code,
    status: "error",
  }));
}
```

## Security Notes

- Validate request size before decrypting or checking codes.
- Rate-limit requests by IP and session.
- Keep decrypted values out of logs.
- Prefer short retention windows for transient check jobs.
- Return generic user-facing errors when the backend cannot complete a check.

This is an overview repository, not a production deployment package.
