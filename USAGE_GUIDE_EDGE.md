### Usage Guide: Lightweight App Store Server API Client for Edge Environments

This guide explains how to use the `app-store-api-edge.ts` utility to call the Apple App Store Server API from WinterCG-compliant edge environments like Cloudflare Workers, Vercel Edge Functions, or Convex.

This lightweight client is designed to perform the `getTransactionInfo` call without relying on Node.js-specific APIs, making it compatible with modern serverless platforms.

#### 1. Installation

This utility requires a JWT library that is compatible with the Web Crypto API. We will use `@tsndr/cloudflare-worker-jwt`.

Open your terminal in your project's root directory and run:

```bash
npm install @tsndr/cloudflare-worker-jwt
```

You will also need to have the `app-store-api-edge.ts` file in your project, which I provided earlier.

#### 2. Setup and Configuration

To communicate with the App Store Server API, you need four pieces of information from App Store Connect.

1.  **Issuer ID**: Found on the "Keys" tab under "Users and Access".
2.  **Key ID**: Generated when you create a new API key.
3.  **Private Key**: The `.p8` file you download when creating the key.
4.  **Bundle ID**: The bundle identifier for your application (e.g., `com.yourcompany.yourapp`).

**Important**: Never hardcode your private key or other secrets in your source code. Store them as environment variables or secrets in your edge provider's dashboard.

**Action**:
1.  Open your `.p8` private key file and copy its entire contents.
2.  Go to your deployment platform's settings (e.g., Cloudflare, Vercel, Convex) and create the following secrets:

    *   `APP_STORE_ISSUER_ID`: Your Issuer ID.
    *   `APP_STORE_KEY_ID`: Your Key ID.
    *   `APP_STORE_BUNDLE_ID`: Your app's bundle ID.
    *   `APP_STORE_SIGNING_KEY`: The full content of your `.p8` private key, including the `-----BEGIN PRIVATE KEY-----` and `-----END PRIVATE KEY-----` lines.

#### 3. How to Use

The following example demonstrates how to import, instantiate, and use the client within an edge function. This example assumes a generic edge function that receives a `transactionId`.

```typescript
// Example for a generic edge environment (e.g., Cloudflare Worker, Vercel Edge Function)

import { AppStoreServerAPIClientEdge, Environment, APIException } from './app-store-api-edge';

// The entry point for your edge function might look like this.
// `env` is typically where your secrets are stored.
export default async function handleRequest(request, env) {

  // 1. Get transactionId from the request (e.g., from query param or body)
  const { searchParams } = new URL(request.url);
  const transactionId = searchParams.get('transactionId');

  if (!transactionId) {
    return new Response(JSON.stringify({ error: 'transactionId is required' }), {
      status: 400,
      headers: { 'Content-Type': 'application/json' },
    });
  }

  // 2. Initialize the App Store API Client
  // It reads your credentials securely from the environment.
  const apiClient = new AppStoreServerAPIClientEdge(
    env.APP_STORE_SIGNING_KEY,
    env.APP_STORE_KEY_ID,
    env.APP_STORE_ISSUER_ID,
    env.APP_STORE_BUNDLE_ID,
    Environment.SANDBOX // Use Environment.PRODUCTION for production
  );

  // 3. Call the API and handle the response
  try {
    console.log(`Fetching info for transactionId: ${transactionId}`);
    
    const response = await apiClient.getTransactionInfo(transactionId);
    
    console.log('Successfully received signed transaction info.');
    
    // The raw JWS token is in response.signedTransactionInfo
    // You will need to verify and decode this token to get the transaction details.
    return new Response(JSON.stringify(response), {
      status: 200,
      headers: { 'Content-Type': 'application/json' },
    });

  } catch (error) {
    if (error instanceof APIException) {
      // Handle Apple-specific API errors
      console.error(`API Error: ${error.apiError} - ${error.errorMessage} (HTTP ${error.httpStatusCode})`);
      return new Response(JSON.stringify({
        apiError: error.apiError,
        errorMessage: error.errorMessage,
      }), { status: error.httpStatusCode });
    } else {
      // Handle other errors (e.g., network issues)
      console.error('An unexpected error occurred:', error);
      return new Response(JSON.stringify({ error: 'An internal server error occurred.' }), { status: 500 });
    }
  }
}
```

#### 4. What to Expect

**On a successful call**, the `getTransactionInfo` method will return a `TransactionInfoResponse` object.

*   **`TransactionInfoResponse` object**:
    ```json
    {
      "signedTransactionInfo": "eyJhbGciOiJFUzI1NiIsIng1YyI6WyJNSUl...a very long string...In0.eyJ0cmFuc2FjdGlv..."
    }
    ```
    The `signedTransactionInfo` field contains a **JSON Web Signature (JWS)**. This is not the transaction data itself, but a signed, tamper-proof container for it. You must decode and verify this token using Apple's public key to access the transaction details. Libraries like `jose` can be used for this verification step.

**If an error occurs**, the method will throw an `APIException`.

*   **`APIException` object**: This custom error object contains details about the failure.
    *   `httpStatusCode`: The HTTP status code from the API response (e.g., `404`, `400`).
    *   `apiError`: The specific Apple error code (e.g., `4040010` for `TransactionIdNotFoundError`).
    *   `errorMessage`: A human-readable error message from the API.

By catching this exception, you can build robust error handling into your application.