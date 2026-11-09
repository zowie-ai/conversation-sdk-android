# Zowie Conversation SDK for Android

Embed the Zowie chat and your AI agent in your Android app. Add `ZowieChat` to
your Compose UI — or `ZowieChatFragment` / `ZowieChatActivity` to a View-based
app — give it your `baseUrl` and `instanceId`, and it renders the full Zowie chat
experience with configuration fetching and encrypted session persistence built
in.

## Requirements

- **minSdk 24** (Android 7.0 Nougat) or higher.
- **compileSdk 37** or higher.
- Build your app with **JDK 17** or newer.
- The SDK exports the **Compose BOM `2026.08.00`**, so your own Compose
  artifacts resolve to at least those versions.

You only need the Compose compiler plugin enabled in your app if you use the
`ZowieChat` composable. `ZowieChatFragment` and `ZowieChatActivity` render Compose
internally and work in a purely View-based app.

Everything the SDK needs arrives transitively. Should you hit a version clash,
the dependencies likely to collide with your own are `androidx.webkit`,
`androidx.security:security-crypto`, OkHttp, Coil and kotlinx.serialization,
alongside AndroidX Compose, Activity and Fragment.

## Installation

The SDK is published as `ai.zowie:conversation-sdk`. Add the repository, which is
open for reading and needs no credentials:

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven("https://zowie.jfrog.io/artifactory/zowie-android-sdk")
    }
}
```

```kotlin
// app/build.gradle.kts
dependencies {
    implementation("ai.zowie:conversation-sdk:2.0.2")
}
```

See [CHANGELOG.md](CHANGELOG.md) for what changed in each version. All released
versions are in
[the repository](https://zowie.jfrog.io/artifactory/zowie-android-sdk/ai/zowie/conversation-sdk/).

The SDK declares `android.permission.INTERNET` in its own manifest, so you do not
need to add it. See [Permissions](#permissions) for what you *do* need to declare.

### R8 and ProGuard

Nothing to add. The SDK ships `consumer-rules.pro`, which keeps the JavaScript
bridge its `WebView` calls into, so a minified release build works as it stands.

## SDK version

Read the version you are running with `ZowieConversationSdk.VERSION`:

```kotlin
import ai.zowie.conversationsdk.ZowieConversationSdk

val sdkVersion = ZowieConversationSdk.VERSION
```

## Quick start

### Compose

```kotlin
import ai.zowie.conversationsdk.ZowieChat
import ai.zowie.conversationsdk.zowieChatConfig

setContent {
    ZowieChat(
        config = zowieChatConfig(
            baseUrl = "https://<slug>.chat.getzowie.com",
            instanceId = "<your-instance-id>",
        ),
        modifier = Modifier.fillMaxSize(),
    )
}
```

`ZowieChat` has **no intrinsic size** — pass `Modifier.fillMaxSize()` or an
explicit height, otherwise it renders with zero height.

### Fragment

```kotlin
val fragment = ZowieChatFragment.newInstance(
    zowieChatConfig(baseUrl = "…", instanceId = "…")
)
fragment.setEventListener(object : ZowieChatEventListener {
    override fun onChatStarted(conversationId: String) { /* … */ }
})

supportFragmentManager.beginTransaction()
    .replace(R.id.container, fragment)
    .commit()
```

Both `ZowieChatFragment` and `ZowieChatActivity` are `open`. Inherit from them when using the
**Jetpack Navigation Component**, where you don't have a direct reference to the fragment instance
to set listeners:

```kotlin
class MyChatFragment : ZowieChatFragment(), ZowieChatEventListener {

    override fun getZowieConfig(): ZowieChatConfig =
        zowieChatConfig(
            baseUrl = "…",
            instanceId = "…",
        )

    override fun onAttach(context: Context) {
        super.onAttach(context)
        setEventListener(listener = this)
    }

    override fun onChatStarted(conversationId: String) { /* … */ }
}
```

> [!NOTE]
> When providing a `ZowieChatEventListener` for a fragment, you are responsible for handling 
> the `onMinimized` and `ChatEnded` event (e.g., by popping the backstack). If no listener is set,
> the fragment calls `parentFragmentManager.popBackStack()` by default, or removes itself when the
> back stack is empty.

### Activity

```kotlin
startActivity(ZowieChatActivity.createIntent(this, config))
```

The activity renders the chat full screen and finishes itself when the user taps
minimize. Inherit from `ZowieChatActivity` to set event listeners, handlers and
access the controller:

```kotlin
class CustomChatActivity : ZowieChatActivity(), ZowieChatEventListener {

    override fun getZowieConfig(): ZowieChatConfig =
        zowieChatConfig(
            baseUrl = "…",
            instanceId = "…",
        )

    override fun onCreate(savedInstanceState: Bundle?) {
        setEventListener(this)
        super.onCreate(savedInstanceState)
    }

    override fun onChatStarted(conversationId: String) { /* … */ }
}
```

> [!NOTE]
> When providing a `ZowieChatEventListener` for an activity, you are responsible for handling 
> the `onMinimized` and `ChatEnded` event. If no listener is set, the activity calls `finish()`
> by default.

## Window insets and edge-to-edge

The SDK applies the on-screen keyboard inset to its `WebView` itself. Without it
the chat would keep its full height when the keyboard opens, and the browser
would scroll the whole page to reveal the composer, carrying the header off the
top of the screen. Declaring `android:windowSoftInputMode="adjustResize"` does
not prevent this: it has no effect once the window is edge-to-edge, which
Android enforces once your app targets SDK 35 or higher.

System bars are left to you, since only you know what else is on the screen.
`ZowieChatActivity` applies them for you; in Compose and Fragment, pad the
container you put the chat in — `Modifier.systemBarsPadding()`, or whatever
your layout already does for the rest of the screen.

If your app already applies the keyboard inset at the root — a global
`ViewCompat.setOnApplyWindowInsetsListener`, or `Modifier.imePadding()` on an
outer container — the chat would be inset twice and leave a gap above the
keyboard. Consume the insets at your level instead:

```kotlin
ViewCompat.setOnApplyWindowInsetsListener(root) { _, _ ->
    // Handle your app's own views here

    // Stop nested content, the chat included, from applying them again
    WindowInsetsCompat.CONSUMED
}
```

Or keep the container that hosts the chat out of your global inset handling.

## Configuration

`ZowieChatConfig` is built with the `zowieChatConfig { }` DSL from Kotlin, or with
`ZowieChatConfig.Builder(baseUrl, instanceId)` from Java.

```kotlin
val config = zowieChatConfig(
    baseUrl = "https://<slug>.chat.getzowie.com",
    instanceId = "<your-instance-id>",
) {
    title = "Support"
    primaryColor = "#2563EB"
    metadata {
        firstName = "Ada"
        email = "ada@example.com"
        locale = "en-US"
    }
}
```

### Required

| Field        | Type     | Description                                                                   |
| ------------ | -------- | ----------------------------------------------------------------------------- |
| `baseUrl`    | `String` | Base origin of your Zowie deployment, e.g. `https://<slug>.chat.getzowie.com`. |
| `instanceId` | `String` | Zowie chat instance ID.                                                       |

### Session and behaviour

| Field                     | Type               | Default          | Description                                             |
| ------------------------- | ------------------ | ---------------- | ------------------------------------------------------- |
| `metadata`                | `Metadata?`        | `null`           | User attributes. `metadata.locale` also selects the AI agent's region. |
| `externalAccessToken`     | `String?`          | `null`           | JWT for authenticated sessions; anonymous when omitted. |
| `context`                 | `String?`          | `null`           | ID of a context set up by your Zowie TAM. It routes the conversation to the AI agent assigned to that context. |
| `resetSession`            | `Boolean`          | `false`          | Clear any persisted session on init.                    |
| `initialConversationMode` | `ConversationMode` | `TEXT`           | Mode the chat opens in (`TEXT` or `VOICE`).             |
| `referral`                | `String`           | `"start"`        | Referral used for auto-start.                           |
| `initialUserMessage`      | `String?`          | `null`           | Initial user message sent with auto-start.              |
| `versionType`             | `VersionType`      | `PRODUCTION`     | AI agent version the conversation runs against (`PRODUCTION` or `STAGING`). |

#### Testing against the staging version of your AI agent

Set `versionType = VersionType.STAGING` to run the conversation against the
AI agent's current staging version instead of the published version, so the AI
agent can be tried out from the app before it goes live:

```kotlin
val config = zowieChatConfig(baseUrl = BASE_URL, instanceId = INSTANCE_ID) {
    versionType = VersionType.STAGING
}
```

Leave it at `PRODUCTION` in release builds.

### Branding

Values set here override the ones configured in the Zowie panel.

| Field                    | Type         | Description                                     |
| ------------------------ | ------------ | ----------------------------------------------- |
| `title`                  | `String?`    | Header title.                                   |
| `primaryColor`           | `String?`    | Primary color, applied to both light and dark.  |
| `fontColor`              | `FontColor?` | `WHITE` or `BLACK`, applied to both themes.     |
| `userMessageBackgroundColor` | `String?` | User message bubble background color.           |
| `userMessageFontColor`   | `FontColor?` | User message bubble text color.                 |
| `logoUrl`                | `String?`    | Logo shown in the chat header.                  |
| `headerVisible`          | `Boolean`    | Whether the chat header is visible (default `false`). |
| `voiceExperienceEnabled` | `Boolean?`   | Enable voice conversation mode.                 |
| `voiceBlobColor`         | `String?`    | Color of the voice visualization.               |
| `theme`                  | `ThemeMode?` | `SYSTEM`, `LIGHT` or `DARK`. Defaults to system. |

## Events

In Compose pass `onEvent` and match on the `ZowieChatEvent` sealed interface; with
a fragment implement `ZowieChatEventListener`, whose methods all have empty
defaults so you override only what you need.

| Event / callback     | Payload                        | Fires when…                                  |
| -------------------- | ------------------------------ | -------------------------------------------- |
| `Loaded`             | —                              | The chat finished loading.                   |
| `ChatStarted`        | `conversationId: String`       | A conversation started or was resumed.                    |
| `ChatEnded`          | —                              | The conversation ended.                      |
| `MessageSent`        | `message: String`              | The user sent a message.                     |
| `MessageReceived`    | `payload: String` (raw JSON)   | A message was received.                      |
| `SessionExpired`     | —                              | The session expired.                         |
| `UnreadMessages`     | `count: Int`                   | The unread message count changed.            |
| `Minimized`          | —                              | The user tapped the minimize control.         |
| `ConfigError`        | `error: Throwable`             | Loading the chat configuration or page failed. |
| `GenericEvent`       | `name: String`, `params: Map<String, String>` | A backend event fired (Decision Engine). |

## Back handling

Pass a `ZowieBackHandler` to intercept the system back gesture while the chat is
showing:

```kotlin
ZowieChat(
    config = config,
    modifier = Modifier.fillMaxSize(),
    backHandler = {
        navigateUp()
        true
    },
)
```

Return `true` when your app consumed the gesture, `false` to let it fall through
as if no handler were set. Without a handler the chat does not touch back at all
and the host's own handling applies.

For Fragment / Activity use `setBackHandler()`.

## AI session notice handling

Pass a `ZowieAiSessionNoticeMorePressHandler` to take over the "more" link in the
AI Session Notice:

```kotlin
ZowieChat(
    config = config,
    modifier = Modifier.fillMaxSize(),
    aiSessionNoticeMorePressHandler = { header, message ->
        showCustomDialog(header, message)
    },
)
```

Without a handler the SDK shows its own modal with the full notice content. When a
handler is set the modal is not shown — the host app takes over. The `message`
parameter may contain markdown.

For Fragment / Activity use `setAiSessionNoticeMorePressHandler()`.

## Link handling

Pass a `ZowieLinkHandler` to open links yourself:

```kotlin
ZowieChat(
    config = config,
    modifier = Modifier.fillMaxSize(),
    linkHandler = { url ->
        if (url.startsWith("http")) {
            openInAppBrowser(url)
            true
        } else {
            false
        }
    },
)
```

Return `true` when your app opened the link, `false` to let the SDK open it with
an `ACTION_VIEW` intent. **Returning `true` without doing anything silently drops
the link.**

Links pointing at the configured `baseUrl` stay inside the chat and never reach
the handler. Everything else does, including `mailto:` and `tel:` — guard on the
scheme if your handler can only open web pages.

## Download handling

Transcripts and attachments the user downloads are, by default, saved to the
shared **Downloads** folder and then opened with an `ACTION_VIEW` intent. If no
app can open the file type, the file is still saved and the SDK only logs a
warning.

Files in Downloads are visible to other apps and survive uninstalling your app.
If that does not fit your privacy or storage requirements, take over with a
`ZowieDownloadHandler`:

```kotlin
ZowieChat(
    config = config,
    modifier = Modifier.fillMaxSize(),
    downloadHandler = { download ->
        storeEncrypted(download)
        true
    },
)
```

Return `true` when you handled the download, `false` to let the SDK save it. To
observe downloads without changing what happens to them, do your work and return
`false`.

On **API 24–28** the shared Downloads folder needs a runtime storage permission,
which the SDK will not request on your behalf. There it falls back to the app's
private cache directory, which means the file is not visible to the user. Handle
downloads yourself if you need to support those versions properly.

## Types

### `Metadata`

User attributes, built with the `metadata { }` DSL or `Metadata.Builder`. All
fields are optional.

| Field         | Type                   | Description                                          |
| ------------- | ---------------------- | ---------------------------------------------------- |
| `firstName`   | `String?`              | User's first name.                                   |
| `lastName`    | `String?`              | User's last name.                                    |
| `name`        | `String?`              | Full display name.                                   |
| `email`       | `String?`              | User's email address.                                |
| `phoneNumber` | `String?`              | User's phone number.                                 |
| `locale`      | `String?`              | Locale, e.g. `en-US`. Also selects the AI agent's region. |
| `timezone`    | `String?`              | IANA timezone, e.g. `Europe/Warsaw`.                 |
| `extraParams` | `Map<String, String>?` | Arbitrary custom attributes.                         |

### `ThemeMode`

Controls the chat's color theme.

| Value    | Description                                        |
| -------- | -------------------------------------------------- |
| `SYSTEM` | Follow the device's light/dark setting (default).  |
| `LIGHT`  | Force light theme.                                 |
| `DARK`   | Force dark theme.                                  |

### `ChatDownload`

Passed to `ZowieDownloadHandler`. Exactly one of `base64` or `url` is set.

| Field      | Type      | Description                                                    |
| ---------- | --------- | -------------------------------------------------------------- |
| `filename` | `String`  | Suggested file name.                                           |
| `mimeType` | `String?` | MIME type, when known.                                         |
| `base64`   | `String?` | Base64 contents, for client-generated files (e.g. transcript). |
| `url`      | `String?` | Direct download URL.                                           |

## Permissions

The SDK declares only `android.permission.INTERNET`. Anything else you declare
yourself, so an app that does not use voice is never forced to ask for the
microphone.

### Voice (microphone)

Add to your `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
```

The SDK requests `RECORD_AUDIO` at runtime the first time the chat asks for
audio capture, but the request is denied immediately unless the permission is
declared here. `MODIFY_AUDIO_SETTINGS` is a normal permission (no runtime
prompt) that the `WebView` needs to give the chat access to the microphone.

### File upload and camera

The in-chat picker offers the files on the device alongside **Camera**, and
needs no permission for either. Taking a photo hands off to the system camera
app, which holds the permission itself, so nothing is required of you.

Declaring `CAMERA` in your own manifest (directly or through another library) is
the one thing that changes this. Android then expects *your* app to hold the
permission before it may hand off, so the SDK requests it when the picker opens.
If the user declines, the picker shows files only.

## Controlling the chat

Drive the chat from your own UI with a `ZowieChatController`. In Compose create
one with `rememberZowieChatController()` and pass it to `ZowieChat`:

```kotlin
val controller = rememberZowieChatController()

ZowieChat(
    config = config,
    modifier = Modifier.fillMaxSize(),
    controller = controller,
)

// elsewhere
controller.sendMessage("Hello")
```

With a fragment, read `fragment.controller`. A controller that is never passed to
a `ZowieChat` does nothing, and calls made before the chat has loaded are
dropped.

| Method              | Description                                                        |
| ------------------- | ------------------------------------------------------------------ |
| `sendMessage`       | Send a message as the user.                                        |
| `sendReferral`      | Send a referral / campaign ID to the conversation.                 |
| `updateMetadata`    | Update the user attributes.                                        |
| `setVisible`        | Set whether the chat is visible.                                   |
| `showCsat`          | Show the CSAT survey. Optional `onCompleted` callback fires when the user submits or dismisses it. |
| `startChat`         | Start a new conversation when none is active (no-op otherwise).    |
| `endChat`           | Clear the stored session and current conversation (use on logout). |
| `getConversationId` | The active conversation's ID, or `null` when none is open.         |
| `registerPushToken` | Register an FCM registration token for push notifications.         |
| `deregisterPushToken` | Deregister the current device token; stops push notifications.   |

## Session persistence

Sessions are stored in encrypted `SharedPreferences` backed by the Android
Keystore, under the name `ai.zowie.session.<instanceId>`. Sessions survive app
restarts and process death.

Call `ZowieChatSession.clearAnonymousSession` to clear a stored session without
opening the chat (e.g. on user logout when no chat is on screen):

```kotlin
ZowieChatSession.clearAnonymousSession(context, instanceId)
```

If a chat is currently open, use `controller.endChat()` instead — it ends the
conversation and clears the session in one step.

## Push notifications

The SDK can register an FCM registration token with the Zowie backend so the user
receives push notifications for new messages.

```kotlin
controller.registerPushToken(fcmToken)
```

Call `deregisterPushToken()` when push notifications should stop for this device
(e.g. on user logout).

```kotlin
controller.deregisterPushToken()
```

If the FCM token value changes, call `registerPushToken` again with the new token.

**Behaviour notes:**

- If the chat session is not yet authenticated, `registerPushToken` queues the
  token and flushes it automatically when authentication completes.
- The token is automatically cleared when the chat ends or the session expires.
- The host app is responsible for Firebase/FCM setup (dependencies, services,
  `google-services.json`). The SDK only handles token registration with the
  Zowie backend.

## Known limitations

- **Downloads fall back to private cache on API 24–28** (see above).

## License

MIT
