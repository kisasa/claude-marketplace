---
description: Collaborate with agents to create a Linear backlog from meeting notes
argument-hint: [agent1,agent2,...] [notes-path]
---

You are orchestrating the creation of a comprehensive Linear backlog by collaborating with specialized agents who understand the scope of work.

**Agents**: $1 (comma-delimited list of agent names who understand the work scope) (required)

**Notes Path**: $2 (folder or file path containing meeting notes that define the work) (required)

## Prerequisites

Before starting, verify that:
1. The Linear MCP server is available and accessible
2. You have full permissions to interact with Linear (teams, issues, labels, workflows)
3. The notes path exists and contains work definition

## Phase 1: Discovery & Context Gathering

### Step 1.1: Load Meeting Notes
Read all files from the provided notes path ($2) to understand:
- The project/feature scope
- Business requirements and goals
- Technical constraints
- Success criteria
- Any decisions already made

### Step 1.2: Identify Agent Capabilities
For each agent in $1:
- Confirm the agent exists in the kisasa marketplace
- Understand what aspects of work they can help define
- Note their specific domain expertise

## Phase 2: Linear Team Setup

### Step 2.1: Ask for Team Name
Ask the user: "What Linear team will hold this backlog?"

If the user doesn't have a preference, present the question to the agents and let them analyze the notes to recommend 3 appropriate team names.

### Step 2.2: Check Team Existence
Use the Linear MCP server to check if the team exists.

If the team doesn't exist:
1. Present 3 recommended team names based on the work scope (if not already done)
2. Ask the user to choose one or provide a custom name
3. Create the team in Linear

## Phase 3: Workflow Configuration

### Step 3.1: Create Custom Statuses
Use the Linear MCP server to ensure the following workflow statuses:

1. **Backlog** (Category: Backlog)
   - Initial status for newly created issues that haven't been prioritized yet

2. **Todo** (Category: Unstarted)
   - Issues that are ready to be worked on

3. **In Progress** (Category: Started)
   - Issues currently being worked on
   - Note: If the issue includes an agent label, the webhook bridge will engage the agent to begin work

4. **Needs Review** (Category: Started)
   - Development completed, PR opened, review needed

5. **Done** (Category: Completed)
   - Finished issues

6. **Canceled** (Category: Canceled)
   - Issues that won't be pursued

**Workflow progression**: Backlog → Todo → In Progress → Review → Done | Canceled

### Step 3.2: Set Default Status
Ensure "Backlog" is set as the default status for new issues.

## Phase 4: Label Group Configuration

### Step 4.1: Create "Agents" Label Group
Create a label group called "Agents" (single-select, so no issue can have more than one agent):

For each agent in the ./claude/agents, create a label with:
- Name: Agent's name (e.g., "setup-agent")
- Group: Agents
- Color: Assign a distinct color for visual differentiation

### Step 4.2: Create "Types" Label Group
Create a label group called "Types" (single-select):
- Bug
- Feature
- Improvement
- Refactor
- Docs

### Step 4.3: Agent-Suggested Labels
Present the meeting notes to the agents and ask if they recommend any additional label groups or labels. If they do:
1. Document their recommendations with rationale
2. Ask the user for approval
3. Create approved labels in Linear

## Phase 5: Backlog Generation

### Step 5.1: Collaborative Issue Definition
Engage with the specified agents ($1) to break down the work into issues:

**For each agent in turn:**
1. Present the meeting notes and existing context
2. Ask the agent to identify issues they can help define based on their expertise
3. Have the agent specify for each issue:
   - Title (clear, actionable)
   - Description (comprehensive, with acceptance criteria)
   - Type label (Bug, Feature, Improvement, Refactor, Docs)
   - Agent label (which agent in ./claude/agents should handle this work)
   - Priority/urgency assessment
   - Dependencies on other issues
   - Parent issue relationship (if this is a sub-issue)
   - Estimated complexity (Small, Medium, Large)

### Step 5.2: Cross-Agent Review
After all agents have proposed their issues:
1. Present the complete list to all agents for review
2. Identify any gaps, overlaps, or conflicts
3. Resolve issues through agent discussion
4. Refine the backlog based on feedback

### Step 5.3: User Review
Present the proposed backlog to the user:
- Show a summary table with: ID, Title, Type, Agent, Priority, Dependencies
- Ask if they want to modify, add, or remove any issues
- Make adjustments based on feedback

## Phase 6: Issue Creation in Linear

### Step 6.1: Create Parent Issues First
For issues that have child issues, create them first to establish the parent-child relationships.

### Step 6.2: Create All Issues
For each approved issue, use the Linear MCP server to:
1. Create the issue with all specified attributes
2. Assign appropriate labels (Type and Agent)
3. Set parent issue relationship if applicable
4. Add the issue to the backlog (default status)

### Step 6.3: Capture Issue Metadata
Keep track of created issues for the webhook bridge:
- Issue Key (identifier)
- Issue ID
- Agent Label ID
- Type Label ID
- Parent Issue Key (if applicable)

## Phase 7: Webhook Bridge Setup

### Step 7.1: Choose Cloud Provider
Ask the user: "Which cloud provider would you like to use for the webhook bridge function?"

Options:
- Vercel
- AWS Lambda
- Netlify Functions
- Google Cloud Functions

### Step 7.2: Generate Webhook Function
Create a TypeScript function based on the selected provider that:

**Core functionality:**
1. Validates the Linear webhook signing secret
2. Processes issue status changes (Todo → In Progress)
3. Identifies the agent from the issue's labels
4. Determines the branching strategy based on parent relationships
5. Creates a GitHub repository dispatch event

**Required environment variables:**
- `LINEAR_WEBHOOK_SECRET`: Linear webhook signing secret
- `GITHUB_TOKEN`: GitHub personal access token with repo access
- `GITHUB_OWNER`: GitHub repository owner
- `GITHUB_REPO`: GitHub repository name
- `LINEAR_API_KEY`: Linear key to look up issue details
- `DEBUGGING`: Optional flag for verbose logging

**Branching logic:**
- Issues without parents: Clone from `main`, create branch `feature/<branch-name>`
- Issues with parents: Clone from parent's branch, create branch `feature/<parent-issue-key>/<branch-name>`

**Key functions to include:**
```typescript
function verifySignature(payload: string, signature: string, secret: string): boolean {
  const hmac = crypto.createHmac('sha256', secret);
  hmac.update(payload);
  const expectedSignature = hmac.digest('hex');
  return (signature == expectedSignature);
}

function shouldTriggerWorkflow(payload): boolean {
  // Only process issue updates
  if (payload.type !== 'Issue') return false;

  // Only process update actions
  if (!['update'].includes(payload.action)) return false;

  // Check if issue has agent label
  const agent = findAgentLabelInLabelIds(payload.data.labelIds);
  if (!agent) {
    if(DEBUGGING) console.log('No agent label found');
    return false;
  }

  // Only trigger when issue moves from "Todo" to "In Progress"
  return (
    payload.updatedFrom?.stateId == knownIssueStates.find(state => state.name === 'Todo')?.id
    && payload.data.state?.name === 'In Progress'
  );
}
```

**Repository dispatch event structure:**
```typescript
const event = {
  owner: process.env.GITHUB_OWNER,
  repo: process.env.GITHUB_REPO,
  event_type: 'linear-agent-delegated',
  client_payload: {
    issue_key: linearPayload.data.identifier,
    issue_title: linearPayload.data.title,
    issue_description: linearPayload.data.description || '',
    issue_url: linearPayload.data.url,
    agent_name: issueAgentname,
    branch_name: branchName,
    ref: parentBranchName
  }
};

const octokit = new Octokit({ auth: process.env.GITHUB_TOKEN });
await octokit.repos.createDispatchEvent(event);
```

### Step 7.3: Create Label Reference Map
Generate a JSON mapping file that the webhook bridge will use. The file should contain:
- Label IDs to names
- Status IDs to names

The label ids and status ids can be found in the Linear MCP server.

```json
{
  "labelIdToName": {
    "label-id-1": "frink-principal-engineer",
    "label-id-2": "marge-ux-designer",
    "label-id-3": "Feature",
    "label-id-4": "Bug"
  },
  "statusIdToName": {
    "status-id-1": "Backlog",
    "status-id-2": "Todo",
    "status-id-3": "In Progress",
    "status-id-4": "Review",
    "status-id-5": "Done",
    "status-id-6": "Canceled"
  }
}
```

### Step 7.4: Deployment Instructions
Provide step-by-step deployment instructions based on the chosen cloud provider:
- How to deploy the function
- How to configure environment variables
- How to get the webhook URL

### Step 7.5: Configure Linear Webhook
Provide instructions to configure the webhook in Linear:
1. Navigate to Team Settings → Integrations → Webhooks
2. Create a new webhook with:
   - URL: [Function deployment URL]
   - Events: Issue updates only
   - Signing secret: Generate and save to env vars
3. Test the webhook

## Phase 8: GitHub Workflow Setup

### Step 8.1: Create Workflow File
Generate a GitHub Actions workflow file that listens for the dispatch events sent by the webhook bridge.

**File location**: `.github/workflows/linear-agent-delegation.yml`

**Workflow structure:**

```yaml
name: Linear Agent Delegation

# Triggered when Linear webhook sends issue delegation event
on:
  repository_dispatch:
    types: [linear-agent-delegated]

  # Manual trigger for testing
  workflow_dispatch:
    inputs:
      issue_key:
        description: 'Linear issue ID (e.g., KIS-15)'
        required: true
        type: string
      branch_name:
        description: 'Branch name'
        required: true
        type: string
      issue_title:
        description: 'Issue title'
        required: true
        type: string
      issue_description:
        description: 'Issue description'
        required: false
        type: string
      delegate:
        description: 'Agent delegate name (e.g., frink-principal-engineer)'
        required: true
        type: string

jobs:
  orchestrate-feature-generation:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.client_payload.ref }}
          fetch-depth: 0

      - name: Extract issue details
        id: issue
        run: |
          # Extract from webhook payload or manual input
          if [ "${{ github.event_name }}" = "repository_dispatch" ]; then
            echo "issue_key=${{ github.event.client_payload.issue_key }}" >> $GITHUB_OUTPUT
            echo "title=${{ github.event.client_payload.issue_title }}" >> $GITHUB_OUTPUT
            echo "description=${{ github.event.client_payload.issue_description }}" >> $GITHUB_OUTPUT
            echo "url=${{ github.event.client_payload.issue_url }}" >> $GITHUB_OUTPUT
            echo "delegate=${{ github.event.client_payload.agent_name }}" >> $GITHUB_OUTPUT
            echo "branch_name=${{ github.event.client_payload.branch_name }}" >> $GITHUB_OUTPUT
          else
            echo "issue_key=${{ inputs.issue_key }}" >> $GITHUB_OUTPUT
            echo "title=${{ inputs.issue_title }}" >> $GITHUB_OUTPUT
            echo "description=${{ inputs.issue_description }}" >> $GITHUB_OUTPUT
            echo "url=https://linear.app/kisasa/issue/${{ inputs.issue_key }}" >> $GITHUB_OUTPUT
            echo "delegate=${{ inputs.delegate }}" >> $GITHUB_OUTPUT
            echo "branch_name=${{ inputs.branch_name }}" >> $GITHUB_OUTPUT
          fi

      - name: Extract delegate agent info
        if: success()
        id: agent
        run: |
          DELEGATE="${{ steps.issue.outputs.delegate }}"

          # Use delegate name directly as subagent_type
          echo "subagent_type=$DELEGATE" >> $GITHUB_OUTPUT

          # Convert delegate name to display name (capitalize words, remove hyphens)
          DISPLAY_NAME=$(echo "$DELEGATE" | sed 's/-/ /g' | awk '{for(i=1;i<=NF;i++) $i=toupper(substr($i,1,1)) tolower(substr($i,2))}1')
          echo "display_name=$DISPLAY_NAME" >> $GITHUB_OUTPUT

          # Create email from delegate name
          echo "email=${DELEGATE}@kisasa.io" >> $GITHUB_OUTPUT

      - name: Create feature branch or use existing
        if: success()
        run: |
          git config user.name "${{ steps.agent.outputs.display_name }} (Claude Code)"
          git config user.email "${{ steps.agent.outputs.email }}"
          git checkout "${{ steps.issue.outputs.branch_name }}" || git checkout -b "${{ steps.issue.outputs.branch_name }}"

      - name: Create MCP Config
        id: claude_mcp_config
        run: |
          cat > mcp-config.json << 'EOF'
          {
            "mcpServers": {
              "linear": {
                "type": "http",
                "url": "https://mcp.linear.app/mcp"
              }
            }
          }
          EOF

      - name: Create Claude settings
        id: claude_settings
        run: |
          cat > claude-settings.json << 'EOF'
          {
            "env": {
              "DEBUG": "false"
            },
            "permissions": {
              "allow": ["Bash", "Read", "Write"],
              "deny": ["WebFetch"]
            },
            "hooks": {
              "PreToolUse": [{
                "matcher": "Bash",
                "hooks": [{
                  "type": "command",
                  "command": "echo Running bash command..."
                }]
              }]
            }
          }
          EOF

      - name: Set up Claude Code
        if: success()
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          settings: "claude-settings.json"
          claude_args: |
            --mcp-config mcp-config.json
            --max-turns 15
            --system-prompt "You are ${{ steps.issue.outputs.delegate }}, a Claude Code agent."
          prompt: |
            You have been delegated the following Linear issue.

            ## Linear Issue
            - **ID**: ${{ steps.issue.outputs.issue_key }}
            - **Title**: ${{ steps.issue.outputs.title }}
            - **URL**: ${{ steps.issue.outputs.url }}

            ## Description
            ${{ steps.issue.outputs.description }}

            ## Your Task

            Handle this issue according to your agent definition and expertise.
            Follow the instructions in your agent definition file (.claude/agents/${{ steps.agent.outputs.subagent_type }}.md).

            Generated code will be automatically committed and turned into a PR linked to this Linear issue.

      - name: Clean up Claude Code
        run: |
          rm mcp-config.json
          rm claude-settings.json

      - name: Commit generated code
        run: |
          git add .

          # Check if there are changes to commit
          if git diff --staged --quiet; then
            echo "No changes to commit"
            exit 0
          fi

          git commit -m "feat: ${{ steps.issue.outputs.title }}

          Implemented by ${{ steps.agent.outputs.display_name }} (Claude Code Agent)

          Linear Issue: ${{ steps.issue.outputs.issue_key }}

          🤖 Generated with Claude Code

          Co-Authored-By: Claude <noreply@anthropic.com>"

          git push origin "${{ steps.issue.outputs.branch_name }}"

      - name: Create Pull Request
        if: success()
        id: pr
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PR_URL=$(gh pr create \
            --title "feat: ${{ steps.issue.outputs.title }}" \
            --body "$(cat <<'EOF'
          ## Summary
          Implementation for Linear issue [${{ steps.issue.outputs.issue_key }}](${{ steps.issue.outputs.url }})

          **Handled by**: ${{ steps.agent.outputs.display_name }} (Claude Code Agent)

          ## Issue Description
          ${{ steps.issue.outputs.description }}

          ## Next Steps
          - [ ] Review code/analysis
          - [ ] Test changes (if applicable)
          - [ ] Update documentation
          - [ ] Merge when ready

          ---

          🤖 Generated with [Claude Code](https://claude.com/claude-code) by **${{ steps.agent.outputs.display_name }}**

          Linear will automatically sync this PR status back to issue ${{ steps.issue.outputs.issue_key }}
          EOF
          )" \
            --base "${{ github.event.client_payload.ref }}" \
            --head "${{ steps.issue.outputs.branch_name }}" \
            --label "agent-generated" \
            --label "needs-human-review")

          echo "url=${PR_URL}" >> $GITHUB_OUTPUT

      - name: Report success
        if: success()
        run: |
          echo "✅ Feature generation complete!"
          echo "PR created: ${{ steps.pr.outputs.url }}"
          echo "Linear will automatically link and sync this PR to issue ${{ steps.issue.outputs.issue_key }}"

      - name: Report failure
        if: failure()
        run: |
          echo "❌ Feature generation failed"
          echo "Workflow run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          echo "Check logs and retry manually"
```

### Step 8.2: Workflow Configuration Notes

**Key aspects of the workflow:**

1. **Trigger Events**:
   - `repository_dispatch`: Triggered by webhook bridge when Linear issue moves to "In Progress"
   - `workflow_dispatch`: Manual trigger for testing with input parameters

2. **Agent Identification**:
   - Extracts agent name from dispatch payload
   - Converts agent name to display name for commits
   - Creates agent-specific email address

3. **Branching**:
   - Checks out from the ref provided by webhook (handles parent branching)
   - Creates or checks out the target branch

4. **Claude Code Integration**:
   - Creates temporary MCP config for Linear access
   - Creates Claude settings with appropriate permissions
   - Passes agent definition and issue details to Claude Code
   - Agent follows instructions from `.claude/agents/{agent-name}.md`

5. **Automated PR Creation**:
   - Commits changes with proper attribution
   - Creates PR with Linear issue link
   - Labels PR for review
   - Linear automatically syncs PR status back to issue

### Step 8.3: Required GitHub Secrets

Inform the user they need to configure these GitHub repository secrets:
- `ANTHROPIC_API_KEY`: Claude API key for Claude Code actions
- `GITHUB_TOKEN`: Automatically provided by GitHub Actions (no setup needed)

### Step 8.4: Agent Definition Files

Remind the user that each agent must have a definition file at:
`.claude/agents/{agent-name}.md`

These files should contain:
- Agent's role and expertise
- Instructions for handling delegated issues
- Expected deliverables
- Quality standards

### Step 8.5: Test the Workflow

Provide instructions for testing:
1. **Manual test**: Use workflow_dispatch with sample issue data
2. **Integration test**: Move a test issue from Todo → In Progress in Linear
3. **Verify**: Check that workflow runs, creates branch, and opens PR

## Phase 9: Documentation & Summary

### Step 9.1: Create Backlog Summary
Generate a markdown document with:
- Project overview
- Total issues created (by type, by agent)
- Team configuration details
- Label groups and labels
- Workflow statuses
- Branching strategy explanation
- Webhook bridge information
- GitHub workflow configuration

**Filename**: `linear-backlog-summary-[YYYY-MM-DD-HHMM].md`
**Location**: Same directory as meeting notes ($2)

### Step 9.2: Display Summary
Show the user:
1. Total issues created
2. Linear team URL
3. Location of webhook bridge function
4. Location of label reference map
5. Location of GitHub workflow file
6. Required GitHub secrets
7. Next steps for deployment

### Step 9.3: Verify Completion
Ask the user if:
- They want to review any specific issues in Linear
- They need help with webhook deployment
- They want to make any adjustments to the backlog

## Important Notes

- **Agent Context**: The agents specified should already understand the work scope..
- **Linear MCP Access**: Full permissions required for team, label, status, and issue management
- **Webhook Bridge**: The bridge is critical for the automated workflow. Agent labels trigger agent engagement when issues move to "In Progress"
- **Branching Strategy**: Parent-child issue relationships determine branching structure for organized feature development
- **Label IDs**: The webhook bridge needs a reference map of label IDs to names for processing

## Graceful Handling

If at any point:
- An agent is not available: Notify user and continue with remaining agents
- Linear MCP is unavailable: Stop and ask user to configure Linear integration
- Meeting notes are incomplete: Ask user for clarification
- User rejects proposed backlog: Iterate with agents to refine

Throughout this process, maintain clear communication with the user about progress, decisions, and any issues encountered.
