Canonical Design Document: The "Terminal Summary" & Pipeline Architecture
Project: Therapy Chatbot
Version: 10.0 (Definitive Blueprint)
Status: Final, Approved
Author: Gemini Architect
1.0 Executive Summary
This document specifies the final, definitive architecture for the application's conversational memory and analysis system. The chosen design is the "Terminal Summary" model. This architecture makes a conscious trade-off, prioritizing maximum simplicity and performance during the live user session by deferring all computationally intensive processing until after a session is explicitly concluded.
The core principle is a strict separation between the live conversation and post-session processing. Upon conclusion, a single, non-blocking process generates a summary for future-session persistence, while a parallel process runs the full transcript through an analysis pipeline. A "Janitor" function ensures that even sessions ended improperly (e.g., via a browser crash) are correctly processed upon the user's next login, guaranteeing complete data integrity and memory persistence.
2.0 Core Architectural Principles
Live Session Purity: The live conversation loop is kept pure. It does not perform any real-time summarization.
User-Controlled Finalization: The end of a session is an explicit event, triggered by the user or a timer.
Post-Session Processing: All memory creation (summarization) and deep analysis (pipeline) happens after the session is complete.
Automated Cleanup ("The Janitor"): A safety-net function runs upon user login to find and process any "orphaned" sessions.
The Database is the Definitive Record: The full_transcript_json in the database is the immutable source of truth for all post-session processing.
3.0 Data Schemas & Type Definitions
3.1 Database Schema (sessions table)
This is the required SQL to create the primary data table in Supabase. The ON DELETE CASCADE ensures data integrity if a user is deleted.
Generated sql
CREATE TABLE public.sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now() NOT NULL,
  full_transcript_json JSONB,
  last_llm_context_summary TEXT,
  dap_note TEXT
);
Use code with caution.
SQL
3.2 Frontend Application State Machine (AppState type)
This type defines the valid states for the client application, enforcing a strict control flow.
Generated typescript
type AppState =
  | "idle"           // Waiting for user input
  | "recording"      // Microphone is active
  | "transcribing"   // Awaiting transcription
  | "asking"         // Awaiting AI response
  | "speaking"       // Playing AI audio response
  | "finalizing"     // User has clicked "End Session", waiting for sync
  | "housekeeping"   // "Janitor" is running on login
  | "showing_results"// Displaying pipeline feedback
  | "error";
Use code with caution.
TypeScript
4.0 API Endpoint Definitions
These are the formal contracts for all client-server interactions, including failure cases.
A. POST /api/ask
Responsibility: Generate the AI's next response.
Request Body: { "history": OpenAI.Chat.ChatCompletionMessageParam[] }
Success Response (200 OK): { "question": "string" }
Failure Response (500 Error): { "error": "string" }
B. POST /api/pipeline
Responsibility: Run the full user transcript against analysis prompts.
Request Body: { "transcript": "string" }
Success Response (200 OK): { "feedback": Array<{ criterion: string; feedback: string; }> }
Failure Response (400/500 Error): { "error": "string" }
C. POST /api/session/update-transcript
Responsibility: Force-save the complete transcript to the database.
Request Body: { "sessionId": "uuid", "fullTranscriptJson": ChatMessage[] }
Success Response (200 OK): { "success": true }
Failure Response (400/500 Error): { "error": "string" }
D. POST /api/session/finalize
Responsibility: Generate and save the summary for a completed session.
Request Body: { "sessionId": "uuid" }
Success Response (200 OK): { "success": true }
Failure Response (404/500 Error): { "error": "string" }
E. POST /api/session/cleanup
Responsibility: The "Janitor" function. Finds and processes all orphaned sessions for the user.
Request Body: None.
Success Response (200 OK): { "cleanedSessionCount": number }
Failure Response (500 Error): { "error": "string" }
F. GET /api/session/latest-summary
Responsibility: Fetch the last known summary for a user.
Request Body: None.
Success Response (200 OK): { "lastSummary": "string" | null }
Failure Response (404/500 Error): { "error": "string" }
5.0 Key Workflows & Implementation Snippets
This section details the logic for the most complex interactions.
5.1 Session Finalization (Clean Exit)
This workflow mitigates the transcript synchronization race condition.
Trigger: User clicks the "End Session" button.
Reference Implementation (handleEndSession in page.tsx):
Generated typescript
const handleEndSession = async () => {
  // 1. Prevent other actions and show loading UI.
  setAppState("finalizing");

  // 2. Force an immediate, final save of the complete transcript.
  await fetch('/api/session/update-transcript', { /* ...body... */ });

  // 3. With the DB synchronized, trigger background tasks.
  fetch('/api/session/finalize', { /* ...body... */ });

  // 4. Fetch pipeline results to show the user.
  const userTranscript = fullChatHistory.filter(m => m.role === 'user').map(m => m.content).join('\n');
  const pipelineResponse = await fetch('/api/pipeline', { /* ...body... */ });
  const feedbackData = await pipelineResponse.json();
  setFeedbackResults(feedbackData.feedback);
  
  // 5. Show results.
  setAppState("showing_results");
};
Use code with caution.
TypeScript
5.2 Session Recovery ("Janitor" on Login)
This workflow mitigates the risk of orphaned sessions from crashes or dirty exits.
Trigger: User successfully logs in.
Design Rationale: This server-side cleanup on the next login is chosen over client-side events like beforeunload because the latter are notoriously unreliable and not guaranteed to execute, especially on mobile browsers. This "Janitor" approach guarantees 100% of orphaned sessions are eventually processed.
Reference Implementation (Login Flow):
Generated typescript
const handleLoginSuccess = async () => {
  // 1. Enter the "housekeeping" state to provide user feedback.
  setAppState("housekeeping");

  // 2. Call the dedicated cleanup endpoint and wait for it to complete.
  await fetch('/api/session/cleanup', { method: 'POST' });

  // 3. Now that cleanup is done, fetch context for the new session.
  const response = await fetch('/api/session/latest-summary');
  const data = await response.json();
  if (data.lastSummary) {
    // Inject the summary into the initial context.
    setLlmContextMessages([
      { role: 'system', content: '...' },
      { role: 'assistant', content: `To provide context, here is a summary of our last conversation: ${data.lastSummary}` }
    ]);
  }

  // 4. Transition to the main application state.
  setAppState("idle");
};
Use code with caution.
TypeScript
6.0 Formula of Memory Creation
This architecture uses one simple formula, invoked only after a session is complete.
S_final = LLM_Summarize_Cheap(T_complete)
S_final: The final, canonical summary stored in last_llm_context_summary.
LLM_Summarize_Cheap(...): The call to gpt-4o-mini performed by /api/session/finalize.
T_complete: The sole input: the entire, immutable full_transcript_json.
7.0 Conclusion
This document provides a complete, unambiguous, and implementation-ready specification for the "Terminal Summary" architecture. It defines the data schemas, API contracts, and logical workflows required to build a system that is performant during live use, robust against common failure modes, and capable of the persistent, multi-session memory that is critical to the project's success. This is the definitive blueprint for implementation.