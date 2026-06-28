# Designing with the Core

## 1. Clocking (`aclk`)
Main clock source that synchronously controls all internal states, inputs, and outputs.

### Clock Enable (`aclken`)
* **Operations:**
  * **LOW:** Pauses the core (freezes current state).
  * **HIGH:** Resumes normal core processing.
* **Design Note:** Using `aclken` can reduce the core's maximum operating frequency ($f_{max}$).
