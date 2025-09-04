# Vue 3 + Vite

This template should help get you started developing with Vue 3 in Vite. The template uses Vue 3 `<script setup>` SFCs, check out the [script setup docs](https://v3.vuejs.org/api/sfc-script-setup.html#sfc-script-setup) to learn more.

Learn more about IDE Support for Vue in the [Vue Docs Scaling up Guide](https://vuejs.org/guide/scaling-up/tooling.html#ide-support).

## Deploy to GitHub Pages

To deploy to GitHub Pages, you can use either of the following methods:

1. **Automatic deployment with GitHub Actions** (Recommended):
   - The project includes a GitHub Actions workflow in `.github/workflows/deploy.yml`
   - Push to the `main` branch to trigger automatic deployment
   - Or manually trigger the workflow from the GitHub Actions tab

2. **Manual deployment**:
   - Build the project: `npm run build`
   - Deploy using the deploy script: `npm run deploy`
   - Note: For manual deployment, you need to set up GitHub personal access token with `public_repo` scope as `GH_TOKEN` environment variable.
