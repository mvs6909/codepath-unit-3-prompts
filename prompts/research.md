# Research Codebase


You are tasked with conducting comprehensive research across the codebase to answer user questions about a specific area/functionality/feature of the codebase.


## Initial Setup:


When this command is invoked, respond with:
```
I'm ready to research the codebase. Please provide your research question or area of interest, and I'll analyze it thoroughly by exploring relevant components and connections.
```


Then wait for the user's research query.


## Steps to follow after receiving the research query:


1. **Read any directly mentioned files first:**
  - If the user mentions specific files (tickets, docs, JSON), read them FULLY first
  - **IMPORTANT**: Use the Read tool WITHOUT limit/offset parameters to read entire files
  - **CRITICAL**: Read these files yourself in the main context before spawning any sub-tasks
  - This ensures you have full context before decomposing the research


2. **Analyze and decompose the research question:**
  - Break down the user's query into composable research areas
  - Take time to ultrathink about the underlying patterns, connections, and architectural implications the user might be seeking
  - Identify specific components, patterns, or concepts to investigate
  - Create a research plan to track all subtasks
  - Consider which directories, files, or architectural patterns are relevant


3. **Spawn parallel sub-agent tasks for comprehensive research:**
  - Create multiple Task agents to research different aspects concurrently


4. **Wait for all sub-agents to complete and synthesize findings:**
  - IMPORTANT: Wait for ALL sub-agent tasks to complete before proceeding
  - Compile all sub-agent results (both codebase and thoughts findings)
  - Prioritize live codebase findings as primary source of truth
  - Use `memory-bank/memory` or `memory-bank/outputs` findings as supplementary historical context
  - Connect findings across different components
  - Include specific file paths and line numbers for reference
  - Highlight patterns, connections, and architectural decisions
  - Answer the user's specific questions with concrete evidence


5. **Gather metadata for the research document:**
  - generate all relevant metadata
  - Filename: `memory-bank/memory/research/YYYY-MM-DD-description.md`
    - Format: `YYYY-MM-DD-ENG-XXXX-description.md` where:
      - YYYY-MM-DD is today's date
      - description is a brief kebab-case description of the research topic
    - Examples:
      - With description: `2025-01-08-add-title.md`


6. **Generate research document:**
  - Use the metadata gathered in step 4
  - Structure the document with YAML frontmatter followed by content:
    ```markdown
    ---
    topic: "[User's Question/Topic]"
    tags: [research, codebase, relevant-component-names]
    ---


   # Research: [User's Question/Topic]


    **Date**: [Current date and time with timezone from step 4]


   ## Research Question
    [Original user query]


   ## Summary
    [High-level findings answering the user's question]


   ## Detailed Findings


   ### [Component/Area 1]
    - Finding with reference ([file.ext:line](link))
    - Connection to other components
    - Implementation details


   ### [Component/Area 2]
    ...


   ## Code References
    - `path/to/file.py:123` - Description of what's there
    - `another/file.ts:45-67` - Description of the code block


   ## Architecture Insights
    [Patterns, conventions, and design decisions discovered]


   ## Historical Context (from thoughts/)
    [Relevant insights from thoughts/ directory with references]
    - `thoughts/shared/something.md` - Historical decision about X
    - `thoughts/local/notes.md` - Past exploration of Y
    Note: Paths exclude "searchable/" even if found there


   ## Related Research
    [Links to other research documents in thoughts/shared/research/]


   ## Open Questions
    [Any areas that need further investigation]
    ```




7. **Sync and present findings:**
  - Present a concise summary of findings to the user
  - Include key file references for easy navigation
  - Ask if they have follow-up questions or need clarification


9. **Handle follow-up questions:**
  - If the user has follow-up questions, append to the same research document
  - Update the frontmatter fields `last_updated` and `last_updated_by` to reflect the update
  - Add `last_updated_note: "Added follow-up research for [brief description]"` to frontmatter
  - Add a new section: `## Follow-up Research [timestamp]`
  - Spawn new sub-agents as needed for additional investigation
  - Continue updating the document and syncing


## Important notes:
- Always use parallel Task agents to maximize efficiency and minimize context usage
- Always run fresh codebase research - never rely solely on existing research documents
- The memory-bank/memory directory provides historical context to supplement live findings
- Focus on finding concrete file paths and line numbers for developer reference
- Research documents should be self-contained with all necessary context
- Each sub-agent prompt should be specific and focused on read-only operations
- Consider cross-component connections and architectural patterns
- Include temporal context (when the research was conducted)
- Keep the main agent focused on synthesis, not deep file reading
- Encourage sub-agents to find examples and usage patterns, not just definitions
- **File reading**: Always read mentioned files FULLY (no limit/offset) before spawning sub-tasks
- **Critical ordering**: Follow the numbered steps exactly
 - ALWAYS read mentioned files first before spawning sub-tasks (step 1)
 - ALWAYS wait for all sub-agents to complete before synthesizing (step 4)
 - ALWAYS gather metadata before writing the document (step 5 before step 6)
 - NEVER write the research document with placeholder values
- **Frontmatter consistency**:
 - Always include frontmatter at the beginning of research documents
 - Keep frontmatter fields consistent across all research documents
 - Update frontmatter when adding follow-up research
 - Tags should be relevant to the research topic and components studied