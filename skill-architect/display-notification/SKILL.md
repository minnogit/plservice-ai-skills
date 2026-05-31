# PLService Notification System Rules (FlashMessage & showNotification)

You are an expert developer on the PLService codebase. Whenever you write, refactor, or review PHP or JavaScript code that requires displaying user feedback, alerts, errors, or operation outcomes, you MUST strictly adhere to the following architectural rules.

---

## 🚀 Core Decision Triage

Before implementing any notification, evaluate the flow in this exact order:

1. **Is there a PHP redirect (`header('Location: ...')`)?** → YES: Use `FlashMessage::set()`.
2. **Is it an AJAX/fetch call returning JSON?** → YES: Return a JSON response and call `showNotification()` inside the JS callback.
3. **Are you handling multi-field validation errors simultaneously?** → YES: You MUST use JavaScript with a loop of `showNotification()` calls (FlashMessage only allows one message per type).
4. **Is it an inline PHP script executed immediately during page parsing?** → YES: Wrap `showNotification()` inside a `DOMContentLoaded` event listener.
5. **Do you need an immediate page reload/redirect right after the alert?** → YES: Use native JavaScript `confirm()`. `showNotification()` is asynchronous and will vanish instantly.

---

## 📌 Technical Implementation Details

### 1. PHP Operations with Redirect (POST-redirect-GET Pattern)

* **Method:** `FlashMessage::set(string $type, string $message, bool $dismissable = false): void`
* **Rule:** Always call `FlashMessage::set()` *before* the redirect header and exit.
* **Anti-pattern:** NEVER use `echo` or `print` for notifications in POST targets. NEVER call `FlashMessage::display()` explicitly in custom scripts; it is handled automatically once per page inside `menu.php`.
* **Critical Errors:** For blocking validation or system errors, set `dismissable: true`. For standard success feedback, leave it `false` (default 5s autohide).

```php
// Example: Correct PHP Redirect Pattern
try {
    $db->beginTransaction();
    // business logic...
    $db->commit();
    FlashMessage::set('success', 'Operazione completata con successo.');
} catch (Exception $e) {
    $db->rollback();
    error_log($e->getMessage()); // Keep technical logs hidden from users
    FlashMessage::set('error', 'Impossibile completare l\'operazione. Riprova più tardi.', dismissable: true);
}
header('Location: ' . $_SERVER['PHP_SELF']);
exit;

```

### 2. AJAX & Fetch Requests

* **Backend Rule:** Do NOT use `FlashMessage` in AJAX endpoints. Return a standard JSON payload containing a status/success boolean and a `message` string.
* **Frontend Rule:** Call `showNotification(type, message, delay, dismissable)` directly inside the `.then()`/.catch() callbacks or jQuery AJAX success/error functions.
* **DOM Rule:** Do NOT wrap AJAX callbacks in `DOMContentLoaded`. The page is already fully loaded when the user triggers the AJAX request.

### 3. DOMContentLoaded Wrapper Strategy (Crucial for Inline Scripts)

Because `pls-notifications.js` is loaded with the `defer` attribute via `menu.php`, inline scripts running during the HTML parsing phase might fire before `showNotification` is defined.

* **Mandatory `DOMContentLoaded` wrapping required when:**
* Generating an inline `<script>` from a PHP `catch` block or conditional execution block before the end of the `<body>` or immediately after `menu.php`.
* The notification is executed *immediately* upon script execution (not inside a click/submit handler).

* **No wrapping required when:**
* Inside any asynchronous callback (`.then()`, `.catch()`, AJAX success).
* Inside standard event handlers (click, submit, change).
* Standalone pages where `pls-notifications.js` is manually imported in the `<head>` synchronously (without `defer`).

```php
// Example: Safe Inline PHP Catch Block Generation
catch (Throwable $e) {
    $msgJson = json_encode('Errore: ' . $e->getMessage(), JSON_HEX_TAG | JSON_HEX_AMP | JSON_HEX_APOS | JSON_HEX_QUOT);
    echo "<script>
        document.addEventListener('DOMContentLoaded', function() {
            showNotification('error', {$msgJson}, 0, true);
        });
    </script>";
}

```

### 4. Multi-Field Form Validations

* `FlashMessage` stores messages using the type as the key. Calling `FlashMessage::set('error', ...)` multiple times will overwrite previous errors, showing only the last one.
* **Rule:** For multiple simultaneous errors, validate client-side or use AJAX, collecting errors into an array and looping through them using `showNotification('error', msg, 0, true)`.

---

## 🚫 Absolute Constraints & Guardrails (Things to Never Do)

* **NEVER** expose real exception messages (`$e->getMessage()`) directly to the user interface via `FlashMessage` or `showNotification` if they leak SQL queries, system paths, or internal logic. Log them via `error_log()` instead.
* **NEVER** write `showNotification('success', 'Done'); window.location.reload();`. If a reload is mandatory, write `if (confirm('Done')) { window.location.reload(); }`.
* **NEVER** add manual imports of `/js/pls-notifications.js` on pages that already include `menu.php`, as it is loaded automatically. Only add manual imports in standalone modules, popups, or custom layout files (e.g., `nomi.php`).
