# How to Restore the Multi-Step System

This document describes the original multi-step conversation architecture. The system has been temporarily simplified to use a single prompt. To restore the turn-by-turn functionality, follow these steps. All changes are in `ielts-speaking-mock-test/app/page.tsx`.

1.  **Restore State Variables:**
    *   Re-introduce the `turn` state: `const [turn, setTurn] = useState<number>(0);`
    *   Re-introduce the `activePromptList` state: `const [activePromptList, setActivePromptList] = useState<PromptType[]>([]);`

2.  **Update `handleMicPress` function:**
    *   Inside `handleMicPress`, restore the logic to select a topic and initialize the prompt list.
    ```typescript
    // ... existing reset logic
    setTurn(0);
    const topics = Object.keys(TOPIC_PROMPT_SETS);
    const randomTopic = topics[Math.floor(Math.random() * topics.length)];
    const prompts = TOPIC_PROMPT_SETS[randomTopic];
    setActivePromptList(prompts);
    // ...
    ```

3.  **Update the `useEffect` for `callAskApi`:**
    *   In the `useEffect` hook that runs when `appState` is `'asking'`, find the call to `callAskApi`.
    *   Modify it to use the `turn` state to select the current prompt from `activePromptList`.
    ```typescript
    // was: callAskApi({ prompt: SINGLE_PROMPT, history: chatHistoryRef.current });
    // should be:
    callAskApi({
      prompt: activePromptList[turn],
      history: chatHistoryRef.current,
    });
    ```

4.  **Update the Audio `onEnded` Handler:**
    *   In the `onEnded` handler for the examiner's audio playback, restore the logic to increment the turn counter.
    *   This handler should also check if the interview is over by comparing `turn` to the `activePromptList` length.
    ```typescript
    // Inside the onEnded function for the audio element
    if (turn + 1 >= activePromptList.length) {
      setAppState("finished");
    } else {
      setTurn(prev => prev + 1);
      setAppState("idle"); // Or 'recording' depending on original flow
    }
    ```

5. **Re-introduce `TOPIC_PROMPT_SETS` import:**
    * Make sure `TOPIC_PROMPT_SETS` is imported from `lib/interviewPrompts.ts`.

After following these steps, the application will revert to its original, multi-stage interview flow.

---

# How Multi-Stage Prompts Work in the IELTS Speaking Mock Test

This document outlines the architecture and control flow that enables the multi-stage (turn-by-turn) conversation in this application.

## Core Components

The system relies on the interaction between three key parts:

1.  **Prompt Lists (`ielts-speaking-mock-test/lib/interviewPrompts.ts`):** This file contains the content of the interview. For each topic (e.g., "Hometown," "Computers"), there is an array of `PromptType` objects. The order of objects in this array defines the exact sequence of questions the examiner will ask.

2.  **Frontend State Machine (`ielts-speaking-mock-test/app/page.tsx`):** The main page component acts as the orchestrator. It manages the interview's state using two crucial state variables:
    *   `appState`: A state machine that tracks the current status of the application (e.g., `idle`, `recording`, `asking`, `speaking`, `playing`).
    *   `turn`: A number that serves as an index to pick the correct prompt from the active prompt list for the current stage of the conversation.

3.  **API Endpoints (`ielts-speaking-mock-test/app/api/...`):** A set of serverless functions that handle specific tasks:
    *   `/api/ask`: Takes a prompt and chat history, queries the OpenAI LLM, and returns the examiner's next question.
    *   `/api/transcribe`: Converts the user's recorded audio into text.
    *   `/api/speak`: Converts the examiner's text question into speech audio.

## The Conversation Flow (Step-by-Step)

The entire interview is a loop managed by the frontend, progressing turn by turn.

### 1. Initialization

1.  **User Action:** The user clicks the microphone button to start a new interview.
2.  **State Setup:** In `page.tsx`, the `handleMicPress` function resets the application state, sets `turn` to `0`, and randomly selects a topic.
3.  **Prompt Loading:** The array of prompts for the selected topic is loaded from `TOPIC_PROMPT_SETS` into the `activePromptList` state.
4.  **First Action:** The app immediately transitions to the `recording` state to capture the user's response to the initial (unspoken) prompt, which is typically "Are you ready?".

### 2. A Single Turn in the Main Loop

This cycle repeats for each question from the examiner.

1.  **User Speaks & Stops Recording:**
    *   The user provides their answer and clicks the mic to stop.
    *   This triggers the `callTranscribeApi` function. The app state becomes `transcribing`.
    *   The user's speech is converted to text. The transcript is added to the `chatHistory`.
    *   The app state transitions to `asking`.

2.  **Examiner's Turn to "Think" and "Ask":**
    *   A `useEffect` hook in `page.tsx` detects that `appState` is now `asking`.
    *   It retrieves the current prompt from `activePromptList` using the `turn` number as the index (`activePromptList[turn]`).
    *   It calls `callAskApi`, sending the current prompt object and the entire chat history to the `/api/ask` endpoint.
    *   The LLM generates the next question based on the prompt's instructions.
    *   The returned question text is added to `chatHistory`.
    *   The app state transitions to `speaking`.

3.  **Examiner "Speaks":**
    *   The `useEffect` hook now sees the `appState` as `speaking`.
    *   It takes the latest examiner message from `chatHistory` and sends it to the `callSpeakApi` function.
    *   The `/api/speak` endpoint returns an audio file of the question.
    *   The audio begins to play, and the app state becomes `playing`.

4.  **Preparing for the Next Turn:**
    *   An `onEnded` event handler on the audio player triggers when the examiner's audio finishes.
    *   This handler increments the `turn` counter (`setTurn(prev => prev + 1)`).
    *   It sets the `appState` back to `idle`.

The application is now ready for the user to click the mic again and respond to the question they just heard, which starts the loop over for the next turn.

### 3. Conclusion of the Interview

*   The loop continues until the `turn` number exceeds the length of the `activePromptList`.
*   At this point, the `onEnded` handler sees that no prompts are left and sets the `appState` to `finished`.
*   This triggers the `callPipelineApi` to send the complete transcript for final evaluation, and the results are displayed.

This client-side, state-driven architecture allows for a controlled, sequential conversation, ensuring the interview follows the predefined structure while still allowing for dynamic interaction with the LLM at each step.
