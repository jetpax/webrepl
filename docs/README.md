# GitHub Pages Setup

This directory contains the files for the GitHub Pages site hosting the WebREPL Binary Protocol specification.

## Files

- `index.html` - Main HTML page that renders the RFC
- `webrepl_binary_protocol_rfc.md` - The RFC specification in Markdown format

## Enabling GitHub Pages

1. Go to your repository settings on GitHub
2. Navigate to **Pages** (under "Code and automation")
3. Under **Source**, select:
   - **Branch**: `main` (or your default branch)
   - **Folder**: `/docs`
4. Click **Save**

GitHub Pages will be available at:
- **HTML version**: `https://jetpax.github.io/webrepl/`
- **Markdown source**: `https://jetpax.github.io/webrepl/webrepl_binary_protocol_rfc.md`

## Testing Locally

You can test the HTML page locally by opening `index.html` in a web browser, or by using a local web server:

```bash
# Using Python 3
cd docs
python3 -m http.server 8000

# Then open http://localhost:8000 in your browser
```

## Notes

- The HTML page uses [marked.js](https://marked.js.org/) from a CDN to render the Markdown
- The page will automatically fetch and render `webrepl_binary_protocol_rfc.md`
- GitHub Pages typically takes a few minutes to update after pushing changes
