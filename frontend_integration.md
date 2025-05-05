# Frontend Integration Guide: Receiving Macro Updates from AIScreen Recorder Extension

This guide outlines the steps required **solely on the `aiscreen.io` frontend** to receive macro JSON data updates from the AIScreen Recorder browser extension.

## Overview

When a user stops recording in the AIScreen Recorder extension, the extension automatically saves the macro data and sends it to the `aiscreen.io` page via the extension's content script. The content script then dispatches a custom JavaScript event named `AIScreenMacroUpdated` on the `window` object.

The frontend needs to listen for this specific event to receive the data.

## Frontend Implementation Steps

1.  **Add Event Listener in Frontend JavaScript:**

    *   **Location:** In your main JavaScript file for `aiscreen.io` (e.g., `app.js`, `main.js`, or wherever your primary frontend logic/component initialization resides).
    *   **Code:** Add the following code to listen for the `AIScreenMacroUpdated` custom event dispatched by the extension's content script.

    ```javascript
    // In your website's main JS file (e.g., app.js, main.js)

    window.addEventListener('AIScreenMacroUpdated', (event) => {
      const updatedMacroData = event.detail; // The macro data is in event.detail
      console.log('AIScreen Frontend received macro update via CustomEvent:', updatedMacroData);

      // --- Your Logic Here ---
      // TODO: Implement the logic to handle the received macro data.
      // This might involve:
      //   - Storing the data (e.g., in component state, a global store like Redux/Vuex, or localStorage).
      //   - Updating the UI to reflect the new/updated macro.
      //   - Triggering data synchronization if needed.
      // Example:
      //   updateApplicationState(updatedMacroData);
      //   displayUpdatedMacro(updatedMacroData);
      // -----------------------

    });

    // Add a log to confirm the listener is set up on page load
    console.log("AIScreen Frontend listening for 'AIScreenMacroUpdated' event from extension.");
    ```

    *   **Data Structure:** The `event.detail` object will contain the macro data sent by the extension, likely structured similarly to this:
        ```json
        {
          "name": "NameOfMacro",
          "commands": [ 
            { "cmd": "open", "target": "https://example.com", "value": "" },
            { "cmd": "click", "target": "id=button", "value": "" }
            // ... other commands
          ],
          "meta": { 
            // ... other metadata like src, selectedIndex etc. if included
          }
        }
        ```

## Sending Messages from Frontend to Extension

To send data or commands from the `aiscreen.io` frontend **to** the AIScreen Recorder extension, you can use the standard `window.postMessage` method. The extension's content script (already configured) is listening for these messages and will relay them to the extension's background script/service worker.

1.  **Sending the Message:**

    *   **Code:** Use the following structure in your frontend JavaScript when you need to send a message to the extension:

    ```javascript
    // Example in your frontend code (e.g., inside a button click handler)

    function sendMessageToExtension(messageType, dataPayload) {
      console.log(`AIScreen Frontend: Sending message to extension:`, { type: messageType, payload: dataPayload });
      
      window.postMessage({
        direction: 'AISCREEN_TO_EXTENSION', // Required identifier for the content script
        type: messageType,                  // Your custom message type (e.g., 'GET_MACRO_LIST', 'START_RECORDING')
        payload: dataPayload                // The data you want to send (can be any JSON-serializable object)
      }, '*' // Or specify the target origin 'https://aiscreen.io' for better security
      );
    }

    // --- Example Usage ---
    // sendMessageToExtension('REQUEST_SYNC', { userId: '12345' });
    // sendMessageToExtension('LOAD_MACRO', { macroId: 'abc-def' });
    ```

    *   **Key Components:**
        *   `direction: 'AISCREEN_TO_EXTENSION'`: **Crucial**. This property tells the content script listener that this message is intended for the extension.
        *   `type`: A string you define to indicate the purpose of the message. The extension's background script will need to handle these types.
        *   `payload`: An object containing the data you want to send.
        *   `targetOrigin`: Using `'*'` is convenient but less secure. For production, specify the exact origin of your frontend (`'https://aiscreen.io'`).

2.  **Handling Responses (Optional):**

    *   The content script includes basic logic to potentially relay responses (or errors) back from the extension's background script using `window.postMessage` with `direction: 'EXTENSION_TO_AISCREEN'`.
    *   Your frontend can listen for these response messages if needed:

    ```javascript
    window.addEventListener('message', (event) => {
      // Basic security checks (ensure message is from the same window)
      if (event.source !== window) {
        return;
      }

      const responseData = event.data;

      // Check if it's a response from the extension
      if (responseData && responseData.direction === 'EXTENSION_TO_AISCREEN') {
        console.log('AIScreen Frontend: Received response from extension:', responseData);

        if (responseData.type === 'MESSAGE_RESPONSE') {
          // Handle the successful response payload (responseData.payload)
          // based on the original message type (responseData.originalType)
        } else if (responseData.type === 'MESSAGE_ERROR') {
          // Handle the error payload (responseData.payload)
          console.error(`Error related to ${responseData.originalType}:`, responseData.payload);
        }
      }
    });
    ```

3.  **Extension Background Script:**

    *   Remember that the extension's background script (`background.js`) needs to be updated to *listen* for the message types you send from the frontend (e.g., `'REQUEST_SYNC'`, `'LOAD_MACRO'`) via `chrome.runtime.onMessage.addListener` and implement the corresponding logic.

## Summary of Frontend Responsibilities (Sending)

*   Use `window.postMessage` to send messages.
*   Structure the message data with `{ direction: 'AISCREEN_TO_EXTENSION', type: 'YOUR_TYPE', payload: {...} }`.
*   Optionally, add a `window.addEventListener('message', ...)` to handle responses or errors sent back from the extension (which will have `direction: 'EXTENSION_TO_AISCREEN'`).
