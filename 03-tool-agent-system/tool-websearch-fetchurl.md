# WebSearch & FetchUrl Tools: Tool System Architecture Notes

## Overview

The WebSearch and FetchUrl tools form the web-integration layer of OpenDroid's tool system. Both tools are **server-side executed** (`executionLocation: "server"`), meaning the client dispatches tool calls to the OpenDroid server which handles the actual HTTP requests. WebSearch provides a search API with keyword/neural/auto modes, category filtering, and domain inclusion/exclusion. FetchUrl provides URL content scraping with extensive URL validation (blocking localhost, private networks, corporate infrastructure), returning HTML-to-markdown converted content. Both tools are registered as top-level, visible-to-user tools in the "Web Search" toolkit. Their outputs are subject to large-output truncation and artifact persistence on the client side.

## Module Map

| Module | Size | Role | Vendor/App |
|--------|------|------|------------|
| 0966.js | 0.4 KB | Tool LLM ID constants (qKA="WebSearch", YKA="FetchUrl") | app-tools |
| 1206.js | 3.6 KB | Zod input/output schemas (dZ$, mZ$, iZ$, lZ$) | app-tools |
| 1207.js | 3.2 KB | FetchUrl tool definition (rc = eI({...})) | app-tools |
| 1232.js | 1.9 KB | WebSearch tool definition (kO = eI({...})) | app-tools |
| 2519.js | 5.5 KB | Tool registry (d1 = {WebSearch: aG(kO), FetchUrl: aG(rc)}) | app-tools |
| 2855.js | 2.3 KB | Large-output truncation & artifact persistence (OZf) | app-tools |
| 2856.js | 9.5 KB | Truncation-eligible tool set (CZf includes web_search, fetch_url) | app-tools |
| 3373.js | 3.8 KB | FetchUrl TUI renderer (JJD) | app-tools |
| 3386.js | 4.6 KB | WebSearch TUI renderer (aJD) | app-tools |
| 3387.js | 2.7 KB | Tool handlers map (GIM = {WebSearch: aJD, FetchUrl: JJD}) | app-tools |
| 3922.js | ~ | GUI tool list with descriptions (WebSearch: "Search the web", FetchUrl: "Fetch URL content") | app-tools |

## Architecture

### Tool Registration Flow

```
0966.js (LLM ID constants)
    ↓
1206.js (Zod schemas: dZ$=WebSearch input, mZ$=WebSearch output, iZ$=FetchUrl input, lZ$=FetchUrl output)
    ↓
1207.js (FetchUrl: rc = eI({id:"fetch_url", llmId:"FetchUrl", ...}))
1232.js (WebSearch: kO = eI({id:"web_search", llmId:"WebSearch", ...}))
    ↓
2519.js (Registry: d1 = {WebSearch: aG(kO), FetchUrl: aG(rc)})
    ↓
3387.js (Handlers: GIM = {WebSearch: aJD, FetchUrl: JJD})
```

### WebSearch Tool

- **ID**: `web_search`
- **LLM ID**: `WebSearch` (qKA constant from 0966.js)
- **Execution**: Server-side (`executionLocation: "server"`)
- **Description**: Template literal with current date injection via `iGH({todayDate})`: ensures LLM uses correct year in search queries
- **Input Schema** (dZ$, 1206.js):
  - `query`: string (required): search query
  - `type`: enum ["keyword", "neural", "auto"] (optional): search mode
  - `category`: enum ["company", "research paper", "news", "pdf", "github", "tweet", "personal site", "linkedin profile", "financial report"] (optional)
  - `numResults`: positive integer (optional): 1-100, default 10
  - `includeDomains`: string[] (optional): domain whitelist
  - `excludeDomains`: string[] (optional): domain blacklist
  - `text`: boolean (optional): include full text content in results
- **Output Schema** (mZ$, 1206.js):
  - `results`: array of `{title, url, publishedDate?, author?, score?, text?, highlights?, summary?, favicon?}`
  - `searchType`: enum ["neural", "keyword", "auto"] (optional)

### FetchUrl Tool

- **ID**: `fetch_url`
- **LLM ID**: `FetchUrl` (YKA constant from 0966.js)
- **Execution**: Server-side (`executionLocation: "server"`)
- **Input Schema** (iZ$, 1206.js):
  - `url`: string.url() (required): URL to scrape
- **Output Schema** (lZ$, 1206.js):
  - `markdown`: string: scraped content in markdown
  - `title`: string | null: page title
  - `metadata`: `{url, statusCode, error, title}`: scrape metadata
  - `linkedUrls`: string[] (optional): integration URLs found in content

### URL Validation (FetchUrl)

The FetchUrl tool has extensive built-in URL validation in its description (1207.js) that blocks:
1. **Local/private network**: localhost, 127.0.0.1, 10.*.*.*, 172.16-31.*.*, 192.168.*.*, 169.254.*.*, *.local, *.internal
2. **Non-HTTP protocols**: file:///, ssh://, ftp://, powershell://, view-source:
3. **Corporate infrastructure**: *.corp.{company}.com, internal staging/production
4. **Invalid patterns**: GitHub pull/new/*, session tokens, malformed URLs

Supported integrations (requires setup): Google Docs, Notion, Linear, GitHub PRs/Issues, ErrTracker, GitLab MRs, Jira, PagerDuty, Slack threads.

### Output Truncation

Both tools are in the truncation-eligible set (2856.js):
```js
CZf = new Set(["grep_tool_cli", "web_search", "fetch_url", "ls-cli", "execute-cli"]);
```
When output exceeds the threshold (2855.js), the `IrI` function truncates and persists to `~/.opendroid/artifacts/tool-outputs/` with guidance for the LLM to use Read/Grep/Execute to access the full artifact.

### TUI Rendering

- **WebSearch renderer** (3386.js): Shows search query in header, result count ("Found N results"), and detailed view with result text
- **FetchUrl renderer** (3373.js): Shows URL in header, title extraction from result, line count, and "✓ Content retrieved" indicator

## Key Findings

### 1. WebSearch Tool Definition (1232.js)

```js
V11 = `Performs a web search to find relevant web pages and documents...
    IMPORTANT - Use the correct year in search queries:
      - Today's date is ${Bz$}. You MUST use this year...
    `;
X11 = iGH({ todayDate: new Date().toISOString().split("T")[0] });
kO = eI({
  id: "web_search",
  llmId: qKA,
  uiGroupId: "web_search",
  displayName: "Web Search",
  description: X11,
  executionLocation: "server",
  inputSchema: dZ$,
  isVisibleToUser: true,
  isTopLevelTool: true,
  requiresConfirmation: false,
  outputSchemas: { result: mZ$ },
  toolkit: "Web Search",
  isToolEnabled: true,
});
```

### 2. WebSearch Input Schema (1206.js)

```js
dZ$ = p.object({
  query: p.string().describe("The search query string"),
  type: p.enum(["keyword", "neural", "auto"]).optional()
    .describe("The type of search. Neural uses embeddings-based model, keyword is google-like SERP. Default is auto."),
  category: p.enum(["company", "research paper", "news", "pdf", "github",
    "tweet", "personal site", "linkedin profile", "financial report"]).optional(),
  numResults: oE1.optional()
    .describe("Number of results to return (default: 10). 1-100 allowed."),
  includeDomains: p.array(p.string()).optional()
    .describe('Array of domains to include (e.g., ["example.com"])'),
  excludeDomains: p.array(p.string()).optional()
    .describe('Array of domains to exclude. Only one of includeDomains or excludeDomains should be used.'),
  text: tE1.optional()
    .describe("Whether to include full text content in results."),
});
```

### 3. WebSearch Output Schema (1206.js)

```js
mZ$ = p.object({
  results: p.array(
    p.object({
      title: p.string(),
      url: p.string(),
      publishedDate: p.string().nullable().optional(),
      author: p.string().nullable().optional(),
      score: p.number().nullable().optional(),
      text: p.string().optional(),
      highlights: p.array(p.string()).optional(),
      summary: p.string().optional(),
      favicon: p.string().optional(),
    }),
  ),
  searchType: p.enum(["neural", "keyword", "auto"]).optional(),
});
```

### 4. FetchUrl Tool Definition (1207.js)

```js
rc = eI({
  id: "fetch_url",
  llmId: YKA,
  uiGroupId: "fetch_url",
  displayName: "Web Fetch",
  description: `Scrapes content from URLs...CRITICAL: BEFORE CALLING THIS TOOL, CHECK IF THE URL WILL FAIL
    URLS THAT WILL ALWAYS FAIL:
    1. LOCAL/PRIVATE NETWORK URLs: http://localhost:*, http://127.0.0.1:*, ...
    2. NON-HTTP PROTOCOLS: file:///, ssh://, ftp://, ...
    3. CORPORATE/INTERNAL INFRASTRUCTURE: *.corp.{company}.com, ...
    4. INVALID/BROKEN URL PATTERNS...`,
  executionLocation: "server",
  inputSchema: iZ$,
  isVisibleToUser: true,
  isTopLevelTool: true,
  requiresConfirmation: false,
  outputSchemas: { result: lZ$ },
  toolkit: "Web Search",
  isToolEnabled: true,
});
```

### 5. FetchUrl Input/Output Schema (1206.js)

```js
iZ$ = p.object({ url: p.string().url().describe("The URL to scrape content from") });

lZ$ = p.object({
  markdown: p.string().describe("The scraped content in markdown format"),
  title: p.string().nullable().describe("The title of the webpage"),
  metadata: p.object({
    url: p.string().describe("The URL that was scraped"),
    statusCode: p.number().describe("The HTTP status code of the response"),
    error: p.string().nullable().describe("Error message if any"),
    title: p.string().nullable().describe("The title of the webpage"),
  }).describe("Metadata about the scraped webpage"),
  linkedUrls: p.array(p.string()).optional()
    .describe("Array of integration URLs found in the content"),
});
```

### 6. Output Truncation Eligibility (2856.js + 2855.js)

```js
// 2856.js - Tools eligible for output truncation
CZf = new Set(["grep_tool_cli", "web_search", "fetch_url", "ls-cli", "execute-cli"]);

// 2855.js - Truncation logic
function OZf(H) {
  if (CZf.has(H)) return true;
  if (H.startsWith("mcp_")) return true;
  return false;
}
async function IrI(H, A, L = yt) {
  if (!OZf(A.toolId)) return H;
  if (H.length <= L) return H;
  return ZZf(H, A, L); // Persists to ~/.opendroid/artifacts/tool-outputs/
}
```

### 7. Tool Registry Entry (2519.js)

```js
d1 = {
  Read: aG(tR), Grep: aG(tc), Glob: aG(ac), Ls: aG(oc),
  Create: aG(eR), Edit: aG(sR), Execute: aG(aR), AskUser: aG(sc),
  WebSearch: aG(kO),
  FetchUrl: aG(rc),
  ApplyPatch: aG(Hh), TodoWrite: aG(bO), Skill: aG(U2),
  ExitSpecMode: aG(M2), GenerateDroid: aG(ec), slackPostMessage: aG(qQ),
};
```

## Integration Points

- **02-orchestration (Orchestration/Validation)**: Both WebSearch and FetchUrl appear in tool-confirmation flows and skill prompts (2175.js references WebSearch/FetchUrl for online research).
- **04-desktop-gui (GUI/Electron)**: 3922.js renders tool list in the GUI with descriptions "Search the web" and "Fetch URL content".
- **01-terminal-ui (TUI/Ink)**: TUI renderers in 3386.js (WebSearch) and 3373.js (FetchUrl) render Ink components for CLI display.
- **Permission System**: Both tools have `requiresConfirmation: false` and `isTopLevelTool: true`, suggesting they bypass the sandbox/permission gate.
- **OpenAI API Integration**: 1465.js shows `web_search_options` being injected as tool parameters in OpenAI API calls, indicating web search may also be available as a native model feature via OpenAI's web search tool.
- **MCP Tools**: Output truncation is also applied to MCP tools (`H.startsWith("mcp_")`), sharing the same artifact persistence mechanism.

## Implementation Notes

### Tool API
- **WebSearch**: Call with `{query, type?, category?, numResults?, includeDomains?, excludeDomains?, text?}` → returns `{results: [{title, url, ...}], searchType?}`
- **FetchUrl**: Call with `{url}` → returns `{markdown, title, metadata: {url, statusCode, error, title}, linkedUrls?}`
- Both use Zod schemas for input validation; schemas are serializable for LLM function-calling manifests

### URL Validation
- FetchUrl's validation is **prompt-level** (embedded in tool description), not code-level
- For a port implementation, consider adding **programmatic URL validation** (parse URL, check against private IP ranges, block restricted protocols)

### HTML-to-Markdown
- The `html_to_markdown` function referenced in 3336.js is a Nunjucks template filter, likely part of the server-side rendering
- Actual HTML-to-markdown conversion happens server-side; client receives pre-converted markdown
- For a port, use a library like `turndown` or `html-to-md` for this conversion step

### Output Truncation
- Implement the artifact persistence pattern: when output exceeds threshold, save to file and return truncation notice with file path
- The truncation set (`CZf`) is configurable per tool

## Open Questions / Left Undone

1. **Server-side execution implementation**: The actual HTTP fetching and HTML-to-markdown conversion happens on the OpenDroid server, which is not included in the client-side compiled modules. The exact search API provider (likely Tavily or similar) is unknown.
2. **Web search API provider**: The search API backing WebSearch is not identifiable from client-side code alone: only the input/output schemas are visible.
3. **Integration URL scraping**: The FetchUrl tool mentions supported integration URLs (Google Docs, Notion, etc.), but the specific scraper implementations for each integration are server-side.
4. **HTML-to-markdown pipeline**: The actual conversion library/process is server-side. The `html_to_markdown` reference in 3336.js appears to be a Nunjucks filter, possibly unrelated to the tool's conversion.
5. **Domain validation enforcement**: URL validation in FetchUrl is description-based (LLM prompted to validate), not programmatically enforced. Need to verify if server-side validation also exists.
