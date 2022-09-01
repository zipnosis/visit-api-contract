# API Contract

The below establishes a contract for a general purpose API supporting "interview"-like workflows where the user is presented with a series of prompts.

## GET /interview/:id/action
Current state of the interview may always be retrieved by `GET /interview/:id/action`

Response body will always be a JSON object. Example:

```json
{
    "state_name": "new_user_welcome",
    "title": "Welcome, Stranger!",
    "content": [
        {
            "content_type": "paragraph",
            "content_key": "intro_paragraph",
            "display_text": "Please tell us a little about yourself to get started."
        },
        {
            "content_type": "free_text",
            "content_key": "first_name",
            "content_label": "First Name",
            "required": true,
            "max_length": 60
        },
        {
            "content_type": "radio",
            "content_key": "age_category",
            "content_label": "Age",
            "required": true,
            "options": [
                {
                    "option_label": "I am under age 18 and am completing this with my guardian.",
                    "option_value": "under_18",
                },
                {
                    "option_label": "I am age 18 or older.",
                    "option_value": "over_18",
                }
            ]
        }
    ],
    "actions": {
        "continue": {
            "action_label": "Continue",
        },
        "go_back": {
            "action_label": "Go Back",
        },
        "cancel_interview": {
            "action_label": "Cancel interview"
        }
    }
}
```

If the action is successful, it must return these keys (all keys are always returned unless noted otherwise):

- **`state_name`**: String, may be used as a screen identifier such as `"welcome"`
- **`title`**: String, human-readable slide title/heading, usage optional, will be translated into the current locale
- **`content`**: Array of objects. Each object if any will have these keys:

  - **`content_type`**: String
    - Must be one of `["paragraph", "html", "radio", "checkbox", "select", "free_text"]`.
    - If the content type is `"paragraph"`, the client should convey text to the user.
    - If the content type is `"html"`, the client should render the HTML. Security sanitization is strongly recommended by both the server and client.
    - If the content type is `"radio"`, `"checkbox"`, `"select"`, `"free_text"`, it represents an input field that should be presented to the user.
    - If more types need to be added, API developers should notify clients, treat it as a breaking change, and not release until all clients are ready to handle the new types.

  - **`content_key`**: String
    - Identifies the content item in the current view.
    - Included also for paragraph and HTML content (helps with front-end frameworks that require a unique identifier for repeat data.)
    - The value must be used as the key for providing key-value pairs in the `responses` portion of the next request.

  - **`content_label`**: String
    - Human-readable label, translated into the current locale.
    - Only present for user input content
    - May serve as the `<legend>` for a radio button `<fieldset>`.

  - **`display_text`**: String, human-readable and translated, only present for `paragraph` content.

  - **`display_html`**: String, human-readable and translated, only present for `html` content.

  - **`required`**: Boolean
    - Indicates whether a user response to this item is required in order to take the `"continue"` action.
    - Inputs are never required for other actions.

  - **`options`**: Array
    - always present for `"radio"`, `"checkbox"`, and `"select"`
    - Never present for `"label"` content
    - Possibly present for `"free_text"` to allow the client to provide the options as suggestions (TBD)
    - If present, always an array of one or more objects. Each object will have these keys:

      - **`option_name`**: String, identifier for this specific option.
      - **`option_label`**: String, human-readable label for this specific option, translated into the current locale
      - **`option_value`**: [Varies, String, Boolean, or Numeric]: value representing what should be returned in the next request if this option is selected by the user.  Within a content item, no two options will ever have the same value.
      - **`select_none`**: Boolean. Special field that is present only for checkbox content. If this is true, the UI should deselect all other checkboxes when this checkbox is selected, and it should deselect this checkbox if any other checkbox is selected (within the same content item)

  - **`actions`**: Object, with zero or more of the following keys, depending on current user state:

      - **`continue`**: Object.

        - This is usually the primary action for going forward in the interview.
        - It may be missing for end states where there is no option to continue.
        - For this action, it is implied that all input fields listed in `"content"` should be included in the request.

      - **`go_back`**: Object.
        - This allows the user to go back one step, effectively un-doing their previous action.
        - It may be missing when going back is not possible or not permitted.

      - **`cancel_interview`**: Object.
        - This fully ends and permanently cancels the current interview.
        - It may be missing if there is no option to cancel, for example after a data request has been submitted to others.

      - **`see_other_options`**: Object.
        - This shows the user their other options available at this moment other than continuing the interview.
        - It may be missing if there are no other options, or if we do not wish to highlight any other options in the current state.

      - If an action is listed, then it means that the action is currently supported and the server must accept it.

      - Each action must have these keys:
          - **`action_label`**: String.
            - Human-readable label for the action, translated into the current locale.
            - Note that, for example, the Continue action may not be labeled "Continue" in cases where another description may be more sensible.

## POST /interview/:id/action
- Actions may be taken via `POST /interview/:id/action`. When POSTing, request body JSON data is required. Example:

  ```json
  {
      "action_name": "continue",
      "responses": {
          "first_name": "Magdalena",
          "age_category": "over_18"
      }
  }
  ```

    Request JSON must include these keys:

    - **`action_name`**: String, required. Must be one of the items listed in the `"actions"` object from the GET or previous POST response.

    - **`responses`**: Object, required. Must include a key-value pair for each of the required inputs listed in the previous `content` response. Each key should be the name of a content item, and each value should be represent the user's input for that item.

- If the request is successful, the state of the interview should change, and the POST response data should be different than the previous response data but must be identical in structure to the GET response, allowing each response to be processed by the same client code again and again.

## Error handling
- 401: You are not authenticated.
- 404: The interview cannot be found or you do not have access to it.
- 422: The action cannot be completed and requires user correction.
- 500: The action cannot be completed due to a user-uncorrectable server error.

- If the response is an error 4xx or 5xx, the response will be an object that contains an array of objects:

  ```
  { errors: [{ reason: "reason_identifier", message: "Human readable text error reason" }] }
  ```
