# Screenshots

Only when the user explicitly asked for images or screenshots. Never take or attach them on your own.

- **User gave a file path:** attach that file and skip Playwright.
- **Otherwise take them with the Playwright MCP.**  1. Open the relevant page (the URL given, or the page related to the ticket).
  2. Reproduce the steps if needed to reach the state to capture.
  3. Screenshot the relevant element or area, not the whole page, unless a full page was requested.
  4. Save to `./screenshots/CU-<id>-<short-slug>.png`, numbered `-1`, `-2` when there are several.
- **Attach as files, never inside the description.** For each file: call `clickup_request_attachment_upload` with the task id and file name, then POST it with curl, using an argument list and the ticket from the result:
  `curl -s -X POST <upload_url> -H "X-Upload-Ticket: <ticket>" -F "attachment=@<path>;type=image/png"`
  The ticket expires in a few minutes, so request it right before each upload.
- **Evidence section:** list each attached screenshot with one line saying what it shows, then update the ticket description with it.
- **Failures:** if Playwright is not connected or the page can't be reached, say so and keep the ticket without screenshots. If the upload returns `Over allocated storage` (`GBUSED_005`), the workspace storage quota is full, not the file too big: report it, keep the files on disk and give the path.
