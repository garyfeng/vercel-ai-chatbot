 # Plan: Integrate Dify Chat API into Frontend-Only Chat UI

 This document outlines the major steps, considerations, and detailed instructions (with code snippets) to adapt the existing Next.js chat UI into a pure frontend application that uses the Dify chatbot API as its backend.

 ## Major Steps & Considerations
 1. Create a feature branch and clean up unnecessary server code  
 2. Define environment variables for Dify API and optional auth  
 3. Update the HTTP layer (`fetcher`) to target Dify endpoints  
 4. Replace all internal `/api/...` calls in the UI with Dify-based URLs  
 5. Remove or archive Next.js API routes and server utilities  
 6. Implement or integrate backend endpoints (chat completion, file upload, voting, document versioning)  
 7. (Optional) Integrate an authentication strategy (NextAuth, JWT, or guest tokens)  
 8. Test end-to-end: manual and automated tests (Playwright/SWR caching)  
 9. Clean dependencies and deploy  

 ---

 ## Step-by-Step Instructions

 ### 1. Create & Checkout Feature Branch
 ```bash
 git checkout -b feature/dify-chatui-integration
 ```

 ### 2. Define Environment Variables
 Create a `.env.local` at project root:
 ```env
 NEXT_PUBLIC_CHAT_API_BASE_URL=https://open.dify.ai/api/v1
 NEXT_PUBLIC_DIFY_API_KEY=<your-dify-api-key>
 ```
 If you need user sessions:
 ```env
 AUTH_SECRET=<your-jwt-secret>
 ```

 ### 3. Update the HTTP Layer (`fetcher`)
 In `lib/utils.ts`, replace the old `fetcher` with:
 ```ts
 const API_BASE = process.env.NEXT_PUBLIC_CHAT_API_BASE_URL!;
 const API_KEY  = process.env.NEXT_PUBLIC_DIFY_API_KEY!;

 export const fetcher = async (path: string) => {
   const url = path.startsWith('http')
     ? path
     : `${API_BASE}${path}`;
   const res = await fetch(url, {
     headers: {
       'Content-Type': 'application/json',
       'Authorization': `Bearer ${API_KEY}`
     },
   });
   if (!res.ok) throw new Error(await res.text());
   return res.json();
 };
 ```

 ### 4. Swap Out UI Fetch Calls
 - **Upvote/Downvote** (`components/message-actions.tsx`):
   ```diff
  - fetch('/api/vote', { method: 'PATCH', body: ... })
  + fetch('/chat/vote', { method: 'PATCH', body: ... })
   ```
 - **File Upload** (`components/multimodal-input.tsx`):
   ```diff
  - await fetch('/api/files/upload', { method: 'POST', body: formData })
  + await fetch('/files/upload', { method: 'POST', body: formData })
   ```
 - **Chat History** (`components/chat.tsx`):
   ```diff
  - useSWR(messages.length>=2 ? `/api/vote?chatId=${id}` : null, fetcher);
  + useSWR(messages.length>=2 ? `/chat/vote?chatId=${id}` : null, fetcher);
   ```
 - **Document Endpoints** (`components/artifact.tsx` & `components/version-footer.tsx`):
   ```diff
  - useSWR(`/api/document?id=${documentId}`, fetcher)
  + useSWR(`/document?id=${documentId}`, fetcher)
   ```

 ### 5. Remove Next.js API Routes & Server Code
 ```bash
 rm -rf app/(auth) app/(chat)/api lib/db lib/ai drizzle.config.ts middleware.ts
 ```
 Remove related scripts and dependencies from `package.json`.

 ### 6. Backend: Implement or Integrate Dify Endpoints
 1. **Chat Completion** (streaming):
    ```http
    POST /chat/completions
    Headers: { Authorization: Bearer <API_KEY>, Content-Type: application/json }
    Body: {
      model: "<model-name>",
      messages: [{ role: "user", content: "Hello" }, ...],
      stream: true
    }
    ```
    - Stream back JSON deltas:  
      `{ type: 'text-delta'|'finish'|… , content: string }`
 2. **Vote**:
    - `GET  /chat/vote?chatId=<id>` → `[ { chatId, messageId, isUpvoted } ]`
    - `PATCH /chat/vote` (body `{ chatId, messageId, type: 'up'|'down' }`)
 3. **File Upload**:
    - `POST /files/upload` (multipart/form-data)   
      → `{ url, pathname, contentType }`
 4. **Document Versioning**:
    - `GET    /document?id=<docId>` → `Document[]`
    - `POST   /document?id=<docId>` (body `{ title, content, kind }`)   
      appends a new version  
    - `DELETE /document?id=<docId>&timestamp=<ts>`  
      removes versions ≥ timestamp

 ### 7. (Optional) Integrate Authentication
 - Use **NextAuth** or roll your own JWT endpoints under `/api/auth/*`.  
 - Ensure client-side `SessionProvider` picks up tokens.  
 - Protect UI routes or fallback to "guest" token via `/api/auth/guest`.

 ### 8. Testing & Validation
 - Run locally: `pnpm install && pnpm dev`  
 - Verify:
   - Chat input sends to Dify and streams responses.  
   - File uploads return valid URLs.  
   - Voting changes state in real time.  
   - Artifact previews, version diffs, and restores work end-to-end.  
 - Automated tests: update Playwright tests to hit your real endpoints or mock them.

 ### 9. Clean Up & Deploy
 - Remove unused dependencies (`next-auth`, `drizzle-orm`, `@ai-sdk/*`, etc.)  
 - Run lint/format: `pnpm lint:fix && pnpm format`  
 - Commit and push your branch, then deploy to Vercel or your platform of choice.

 --  
 _End of Plan_