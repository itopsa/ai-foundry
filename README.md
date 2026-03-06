# ai-foundry

## Viewing the HTML diagrams

GitHub does **not** render HTML files when you open them in the repo; it only shows the source. To view the drawio diagrams as rendered pages:

1. **Enable GitHub Pages** (one-time):
   - Repo **Settings** → **Pages**
   - Under **Build and deployment**, set **Source** to **Deploy from a branch**
   - **Branch**: `main` (or your default branch), **Folder**: `/ (root)`
   - Save. After a minute, the site will be at `https://<owner>.github.io/ai-foundry/`

2. **Open the diagrams** (replace `<owner>` with your org or username):
   - [AI Foundry ReadMe 1](https://<owner>.github.io/ai-foundry/html/AI%20Foundry%20ReadMe%201.drawio.html)
   - [AI Foundry ReadMe 2](https://<owner>.github.io/ai-foundry/html/AI%20Foundry%20ReadMe%202.drawio.html)
   - [AI Foundry ALZ](https://<owner>.github.io/ai-foundry/html/AI%20Foundry%20ALZ.drawio.html)
   - [AI Foundry Decision Tree](https://<owner>.github.io/ai-foundry/html/AI%20Foundry%20Decision%20Tree.drawio.html)

The `.nojekyll` file in this repo tells GitHub Pages to serve the files as-is so the HTML diagrams render correctly.
