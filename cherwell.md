# Enhanced GitHub Action Workflow with Cherwell Integration

Here's an updated workflow that includes both the Octopus deployment to UAT and updating a Cherwell table with the SQL filename:

```yaml
name: SQL Deployment to UAT with Cherwell Update
on:
  workflow_dispatch:
    inputs:
      sqlFileName:
        description: 'Name of the SQL file to deploy'
        required: true
        type: string
        default: 'script.sql'
      cherwellRecordId:
        description: 'Cherwell record ID to update'
        required: true
        type: string

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
          octo push --package=SqlScripts.1.0.0-${{ github.run_id }}.zip --server=$OCTOPUS_URL --apiKey=$OCTOPUS_API_KEY

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

      - name: Update Cherwell record
        env:
          CHERWELL_API_KEY: ${{ secrets.CHERWELL_API_KEY }}
          CHERWELL_BASE_URL: ${{ secrets.CHERWELL_BASE_URL }}
        run: |
          # Update Cherwell table with the SQL filename
          curl -X POST \
            "$CHERWELL_BASE_URL/api/V1/updatebusinessobject" \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer $CHERWELL_API_KEY" \
            -d '{
              "busObId": "your_business_object_id",
              "busObRecId": "${{ inputs.cherwellRecordId }}",
              "fields": [
                {
                  "fieldId": "your_field_id_for_filename",
                  "value": "${{ inputs.sqlFileName }}"
                },
                {
                  "fieldId": "your_field_id_for_status",
                  "value": "Deployed to UAT"
                },
                {
                  "fieldId": "your_field_id_for_deployment_date",
                  "value": "$(date -u +'%Y-%m-%dT%H:%M:%SZ')"
                }
              ]
            }'

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

      - name: Verify deployment and update Cherwell
        if: always()
        env:
          CHERWELL_API_KEY: ${{ secrets.CHERWELL_API_KEY }}
          CHERWELL_BASE_URL: ${{ secrets.CHERWELL_BASE_URL }}
        run: |
          # Check if previous steps succeeded
          if [ "${{ job.status }}" == "success" ]; then
            status="Deployment Successful"
          else
            status="Deployment Failed"
          fi

          # Update Cherwell with final status
          curl -X POST \
            "$CHERWELL_BASE_URL/api/V1/updatebusinessobject" \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer $CHERWELL_API_KEY" \
            -d '{
              "busObId": "your_business_object_id",
              "busObRecId": "${{ inputs.cherwellRecordId }}",
              "fields": [
                {
                  "fieldId": "your_field_id_for_status",
                  "value": "'"$status"'"
                },
                {
                  "fieldId": "your_field_id_for_completion_date",
                  "value": "$(date -u +'%Y-%m-%dT%H:%M:%SZ')"
                }
              ]
            }'
```

## Key Enhancements:

1. **Cherwell Integration**:
   - Added a Cherwell record ID as a workflow input
   - Two API calls to Cherwell:
     - First after packaging to update with filename
     - Final update with deployment status (success/failure)

2. **Additional Setup Required**:
   - Add Cherwell API credentials to GitHub secrets:
     - `CHERWELL_API_KEY`
     - `CHERWELL_BASE_URL` (your Cherwell instance URL)
   - Identify your Cherwell:
     - Business Object ID (`your_business_object_id`)
     - Field IDs for:
       - SQL filename (`your_field_id_for_filename`)
       - Status field (`your_field_id_for_status`)
       - Date fields

3. **Workflow Improvements**:
   - The final Cherwell update runs `if: always()` to ensure status is recorded even if deployment fails
   - Includes timestamps for deployment start and completion

## Usage Notes:

1. Trigger the workflow with:
   - SQL filename (required)
   - Cherwell record ID (required)

2. The workflow will:
   - Package and deploy the SQL file via Octopus
   - Update Cherwell before and after deployment
   - Maintain a complete audit trail in Cherwell

3. You may need to adjust the Cherwell API payload structure based on your specific implementation.
