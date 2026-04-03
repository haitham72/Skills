Here is your minimal guide for the complete workflow.

_Note: The `curl` commands below use the external Windows/WSL host (`127.0.0.1:54321`). If executing these from an n8n node, swap `127.0.0.1:54321` with `supabase_kong_User:8000`._

### 1. Minimal `curl` Commands (Storage Operations)

**Upload a File**
Requires a `POST` request with the file passed as binary data. Include the correct `Content-Type` for your file.

```bash
curl -X POST "http://127.0.0.1:54321/storage/v1/object/private_bucket/new.png" \
     -H "apikey: YOUR_SUPABASE_SECRET_KEY" \
     -H "Authorization: Bearer YOUR_SUPABASE_SECRET_KEY" \
     -H "Content-Type: image/png" \
     --data-binary "@path_to_your_local_file.png"
```

**Download a File**
Requires a `GET` request. Use the `-o` flag to save the output locally.

```bash
curl -X GET "http://127.0.0.1:54321/storage/v1/object/private_bucket/new.png" \
     -H "apikey: YOUR_SUPABASE_SECRET_KEY" \
     -H "Authorization: Bearer YOUR_SUPABASE_SECRET_KEY" \
     -o saved_image.png
```

**List Files**
Requires a `POST` request to the `/list` endpoint. The `prefix` property is mandatory in the JSON body.

```bash
curl -X POST "http://127.0.0.1:54321/storage/v1/object/list/private_bucket" \
     -H "apikey: YOUR_SUPABASE_SECRET_KEY" \
     -H "Authorization: Bearer YOUR_SUPABASE_SECRET_KEY" \
     -H "Content-Type: application/json" \
     -d '{"prefix": ""}'
```

**Delete a File**
Requires a `DELETE` request to the specific file path.

```bash
curl -X DELETE "http://127.0.0.1:54321/storage/v1/object/private_bucket/new.png" \
     -H "apikey: YOUR_SUPABASE_SECRET_KEY" \
     -H "Authorization: Bearer YOUR_SUPABASE_SECRET_KEY"
```

---

### 2. How to get the Supabase `.env` Variables

If you need to retrieve your keys, URLs, and database strings formatted as environment variables, run this inside the directory where your Supabase project is initialized:

```bash
npx supabase status -o env
```

This will output a clean list of variables (like `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `POSTGRES_PASSWORD`, etc.) that you can copy directly into your N8N `.env` file or global variables.

---

### 3. How to Link N8N and Supabase in Docker

Since both N8N and Supabase are running locally in Docker, they need to be on the same Docker network to communicate using container names (like `supabase_kong_User`).

**Step 1: Find the Supabase network name**
Run this command and look for a network related to your Supabase project (usually ends in `_network` or `_default`):

```bash
docker network ls
```

**Step 2: Connect the N8N container to the Supabase network**
Assuming your N8N container is named `N8N` and the Supabase network is `supabase_network_User` (replace with your actual network name), run:

```bash
docker network connect supabase_network_User N8N
```

**Step 3: Verify the connection**
Check that N8N is now inside that network:

```bash
docker network inspect supabase_network_User
```

You should see both the `supabase_kong_User` container and the `N8N` container listed under the "Containers" section. Once linked, N8N can seamlessly resolve `http://supabase_kong_User:8000`.

---

### 4. Setting Up Edge Functions (Hybrid Logic)

Edge Functions allow you to run custom TypeScript logic. To make them visible in your local dashboard and reachable via API, follow these steps:

**Step 1: Initialize the Functions Directory**
If you haven't already, create the base folder structure within your Supabase project:

```bash
# This creates the /supabase/functions directory
mkdir -p supabase/functions
```

**Step 2: Create a New Function**
Generate a new function template. This creates a folder and an `index.ts` file:

```bash
npx supabase functions new download-image
```

**Step 3: Start the Edge Runtime Server**
For your functions to be "Active" and visible in the Local Dashboard (`http://localhost:54323`), you must start the serve command. This watches your files for changes and reloads automatically:

```bash
npx supabase functions serve --no-verify-jwt
```

- `--no-verify-jwt`: Optional flag for local testing if you want to bypass strict Auth headers temporarily.

**Step 4: Verify Visibility**

1.  Open your **Local Supabase Dashboard**.
2.  Navigate to the **Edge Functions** tab in the sidebar.
3.  You should now see your function name listed as **"Ready"** or **"Active"**.

**Step 5: Invoking from n8n**
When calling these functions from an n8n node, remember to use the internal Docker network address:

- **Method:** `POST`
- **URL:** `http://supabase_kong_User:8000/functions/v1/download-image`
- **Headers:** Include `apikey` and `Authorization` using your Service Role Key.

---

### 5. Troubleshooting Local Edge Functions

- **Read-Only Dashboard:** The local dashboard is a **viewer only**. You cannot edit code in the browser; you must edit the `.ts` files in your `supabase/functions/` directory using a text editor (like VS Code or Nano).
- **Environment Variables:** Local functions automatically have access to `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY`. You do not need to manually add these to a `.env` for the function to "see" its own project.
- **Binary Data:** If downloading images/files via a function, ensure your code returns a `Response` with the correct `Content-Type` header (e.g., `image/png`) to avoid "empty" or corrupt files.
