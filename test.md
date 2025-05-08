Here are three approaches to handle version incrementing in your Octopus Deploy workflow:

### Option 1: Auto-increment Based on Latest Release (Recommended)
```yaml
- name: Get and Increment Version
  id: version
  run: |
    # Get latest release version
    latest=$(octopus release list --project "githubdeploytestapp" --space "Spaces-22" --output-format=json | jq -r '.[0].Version')
    echo "Latest version: $latest"
    
    # Auto-increment patch version (0.0.X+1)
    IFS='.' read -ra ver <<< "$latest"
    new_version="${ver[0]}.${ver[1]}.$((ver[2]+1))"
    echo "New version: $new_version"
    echo "new_version=$new_version" >> $GITHUB_OUTPUT
```

### Option 2: Let Octopus Auto-Generate Version
```yaml
- name: Create Release
  run: |
    octopus create-release --project "githubdeploytestapp" \
      --space "Spaces-22" \
      --packageVersion "auto" \  # Let Octopus generate version
      --variable "dbName:${{ inputs.database_name }}" \
      --variable "SQLScripts:${{ inputs.script_url }}"
```

### Option 3: Date-Based Versioning
```yaml
- name: Generate Version
  id: version
  run: |
    date=$(date +'%Y.%m.%d')
    count=$(octopus release list --project "githubdeploytestapp" --space "Spaces-22" --filter "$date" | wc -l)
    new_version="$date.$((count+1))"
    echo "new_version=$new_version" >> $GITHUB_OUTPUT
```

### Best Practice Implementation:
```yaml
- name: Handle Versioning
  id: version
  run: |
    # Try to get latest version
    if latest_json=$(octopus release list --project "githubdeploytestapp" --space "Spaces-22" --output-format=json); then
      latest=$(echo "$latest_json" | jq -r '.[0].Version')
      echo "Found latest version: $latest"
      IFS='.' read -ra ver <<< "$latest"
      new_version="${ver[0]}.${ver[1]}.$((ver[2]+1))"
    else
      echo "No existing releases, starting at 1.0.0"
      new_version="1.0.0"
    fi
    
    echo "Using version: $new_version"
    echo "new_version=$new_version" >> $GITHUB_OUTPUT
```

Key Notes:
1. The first option gives you precise control over version increments
2. The second option is simplest but gives less visibility
3. The third option provides date-based tracking
4. The best practice version includes error handling for first-time deployments

Choose the method that best fits your:
- Release frequency
- Version tracking requirements
- Compliance/audit needs
- Team preferences

All methods will work with your existing deployment workflow - just replace the version generation step with your preferred option.
