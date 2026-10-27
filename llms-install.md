# Installing the Crevio MCP server

Crevio is a hosted remote MCP server. There is nothing to clone, build or run.

1. The user needs a Crevio account. A free one works: https://crevio.co/users/signup
2. Add the server. If the client supports remote Streamable HTTP servers, use the URL directly:

   ```json
   {
     "mcpServers": {
       "crevio": {
         "url": "https://mcp.crevio.co/mcp"
       }
     }
   }
   ```

   If the client only runs local stdio servers, bridge it with `mcp-remote`:

   ```json
   {
     "mcpServers": {
       "crevio": {
         "command": "npx",
         "args": ["-y", "mcp-remote", "https://mcp.crevio.co/mcp"]
       }
     }
   }
   ```

3. On first use the client opens a browser to sign in to Crevio with OAuth. No API key is needed.
   Business-plan users can instead pass an API token as an `Authorization: Bearer <token>` header.
4. Verify by calling the `whoami` tool. It returns the connected account.

Docs: https://crevio.co/docs/developer/mcp
