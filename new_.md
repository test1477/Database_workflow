Here's the enhanced GitHub Actions workflow that includes all requested fields (Database Name, Database Script URL, and Database Script Summary) with validation and Cherwell table updates:

```yaml
name: SQL Deployment with Full Tracking
on:
  workflow_dispatch:
    inputs:
      databaseName:
        description: 'Target database name'
        required: true
        type: string
      scriptUrl:
        description: 'Raw GitHub URL of the SQL script/zip'
        required: true
        type: string
      scriptSummary:
        description: 'Brief description of the script changes'
        required: true
        type: string
      cherwellRecordId:
        description: 'Cherwell record ID to update'
        required: true
        type: string

jobs:
  deploy-sql:
    runs-on: ubuntu-latest
    steps:
      - name: Validate inputs
        id: validate
        run: |
          # Validate script URL
          if [[ "${{ inputs.scriptUrl }}" != *".sql" && "${{ inputs.scriptUrl }}" != *".zip" ]]; then
            echo "::error::URL must point to a .sql or .zip file"
            exit 1
          fi
          
          # Verify file exists
          response=$(curl -s -o /dev/null -w "%{http_code}" "${{ inputs.scriptUrl }}")
          if [ "$response" -ne 200 ]; then
            echo "::error::File not found at URL (HTTP $response)"
            exit 1
          fi
          
          echo "All inputs validated successfully"

      - name: Update Cherwell record (pre-deployment)
        env:
          CHERWELL_API_KEY: ${{ secrets.CHERWELL_API_KEY }}
          CHERWELL_BASE_URL: ${{ secrets.CHERWELL_BASE_URL }}
        run: |
          # Extract filename from URL
          filename=$(basename "${{ inputs.scriptUrl }}" | cut -d'?' -f1)
          
          # Update test1477 table
          curl -X POST \
            "$CHERWELL_BASE_URL/api/V1/updatebusinessobject" \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer $CHERWELL_API_KEY" \
            -d '{
              "busObId": "test1477",
              "busObRecId": "${{ inputs.cherwellRecordId }}",
              "fields": [
                {
                  "fieldId": "database_name",
                  "value": "${{ inputs.databaseName }}"
                },
                {
                  "fieldId": "script_url",
                  "value": "${{ inputs.scriptUrl }}"
                },
                {
                  "fieldId": "script_filename",
                  "value": "'"$filename"'"
                },
                {
                  "fieldId": "script_summary",
                  "value": "${{ inputs.scriptSummary }}"
                },
                {
                  "fieldId": "deployment_status",
                  "value": "Validation passed - deploying"
                },
                {
                  "fieldId": "last_updated",
                  "value": "'$(date -u +"%Y-%m-%dT%H:%M:%SZ")'"
                }
              ]
            }'

      - name: Setup Octopus CLI
        uses: OctopusDeploy/install-octopus-cli-action@v1
        with:
          version: latest

      - name: Trigger Octopus deployment
        env:
          OCTOPUS_API_KEY: ${{ secrets.OCTOPUS_API_KEY }}
          OCTOPUS_URL: ${{ secrets.OCTOPUS_URL }}
        run: |
          octo create-release --project="SQL-Deployments" \
                              --version="1.0.${{ github.run_id }}" \
                              --package="SQLScripts:${{ inputs.scriptUrl }}" \
                              --server=$OCTOPUS_URL \
                              --apiKey=$OCTOPUS_API_KEY \
                              --deployto="UAT" \
                              --variable="DatabaseName:${{ inputs.databaseName }}" \
                              --variable="SQLScriptUrl:${{ inputs.scriptUrl }}" \
                              --variable="ScriptSummary:${{ inputs.scriptSummary }}"

      - name: Update Cherwell (post-deployment)
        if: always()
        env:
          CHERWELL_API_KEY: ${{ secrets.CHERWELL_API_KEY }}
          CHERWELL_BASE_URL: ${{ secrets.CHERWELL_BASE_URL }}
        run: |
          status=$([ "${{ job.status }}" == "success" ] && echo "Deployed successfully" || echo "Deployment failed")
          
          curl -X POST \
            "$CHERWELL_BASE_URL/api/V1/updatebusinessobject" \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer $CHERWELL_API_KEY" \
            -d '{
              "busObId": "test1477",
              "busObRecId": "${{ inputs.cherwellRecordId }}",
              "fields": [
                {
                  "fieldId": "deployment_status",
                  "value": "'"$status"'"
                },
                {
                  "fieldId": "completion_time",
                  "value": "'$(date -u +"%Y-%m-%dT%H:%M:%SZ")'"
                },
                {
                  "fieldId": "deployment_id",
                  "value": "${{ github.run_id }}"
                }
              ]
            }'
```

### Key Features:

1. **Complete Field Tracking**:
   - Database Name (passed to Octopus and Cherwell)
   - Database Script URL (validated before deployment)
   - Database Script Summary (stored in Cherwell)

2. **Enhanced Cherwell Updates**:
   - Pre-deployment update with all metadata:
     ```json
     {
       "database_name": "YourDB",
       "script_url": "https://github.com/.../script.sql",
       "script_filename": "script.sql",
       "script_summary": "Added new customer table",
       "deployment_status": "Validation passed - deploying"
     }
     ```
   - Post-deployment status update (success/failure)

3. **Validation**:
   - Ensures URL points to .sql/.zip file
   - Verifies file exists (HTTP 200 check)
   - Validates all required inputs

4. **Audit Trail**:
   - GitHub run ID stored in Cherwell
   - Timestamps for all operations
   - Full deployment status history

### Setup Instructions:

1. Add these secrets to your GitHub repo:
   - `CHERWELL_API_KEY`
   - `CHERWELL_BASE_URL`
   - `OCTOPUS_API_KEY`
   - `OCTOPUS_URL`

2. Configure these Cherwell fields in the "test1477" table:
   - `database_name`
   - `script_url`
   - `script_filename`
   - `script_summary`
   - `deployment_status`
   - `last_updated`
   - `completion_time`
   - `deployment_id`

3. The workflow maintains your existing Octopus:
   - Project name ("SQL-Deployments")
   - UAT environment
   - Variable passing structure

This implementation provides full visibility into deployments while maintaining your existing Octopus process exactly as it was with the Cherwell interface.
