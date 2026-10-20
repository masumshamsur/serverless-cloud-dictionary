# Deploying the Frontend: GitHub and AWS Amplify

This guide covers pushing the React app to your own GitHub repository and deploying it with AWS Amplify.

Before you start, make sure the backend is working: the DynamoDB table, the Lambda function and API Gateway. Also set `apiUrl` in `src/App.js` to your API Gateway Invoke URL. See the main [README](../README.md) for those steps.

## 1. Create your own GitHub repository

1. Log in to GitHub and create a new repository.
2. Enter a repository name, such as `serverless-cloud-dictionary`.
3. (Optional) Add a description, such as *Serverless Cloud Dictionary Application Frontend*.
4. **Don't** initialize the repository with a README or `.gitignore`. This project already has both.
5. Click **Create repository**.

## 2. Push the code to your new repository

From the project directory, run:

```bash
git init
git add .
git commit -m "Initial commit"
git remote remove origin
git remote add origin <your-new-repository-url>
git branch -M main
git push -u origin main
```

| Command | What it does |
|---|---|
| `git init` | Initializes a Git repository in the project folder |
| `git add .` | Stages all project files for commit |
| `git commit -m "Initial commit"` | Commits the staged files |
| `git remote remove origin` | Removes any existing `origin` remote, such as one left over from a cloned template |
| `git remote add origin <your-new-repository-url>` | Links the local repository to your new GitHub repository |
| `git branch -M main` | Renames the current branch to `main` |
| `git push -u origin main` | Pushes the code to `main` and sets it as the default upstream branch |

> **Notes**
> - If the folder has no `origin` remote yet, `git remote remove origin` prints `error: No such remote 'origin'`. That's harmless, so continue with the next command.
> - The project's `.gitignore` keeps `node_modules/`, `build/` and `.env` files out of the repository. Make sure it's present before running `git add .`.

## 3. Deploy the application to AWS Amplify

1. Log in to the **AWS Management Console** and go to **AWS Amplify**.
2. Click **Deploy an app**.
3. Select **GitHub** as the source, and authorize AWS Amplify if prompted.
4. Choose your new repository and the branch (usually `main`).
5. Amplify detects the React app and configures the build settings automatically: build command `npm run build`, output directory `build`. Review the configuration, then click **Save and deploy**.
6. Wait for Amplify to finish the **Provision → Build → Deploy** steps.
7. When the deployment finishes, Amplify gives you a URL such as `https://main.<app-id>.amplifyapp.com`.
8. Open the URL in a browser and search for a term, such as `aws kms`, to confirm the app is live and connected to the API.

Every push to the connected branch triggers a new Amplify build and deployment.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `git push` is rejected with `failed to push some refs` | The GitHub repository was initialized with a README or `.gitignore`. Create it again empty, or run `git pull origin main --allow-unrelated-histories` before pushing. |
| `remote origin already exists` | Run `git remote remove origin` first, then add the remote again. |
| Amplify build fails during `npm install` | Open the build logs in Amplify. Check that `package-lock.json` is committed and `node_modules/` isn't. |
| The app loads, but **Search** shows nothing | Open the browser console. Check that `apiUrl` in `src/App.js` points to your API, and that the API returns CORS headers. |
