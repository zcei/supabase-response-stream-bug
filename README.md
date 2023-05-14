## Bug Description

With Supabase CLI 1.57.4, the edge-runtime version 1.2.19 was rolled out.
This includes a fix for sending back `null` values in the HTTP 204 status code, which rquired no body to be sent.

I suspect that with this change, the support for `ReadableStream`s has been broken, as they don't contain a `.size` and thus might be treated incorrectly by the fix.

## Setup Log 

<details>
  <summary>All steps taken to set up the repository</summary>
```
cd response-bug/
```

```
git init
hint: Using 'master' as the name for the initial branch. This default branch name
hint: is subject to change. To configure the initial branch name to use in all
hint: of your new repositories, which will suppress this warning, call:
hint:
hint: 	git config --global init.defaultBranch <name>
hint:
hint: Names commonly chosen instead of 'master' are 'main', 'trunk' and
hint: 'development'. The just-created branch can be renamed via this command:
hint:
hint: 	git branch -m <name>
Initialized empty Git repository in /Users/********/response-bug/.git/
```

```
response-bug on  master 
supabase init


Generate VS Code workspace settings? [y/N] y
Open the response-bug.code-workspace file in VS Code.
Finished supabase init.
```

```
response-bug on  master
supabase start
Seeding data supabase/seed.sql...me...
Started supabase local development setup.

         API URL: http://localhost:54321
     GraphQL URL: http://localhost:54321/graphql/v1
          DB URL: postgresql://postgres:postgres@localhost:54322/postgres
      Studio URL: http://localhost:54323
    Inbucket URL: http://localhost:54324
      JWT secret: super-secret-jwt-token-with-at-least-32-characters-long
        anon key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZS1kZW1vIiwicm9sZSI6ImFub24iLCJleHAiOjE5ODM4MTI5OTZ9.CRXP1A7WOeoJeXxjNni43kdQwgnWNReilDMblYTn_I0
service_role key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZS1kZW1vIiwicm9sZSI6InNlcnZpY2Vfcm9sZSIsImV4cCI6MTk4MzgxMjk5Nn0.EGIM96RAZx35lJzdJsyH-qQwv8Hdp7fsn3W0YpN81IU
```

```
response-bug on  master
supabase functions new test
Created new Function at supabase/functions/test
```

Set up VSCode settings in `.vscode/settings.json`

```
{
  "deno.enable": true,
  "deno.unstable": true,
  "deno.importMap": "./supabase/functions/import_map.json"
}
```


Adjust `supabase/functions/test/index.ts` with OPTIONS & `Response` stream example

```
response-bug on  master
supabase functions serve
Setting up Edge Functions runtime...
Serving functions on http://localhost:54321/functions/v1/<function-name>
```
</details>

## Example Requests

OPTIONS request should:

* require no auth header
* can send back CORS headers
* sends back a 204 without any further content (`null` Response body)


```
curl -v -X OPTIONS http://localhost:8000/functions/v1/test
```

For a given GET/POST request handler should:

* ✅ require an auth header (repo uses the default one, this is no secrets exposure)
* ✅ be able to not consume the request body
* Craft their own `ReadableStream` (or obtain it from some helpers like Deno's `std`)
* Return the `ReadableStream<Uint8Array>` as the Response body, as allowed by the Fetch spec

```
curl -vv \
-H "authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZS1kZW1vIiwicm9sZSI6ImFub24iLCJleHAiOjE5ODM4MTI5OTZ9.CRXP1A7WOeoJeXxjNni43kdQwgnWNReilDMblYTn_I0" \
http://localhost:54321/functions/v1/test
```

Notice that this call with stop after the request has been sent and will never receive a response byte.
