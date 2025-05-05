# GitHub Action Workflow for SQL Deployment to UAT using Octopus

Here's a GitHub Actions workflow that allows you to dispatch a workflow with an input parameter (SQL file name) and deploy it to your UAT environment using Octopus Deploy:

```yaml
name: SQL Deployment to UAT
on:
  workflow_dispatch:
    inputs:
      sqlFileName:
        description: 'Name of the SQL file to deploy'
        required: true
        type: string
        default: 'script.sql'

jobs:
  deploy-sql-to-uat:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Validate SQL file exists
        id: validate-sql
        run: |
          if [ ! -f "${{ github.workspace }}/${{ inputs.sqlFileName }}" ]; then
            echo "::error::SQL file ${{ inputs.sqlFileName }} not found!"
            exit 1
          fi
          echo "SQL file ${{ inputs.sqlFileName }} found"

      - name: Setup Octopus CLI
        uses: OctopusDeploy/install-octopus-cli-action@v1
        with:
          version: latest

      - name: Package SQL file
        run: |
          octo pack --id="SqlScripts" --format=Zip --version=1.0.0-${{ github.run_id }} --basePath=${{ github.workspace }} --include=${{ inputs.sqlFileName }}

      - name: Push package to Octopus
        env:
          OCTOPUS_API_KEY: ${{ secrets.OCTOPUS_API_KEY }}
          OCTOPUS_URL: ${{ secrets.OCTOPUS_URL }}
        run: |
          octo push --package=SqlScripts.1.0.0-${{ github.run_id }}.zip --server=$OCTOPUS_URL --apiKey=$OCTOPUS_KEY

      - name: Create Octopus release
        env:
          OCTOPUS_API_KEY: ${{ secrets.OCTOPUS_API_KEY }}
          OCTOPUS_URL: ${{ secrets.OCTOPUS_URL }}
        run: |
          octo create-release --project="Your-Project-Name" \
                              --version="1.0.0-${{ github.run_id }}" \
                              --package="SqlScripts:1.0.0-${{ github.run_id }}" \
                              --server=$OCTOPUS_URL \
                              --apiKey=$OCTOPUS_API_KEY \
                              --deployto="UAT"

      - name: Deploy to UAT
        env:
          OCTOPUS_API_KEY: ${{ secrets.OCTOPUS_API_KEY }}
          OCTOPUS_URL: ${{ secrets.OCTOPUS_URL }}
        run: |
          octo deploy-release --project="Your-Project-Name" \
                             --version="1.0.0-${{ github.run_id }}" \
                             --deployto="UAT" \
                             --server=$OCTOPUS_URL \
                             --apiKey=$OCTOPUS_API_KEY \
                             --waitfordeployment
```

## Prerequisites:

1. Set up the following secrets in your GitHub repository:
   - `OCTOPUS_API_KEY`: Your Octopus Deploy API key
   - `OCTOPUS_URL`: Your Octopus Deploy server URL

2. Configure your Octopus Deploy environment:
   - Create a project in Octopus that handles SQL deployments
   - Set up a UAT environment in Octopus
   - Configure your deployment process to handle the SQL file package

## Usage:

1. Manually trigger the workflow from GitHub Actions UI
2. Provide the SQL file name as input (e.g., "migration_script.sql")
3. The workflow will:
   - Verify the SQL file exists in your repo
   - Package it using Octopus CLI
   - Push it to your Octopus server
   - Create and deploy a release to UAT environment

## Notes:

- Adjust the project name ("Your-Project-Name") to match your Octopus project
- You may need to modify the packaging step if your SQL files are in a specific directory
- The version number includes the GitHub run ID to ensure uniqueness
- Consider adding additional validation or testing steps as needed
