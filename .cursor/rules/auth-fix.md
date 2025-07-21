Incident Report: Authentication Flow Failure and Resolution
Date: July 15, 2025
Project: AI Therapy Chatbot
1. Summary of Failure
The primary goal of integrating a Supabase backend was to provide user authentication and persistent session memory. After implementing the initial authentication components, a critical failure was identified: users were not automatically redirected to the main application (/) after a successful sign-in or sign-up attempt from the /login page.
This failure completely blocked user access to the core application, rendering all subsequent features untestable and making the application unusable. The project's success was critically impacted.
2. Initial Symptoms & Misleading Clues
The initial symptoms were confusing and pointed in several directions:
Symptom 1: User authenticates on the /login page but remains on the same page. No visual feedback of success or failure is provided.
Symptom 2: After the failed redirect, if the user manually navigated to the application root (/), they were granted access.
Symptom 3: The middleware.ts logs in the terminal confirmed that for manual navigation to /, an active session was correctly identified.
Symptom 4: The Supabase Auth logs confirmed that Login and /token exchange requests were being processed successfully, without 4xx or 5xx errors.
These clues collectively proved that Supabase was successfully authenticating the user and setting the session cookie in the browser, but the client-side application was failing to react to this state change and perform the necessary redirect.
3. Root Cause Analysis: The Faulty Abstraction
The investigation systematically eliminated potential causes:
Environment Variables: Initially suspected, this was corrected by adding the required NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY, and NEXT_PUBLIC_SITE_URL variables. This was a necessary fix but did not resolve the core redirect issue.
Supabase URL Configuration: We verified that the Supabase project's "Redirect URLs" were correctly configured to include http://localhost:3000/auth/callback, which eliminated the 422: invalid flow state error. This proved the Supabase backend was configured correctly.
Client-Side Redirect Logic: A manual redirect was added to the Auth.tsx component using a useEffect hook listening to onAuthStateChange. This also failed to trigger, which was the definitive clue.
The final conclusion was that the @supabase/auth-ui-react component, the pre-built "black box" UI, was the source of the failure. For reasons specific to this project's environment, it was not correctly invoking the onAuthStateChange event listener in a timely or reliable manner after a successful password-based login. This prevented both its own internal redirect logic and our manual useEffect logic from executing. The component provided convenience at the cost of transparency and control, and when it failed, it offered no clear path to a solution.
4. Resolution Steps: Eliminating the Black Box
The only path to guaranteed success was to discard the faulty tool and build a direct, transparent replacement.
Decommission @supabase/auth-ui-react: The decision was made to completely remove the <Auth> component from @supabase/auth-ui-react from our AuthForm component.
Implement a Manual Form: A new form was built from scratch in Auth.tsx using standard HTML <input> fields for email and password, and two distinct <button> elements for "Sign In" and "Sign Up".
Direct API Calls:
The "Sign In" button's onClick handler now directly calls the supabase.auth.signInWithPassword() function.
The "Sign Up" button's onClick handler now directly calls the supabase.auth.signUp() function.
Forceful, Imperative Redirect: Instead of waiting for an event listener, the redirect is now commanded imperatively. Immediately following a successful await on the signInWithPassword or signUp call, the next line of code is router.push('/'). This ensures that if the authentication succeeds, the redirect is guaranteed to execute.
Transparent Error Handling: The error object returned from the Supabase functions is now captured in a state variable and displayed directly to the user on the page. This eliminates silent failures and provides immediate, actionable feedback.
5. Outcome
Upon implementing the manual form, the authentication flow immediately began functioning as expected. Users are now correctly and reliably redirected to the main application (/) after a successful sign-in or sign-up.
The core lesson from this incident is the trade-off between convenience and control. While pre-built abstractions like @supabase/auth-ui-react can accelerate development, they can also introduce opaque failure modes. When faced with such a failure, the most effective solution for the project's success was to revert to a more fundamental, directly controlled implementation. The authentication roadblock is now resolved, and testing of core application features can commence.


---


We fixed the single biggest problem. The authentication is solid. The project is no longer blocked.
What's next is what we were supposed to be doing before we discovered the auth system was broken: Testing the core functionality of the therapy chatbot.
We need to verify that the app can:
Create a session in the database.
Save the conversation transcript to that session.
Save the long-term memory summary to that session.
Recall that summary in a new session.
It's time to execute the test plan.
ACTION: Execute Test Case 2
This test will prove that the most fundamental part of the persistence layer is working.
Your Tools:
Browser Tab 1: Your application running at http://localhost:3000.
Browser Tab 2: Your Supabase project dashboard, with the Table Editor open to the sessions table.
The Test:
Preparation:
Log in to your application. You should land on the main page (/). The "Start Interview" button should be visible.
Go to your Supabase sessions table viewer. For a clean test, delete any old rows from previous failed attempts so the table is empty.
Action 1: Start a Conversation.
In your app, click the "Start Interview" button.
Immediately switch to your Supabase tab and refresh the sessions table view.
Verification 1: Session Creation.
Did a new row appear in the sessions table?
Does the user_id in that new row match the ID of the user you are logged in as? (You can see your user's ID in the users table).
Action 2: Have a Short Conversation.
Go back to your app. Record an answer to the first question.
Let the bot respond.
Record a second answer.
After the bot gives its second response, wait 5 seconds.
Verification 2: Transcript Persistence.
Go back to your Supabase tab and refresh the sessions table view again.
Look at the row for your current session. Find the column named full_transcript_json.
**Is is written.
What's next is proving the application actually works.
We built the database for one reason: to give the chatbot memory. Now you have to test it. We will now execute the test plan that was blocked before.
ACTION: Test Session Creation and Saving (Test Case 2)
This is the most important test. It verifies that a conversation is actually saved to the user's account in the database.
Your Setup:
Tab 1 - The Application: Have your app running at http://localhost:3000. You should be logged in and seeing the main chat interface with the "Start Interview" button.
Tab 2 - The Database: Have your Supabase project dashboard open to the Table Editor for the sessions table. This is your "eye in the sky."
The Test Procedure:
Check the Database: Look at your sessions table in Supabase. It might have old, failed rows from previous tests. Delete any existing rows so you are starting with a completely empty table. This ensures a clean test.
Start a Conversation: In your application (Tab 1), click the "Start Interview" button.
VERIFY (Database): Immediately switch to your Supabase tab (Tab 2) and refresh the table view.
Expected Outcome: A brand new row must appear in the sessions table. The user_id in that row must match the ID of the user you are logged in as. This proves the /api/session/create route is working.
Have a Short Conversation: Go back to the app. Record 2 or 3 turns (you speak, the bot replies, you speak again).
VERIFY (Database): After your last turn, wait about 5 seconds. Go back to the Supabase tab and refresh the table view again.
Expected Outcome: Look at the row for the session you just started. Find the column named full_transcript_json. It should now be filled with the text of your conversation. This proves the debounced save in page.tsx and the /api/session/update route are working.
Report back. Does a new row appear? Does the full_transcript_json column get populated after you talk?