# Logging Attachments in Braintrust Logs

Adding attachments to Braintrust logs is great for capturing images, PDFs, or other files for your users, but what if you want to add those logs to a dataset and test them in the Braintrust Playground? In this cookbook we will show you how to run prompts in the Playground that use attachments in three separate ways. We’ll be using TypeScript to log a public URL and a base64 string to the span.

The three methods:
- Manual attachment from your computer  
- Public URL  
- Base64 values  

You’ll learn how to:
- Emit spans with embedded file attachments  
- Push your logs into a Braintrust dataset  
- “Try Prompt” in the Playground, save to your library, and iterate  

---

## Prerequisites

- Node.js & pnpm 
- A Braintrust account and a valid `BRAINTRUST_API_KEY` in your environment  
- A `pdfs/` directory at your project root containing one or more PDF files  
- This repo cloned locally  

## Installation

```bash
# Install system and JS dependencies
brew install node pnpm
pnpm install
```

## 1. Setup Braintrust + OpenAI client

In `simulate_logging.ts`, initialize the logger and wrapped OpenAI client:

```typescript
1:8:simulate_logging.ts
import { initLogger, wrapOpenAI, wrapTraced, currentSpan, Attachment } from "braintrust";
import { OpenAI } from "openai";
import path from "path";
import fs from "fs";

// Initialize Braintrust logger
const logger = initLogger({
  projectName: "pdf-attachment-demo",
  apiKey: process.env.BRAINTRUST_API_KEY,
});

// Wrap OpenAI with tracing
const client = wrapOpenAI(
  new OpenAI({
    baseURL: "https://braintrustproxy.com/v1",
    apiKey: process.env.BRAINTRUST_API_KEY,
  }),
);
```

## 2. Three Ways to Attach Files

Below is the “user prompt” span logger. We’ll illustrate three alternative metadata payloads.

```typescript
139:164:simulate_logging.ts
async function logUserPrompt(
  filename: string,
  url: string,
  userPrompt: string,
  summary: string,
  rootSpan: Promise<string>,
  base64String: string
) {
  const userPromptSpan = wrapTraced(async () => {
    const span = currentSpan();
    span.setAttributes({
      name: "user_prompt",
      type: "llm",
      parent: (await rootSpan).toString(),
    });

    // COMMON: the LLM input & output
    span.log({
      input: [{ role: "user", content: userPrompt }],
      output: summary,
      metadata: {
        filename,
        url,
        base64String,
        /** attachment will vary per method **/
      }
    });
  }, logger);
  await userPromptSpan();
}
```

### A. Manual Attachment from Your Computer

Read the file into a Buffer and attach it directly:

```typescript
// ... existing code in logUserPrompt ...

// Read the local PDF file
const pdfPath = path.join(process.cwd(), "pdfs", filename);
const pdfData = fs.readFileSync(pdfPath);

const attachment = new Attachment({
  data: pdfData,
  filename,
  contentType: "application/pdf",
});

span.log({
  input: [{ role: "user", content: userPrompt }],
  output: summary,
  metadata: {
    filename,
    url,
    base64String,
    attachment
  }
});
```

> In the Braintrust UI you’ll see the PDF embedded in the log span. Click **Try Prompt** to load it into the Playground.

### B. Public URL

If your PDF is already hosted, you can simply log its URL:

```typescript
span.log({
  input: [{ role: "user", content: userPrompt }],
  output: summary,
  metadata: {
    filename,
    url // e.g. "https://example.com/transcript.pdf"
  }
});
```

> The Playground will fetch the file from that URL when you **Try Prompt**.

### C. Base64 Values

Embed the PDF as a base64 string:

```typescript
span.log({
  input: [{ role: "user", content: userPrompt }],
  output: summary,
  metadata: {
    filename,
    base64String // e.g. "data:application/pdf;base64,JVBERi0xLjc..."
  }
});
```

> The Playground decodes the base64, reattaches the file, and runs the prompt.

---

## 3. Run & Push Logs

```bash
# Generate spans for each PDF
pnpm logging

# Push your code & spans to Braintrust
braintrust push simulate_logging.ts
```

1. Go to **Logs → Dataset** in the Braintrust UI  
2. Filter by `projectName: pdf-attachment-demo`  
3. Select spans with attachments  
4. Click **Try Prompt** to launch the Playground  
5. Save prompts to your library and iterate  

---

## Next Steps

- Use these attachments in Braintrust Evals for attachment-based evaluations  
- Combine attachments with other metadata for richer datasets  
- Automate log pushes in CI for continuous prompt tuning  

```jsonc
// package.json snippet
{
  "scripts": {
    "logging": "ts-node simulate_logging.ts"
  },
  "dependencies": {
    "braintrust": "^0.0.201",
    "openai": "^4.97.0"
  },
  "devDependencies": {
    "ts-node": "^10.9.2",
    "typescript": "^5.4.2"
  }
}
```
