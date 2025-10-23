---
description: Ideate with one or more agents about a topic using thinking mode
argument-hint: [agent1,agent2,...] [topic] [meeting-notes-path]
---

You are facilitating a strategic team meeting about the following topic.

**Agents participating**: $1 (comma-delimited list of agent perspectives to adopt) (required)
**Meeting Topic**: $2 (required)
**Meeting Notes Path**: $3 (optional - defaults to `meeting-notes`)

## Meeting Preparation

1. **Check for existing meeting series**: If a meeting notes path is provided, read any existing notes from that location to check if this is a follow-up meeting or continuation of previous discussions.

2. **Establish meeting context**: If previous meetings exist, briefly summarize key decisions and action items from prior sessions to provide continuity.

3. **Set agenda**: Before starting, present a proposed agenda with 3-5 key discussion points based on the topic. Ask the user if they want to modify the agenda or add specific questions to address.

## Meeting Conduct

Use extended thinking mode to facilitate a structured, interactive meeting:

### Phase 1: Opening & Agenda Confirmation (2 min)
- State meeting purpose and expected outcomes
- Confirm agenda items with user
- Set ground rules for discussion

### Phase 2: Round-Robin Discussion (per agenda item)
For each agenda item:
1. **Introduce the topic**: Frame the discussion point clearly
2. **Agent perspectives**: Have each agent contribute their viewpoint in turn
   - Technical considerations (if applicable)
   - Business implications (if applicable)
   - User experience impact (if applicable)
   - Risk factors and concerns
3. **Track questions**: Keep a "Questions Raised" list for items needing follow-up
4. **Identify disagreements**: Note areas where agents have conflicting views
5. **User input opportunities**: After each round, pause and ask if the user has questions or input

### Phase 3: Question Resolution
- Address questions raised during discussion
- Have relevant agents provide answers
- Document unresolved questions for follow-up

### Phase 4: Decision Making
For each decision point:
- Present options with pros/cons
- Show agent recommendations with rationale
- Ask user for final decision when needed
- Document decisions made and reasoning

### Phase 5: Action Planning
- Synthesize discussion into concrete action items
- Assign responsibility to agents/roles (even if simulated)
- Suggest realistic timeframes
- Prioritize actions (Critical, High, Medium, Low)
- Identify dependencies and prerequisites

### Phase 6: Meeting Wrap-up
- Summarize key decisions and outcomes
- Confirm next steps
- Ask if a follow-up meeting is needed

## Meeting Note Structure

Track the following throughout the discussion:

**Decision Log**
- Decisions made, who recommended them, rationale, and user confirmation

**Questions & Answers**
- Questions raised during meeting
- Answers provided
- Unresolved items for follow-up

**Parking Lot**
- Topics that came up but are out of scope
- Items to discuss in future meetings

**Risk Register**
- Concerns raised by any agent
- Severity assessment
- Mitigation strategies proposed

**Action Items**
- Specific task description
- Assigned to (agent/role)
- Priority level
- Estimated timeframe
- Dependencies
- Success criteria

**Areas of Consensus**
- Topics where all agents agreed

**Areas of Disagreement**
- Topics where agents had conflicting views
- Each perspective documented
- Resolution approach or decision to revisit

## Output Requirements

### Display to Screen
Provide a meeting summary including:
1. Any unresolved questions requiring follow-up
2. Whether a follow-up meeting is recommended
3. Location of detailed meeting notes document

### Save to Markdown

**Filename**: `meeting-notes-[YYYY-MM-DD-HHMM].md`
**Location**:
- Default: `meeting-notes/`
- Custom: Use provided path from $3

**Document Structure** (if new meeting):
```
# Meeting Notes: [Topic]

## Meeting Metadata
- Date & Time: [YYYY-MM-DD HH:MM]
- Meeting ID: [Generate unique ID]
- Participants: [Agent names/roles]
- Meeting Type: [Brainstorm/Decision-Making/Status Update/Planning]
- Duration: [Estimated discussion time]

## Agenda
[List of topics discussed]

## Executive Summary
[2-3 paragraphs summarizing outcomes]

## Detailed Discussion
[Full meeting conversation organized by agenda item]

## Decision Log
[Table format with Decision, Recommended By, Rationale, Status]

## Questions & Answers
[Q&A pairs with notation for unresolved items]

## Action Items
[Table with Task, Owner, Priority, Due Date, Dependencies, Status]

## Risk Register
[Table with Risk, Severity, Impact, Mitigation Strategy]

## Parking Lot
[List of deferred topics]

## Areas of Consensus & Disagreement
[Summary of agreement and conflict points]

## Follow-up Required
- [ ] Unresolved questions to research
- [ ] Topics requiring another meeting
- [ ] Decisions pending user input

## Next Meeting
[Suggested topics and timing if needed]
```

**Document Structure** (if follow-up meeting):
Append to existing document:
```
---

# Follow-up Meeting: [Date]

## Link to Previous Meeting
Previous Meeting ID: [ID]
Previous Action Item Status: [Brief summary]

[Then follow same structure as new meeting]
```

**Formatting**:
- Heading 1 for main sections
- Heading 2 for subsections
- Tables for structured data (decisions, actions, risks)
- Bold for emphasis and labels
- Bullet points for lists
- Numbered lists for sequences
- Color coding for priority (Red=Critical, Orange=High, Yellow=Medium, Green=Low)
- Checkboxes for action items

## Interactive Guidelines

Throughout the meeting:
- **Pause for user input** after each major agenda item
- **Ask clarifying questions** when agents have conflicting recommendations
- **Invite user to add topics** to parking lot or next meeting agenda
- **Confirm major decisions** before documenting them as final
- **Offer to dive deeper** on any topic that needs more exploration
- **At the end of the meeting**, ask if a follow-up meeting is needed and if the notes should be converted Word doc format.

Remember: This is a collaborative meeting simulation. The goal is to help the user think through complex topics with multiple expert perspectives while maintaining clear documentation of the discussion, decisions, and next steps.