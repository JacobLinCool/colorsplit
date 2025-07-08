# PDF Color Split

A web application that automatically separates color pages from black & white pages in PDF files.

## Features

- Upload multiple PDF files
- Automatic color detection with adjustable threshold
- Client-side processing (no server upload required)
- Supports multiple languages (English, Traditional Chinese)
- Download separate color and B&W PDFs

## Development

```bash
# Install dependencies
pnpm install

# Start development server
pnpm dev

# Build for production
pnpm build
```

## Tech Stack

- SvelteKit
- TypeScript
- Tailwind CSS
- PDF.js & PDF-lib
- Paraglide.js (i18n)

## Privacy

All PDF processing happens locally in your browser. No files are uploaded to any server.
