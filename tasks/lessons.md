# Lessons

## Don't add behavior the user didn't ask for
- **Mistake:** Added "night counts as cloudy" to the salt cell weather automation on my own judgment. The user had it removed.
- **Rule:** Implement exactly the requested behavior. If an extra rule seems useful (e.g. day/night handling), propose it as an option in the plan or ask first. Don't bake it in silently.

## Automations that reduce output should consider timing
- **Pattern:** The user wanted a configurable delay (how long a condition must hold) before acting.
- **Rule:** For condition-driven automations, ask up front whether the user wants a delay or debounce before the action fires, and in which direction(s).
