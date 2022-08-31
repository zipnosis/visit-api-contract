## API Contract

Current state of the interview can always be retrieved by GET /interview

Response body will always be a JSON object.

If the action is successful, it will return these keys (all keys are always returned unless noted otherwise):

  state_name: always a string, can be used as a screen identifier such as 'welcome'

  title: String, human-readable slide title/heading, usage optional, will be translated into the current locale

  content: Array of objects. Each object if any will have these keys:

      type: String, may be any of ["label", "radio", "checkbox", "select", "free_text"]. 
            If the type is "label", the client merely needs to convey the label text to the user.
            If the type is "radio", "checkbox", or "select", it represents an input field that should be presented to the user.
            If more types need to be added, API developers should notify clients, treat it as a breaking change, and not release until all clients are ready to handle the new types.

      required: Boolean, indicating whether a user response to this item is required in order to take the "continue" action.
                Inputs are never required for other actions.

      content_name: String, identifier for the content item.
            This will be necessary for the client to provide key-value pairs in the request to continue.

      label: String, human-readable label for the input, translated into the current locale.

      options: Array, always present for "radio", "checkbox", and "select"
               Never present for "label" content
               Possibly present for "free_text" to allow the client to provide the options as suggestions (TBD)
               If present, always an array of one or more objects. Each object will have these keys:

          option_name: String, identifier for this specific option. Within a content item, no two options will ever have the same name.
          option_label: String, human-readable label for this specific option, translated into the current locale
          option_value: [Varies, String or Numeric]: value representing what should be returned in the next request if this option is selected by the user.
          select_none: Boolean. Special field that is present only for checkbox content. If this is true, the UI should deselect all other checkboxes when this checkbox is selected, and it should deselect this checkbox if any other checkbox is selected (within the same content item)

      actions: Object, with zero or more of the following keys, depending on current user state:

          continue: Object.
              This is usually the primary action for going forward in the interview.
              It will be missing for end states where there is no option to continue.
              For this action, it is implied that all input fields listed in "content" should be included in the request.

          go_back: Object.
              This allows the user to go back one step, effectively un-doing their previous action.
              It will be missing when going back is not possible or not permitted.

          cancel: Object.
              This will fully cancel the current interview.
              It will be missing if there is no option to cancel, for example after a data request has been submitted to others.

          see_other_options: Object.
              This will show the user their other options available at this moment other than continuing the interview.
              It may be missing if there are no other options or if we do not wish to highlight any other options in the current state.

          If an action is listed, then it means that the action is currently supported.

          Each action will have these keys:
              action_label: String
                  Human-readable label for the action, translated into the current locale.
                  Note that, for example, the Continue action may not be labeled "Continue" in cases where another description may be more sensible.
          

Actions can be taken via POST. When POSTing, request JSON data is required:

{
    action_name: String, required. Must be one of the items listed in the "actions" object from the GET or previous POST response.
}

Response data for POST will be identical to the GET response structure.

If the action cannot be completed and requires user correction, a 4xx error will be retured.
If the action cannot be completed due to a user-uncorrectable server error, a 5xx error will be retured.
If the response is an error 4xx or 5xx, the response will be an object that contains an array of objects:

  { errors: [{ reason: "reason_identifier", message: "Human readable text error reason" }] }
