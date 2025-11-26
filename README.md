# Google Play Games Services Plugin for Cordova

A modern Cordova plugin for Google Play Games Services v2 API with comprehensive gaming features.

## Features

- **Authentication**
  - Manual silent sign-in via initialize()
  - Background sign-out detection
  - Manual sign-in support
  - Sign-in state events
  - Modern v2 authentication flow
- **Leaderboards**
  - Submit scores
  - Show leaderboards
  - Get player scores
  - Get player rankings
- **Achievements**
  - Unlock achievements
  - Increment achievements
  - Show achievements UI
  - Reveal hidden achievements
  - Set achievement steps
  - Load all achievements
- **Cloud Saves**
  - Save game data
  - Load game data
  - Show saved games UI
  - Conflict resolution
  - Snapshot management
  - Delete a snapshot
  - Load all snapshots
- **Friends**
  - Get friends list
  - Show player profiles
  - Player search
  - Compare profiles
- **Player Stats**
  - Get player info
  - Get player stats
  - Get player level info
- **Events**
  - Increment events
  - Get event data
  - Get all events
  - Event tracking

## Requirements

- Cordova >= 12.0.0
- Cordova Android >= 14.0.0
- Android SDK >= 24
- Google Play Services >= 21.2.0

## Installation

```bash
cordova plugin add cordova-plugin-gpgs --variable APP_ID="your-app-id" --variable PLAY_SERVICES_VERSION="23.2.0"
```

To install directly from Git (e.g., the latest `main` branch or a fork), supply the repository URL and the same configuration variables:

```bash
cordova plugin add https://github.com/<org-or-user>/cordova-plugin-gpgs.git \
  --variable APP_ID="your-app-id" \
  --variable PLAY_SERVICES_VERSION="23.2.0" \
  --variable SERVER_CLIENT_ID="your-server-client-id"
```

To pin a branch or tag, append `#branch_or_tag` to the URL:

```bash
cordova plugin add https://github.com/<org-or-user>/cordova-plugin-gpgs.git#main --variable APP_ID="your-app-id"
```

### Configuration Variables

- `APP_ID` (required): Your Google Play Games App ID
- `PLAY_SERVICES_VERSION` (optional): Version of Google Play Services to use (default: 23.2.0)
- `SERVER_CLIENT_ID` (optional): OAuth 2.0 server client ID used to request `serverAuthCode` during login. Use the **Web application** client ID from the Google Cloud project linked to your Play Games Services game (Play Console → Game configuration → Linked apps → Google Cloud → Credentials). This is the value your backend will exchange for tokens. When provided, the plugin now silently requests the OpenID Connect scopes (`openid email profile`) and ID token along with the server auth code so your backend receives an `id_token` during the exchange.

  **How to obtain `SERVER_CLIENT_ID`:**
  1. Open [Google Cloud Console](https://console.cloud.google.com/) for the project linked to your Play Games Services game (from Play Console → Game configuration → Linked apps → Google Cloud).
  2. Go to **APIs & Services → Credentials**.
  3. Under **OAuth 2.0 Client IDs**, create or pick a **Web application** client.
  4. Copy its **Client ID** (looks like `1234567890-abcdefg.apps.googleusercontent.com`) and pass it as `SERVER_CLIENT_ID` when installing the plugin.
  5. Use the same client ID (and matching redirect URI) on your backend when exchanging the `serverAuthCode` for tokens.

When exchanging the returned `serverAuthCode`, ensure the Web client in Google Cloud Console has an **Authorized redirect URI** that matches your backend flow:
- If Google Cloud Console rejects `postmessage`, register an HTTPS URI for your backend exchange endpoint (for example `https://api.example.com/oauth2/callback`) and pass the same value as `redirectUri` when calling `getToken` on the server.
- If `postmessage` is allowed for your project, you can keep using it for mobile/JS code exchanges to avoid `redirect_uri_mismatch`.

## Configuration

Add the following to your `config.xml`:

```xml
<preference name="GPGS_DEBUG" value="true" />
```

To mirror the native debug log into the browser console/WebView, attach a logger callback after
`deviceready`:

```javascript
GPGS.setLogger(message => console.log('[GPGS]', message));
// Later, you can stop forwarding with:
// GPGS.clearLogger();
```
The logger only emits when `GPGS_DEBUG` is enabled.

## Usage

### Initialization

Call `initialize()` once after `deviceready`. It performs a silent sign-in and fires the usual events (`gpgs.signin`, `gpgs.signout`, `gpgs.availability`). All Play Games API calls that require authentication should be made after the `gpgs.signin` event has fired.

```javascript
document.addEventListener('deviceready', () => {
    GPGS.initialize()
        .then(() => {
            console.log('GPGS initialization request sent');        
        })
        .catch(console.error);
});
```

The plugin NO LONGER attempts silent sign-in automatically; you are in full control of when the operation happens.

The `gpgs.signin` event always includes `{ isSignedIn: boolean }` and, after a manual `GPGS.login()` call, also contains `playerId`, `username`, and (when `SERVER_CLIENT_ID` is configured) `serverAuthCode`. When a server client ID is present, the payload also includes the scopes the plugin asked for and what Google actually granted: `requestedScopes` and `grantedScopes` (arrays of scope URIs).

> Note: `grantedScopes` can be empty even when sign-in succeeds. This happens when Google cannot silently upgrade to full Google Sign-In with the OpenID scopes and the plugin falls back to the Play Games `requestServerSideAccess` call. In that path the SDK only exposes scopes from the *previously* signed-in Google account (if any), so you will see an empty list when there is no cached Google sign-in. To force consent and get a non-empty list, sign out and sign in again so Google prompts for `openid email profile` with your server client ID.

### Authentication

```javascript
// Check if Google Play Services are available
GPGS.isGooglePlayServicesAvailable().then(result => {
    if (result === true) {
        console.log('Google Play Services are available');
    } else if (typeof result === 'object') {
        console.log('Google Play Services are not available:', result.errorString);
        if (result.isUserResolvable) {
            // Show UI to help user resolve the issue
        }
    }
});
// Returns: Promise<boolean|Object> - If services are not available, returns an object with:
// {
//   available: false,
//   errorCode: number,
//   errorString: string,
//   isUserResolvable: boolean
// }

// Check if user is signed in
GPGS.isSignedIn().then(result => {
    if (typeof result === 'object') {
        console.log('Sign-in status:', result.isSignedIn);
    } else {
        console.log('Sign-in status:', result);
    }
});
// Returns: Promise<boolean|Object> - Returns either a boolean or an object with:
// {
//   isSignedIn: boolean
// }

// Manual sign-in + scope logging
GPGS.login().then(result => {
    console.log('Sign-in successful', result);
    console.log('Requested scopes:', result.requestedScopes);
    console.log('Granted scopes:', result.grantedScopes);
    // result example when SERVER_CLIENT_ID is set:
    // {
    //   isSignedIn: true,
    //   playerId: '1234567890123456789',
    //   username: 'Player One',
    //   serverAuthCode: '4/0AX4XfW...',
    //   requestedScopes: ['openid','email','profile','https://www.googleapis.com/auth/games_lite'],
    //   grantedScopes:   ['openid','email','profile','https://www.googleapis.com/auth/games_lite']
    // }
}).catch(error => {
    console.error('Sign-in failed:', error);
});
// Returns: Promise<{ isSignedIn: boolean, playerId?: string, username?: string, serverAuthCode?: string, requestedScopes?: string[], grantedScopes?: string[] }>

// Manual sign-out (also triggers the gpgs.signout event with { isSignedIn: false, reason: 'user_signout' })
GPGS.signOut().then(() => {
    console.log('Signed out');
}).catch(error => {
    console.error('Sign-out failed:', error);
});
// Returns: Promise<void>
```

### Leaderboards

```javascript
// Submit a score
GPGS.submitScore('leaderboard_id', 1000).then(() => {
    console.log('Score submitted');
});
// Returns: Promise<void>

// Show a leaderboard
GPGS.showLeaderboard('leaderboard_id').then(() => {
    console.log('Leaderboard shown');
});
// Returns: Promise<void>

// Show all leaderboards
GPGS.showAllLeaderboards().then(() => {
    console.log('All leaderboards shown');
});
// Returns: Promise<void>

// Get player's score
GPGS.getPlayerScore('leaderboard_id').then(score => {
    console.log('Player score:', score);
});
// Returns: Promise<{
//   player_score: number,
//   player_rank: number
// }>

// --- Parameter enums ------------------------------------------------------
// timeSpan   → 0 = daily, 1 = weekly, 2 = all-time
// collection → 0 = public, 1 = friends
// For richer code samples see: examples/scores.md

// Load top scores for a leaderboard
GPGS.loadTopScores('leaderboard_id', 2 /*all-time*/, 0 /*public*/, 25).then(result => {
    console.log('Leaderboard:', result.leaderboard);
    console.log('Scores:', result.scores);
});
// Returns: Promise<Object>

// -------------------------------------------------------------------------
// See examples/scores.md for showing a leaderboard slice around the player

// Load scores centered around the signed-in player
GPGS.loadPlayerCenteredScores('leaderboard_id', 2 /*all-time*/, 0 /*public*/, 25).then(result => {
    console.log('Leaderboard:', result.leaderboard);
    console.log('Scores:', result.scores);
});
// Returns: Promise<Object>

// Load metadata for a single leaderboard
GPGS.loadLeaderboardMetadata('leaderboard_id').then(metadata => {
    console.log('Leaderboard Metadata:', metadata);
});
// Returns: Promise<Object>

// Load metadata for all leaderboards
GPGS.loadLeaderboardMetadata().then(metadata => {
    console.log('All Leaderboards Metadata:', metadata);
});
// Returns: Promise<Array<Object>>
```

### Achievements

```javascript
// Unlock an achievement
GPGS.unlockAchievement('achievement_id').then(() => {
    console.log('Achievement unlocked');
});
// Returns: Promise<void>

// Increment an achievement
GPGS.incrementAchievement('achievement_id', 1).then(() => {
    console.log('Achievement incremented');
});
// Returns: Promise<void>

// Show achievements UI
GPGS.showAchievements().then(() => {
    console.log('Achievements UI shown');
});
// Returns: Promise<void>

// Reveal a hidden achievement
GPGS.revealAchievement('achievement_id').then(() => {
    console.log('Achievement revealed');
});
// Returns: Promise<void>

// Set achievement steps
GPGS.setStepsInAchievement('achievement_id', 5).then(() => {
    console.log('Achievement steps set');
});
// Returns: Promise<void>

// Load all achievements
GPGS.loadAchievements(false).then(achievements => {
    console.log('Achievements:', achievements);
});
// Returns: Promise<Array<Object>>

// Get friends list
GPGS.getFriendsList().then(friends => {
    console.log('Friends:', friends);
});
// Returns: Promise<Array<{
//   id: string,
//   displayName: string
// }>>

// Show player profile
GPGS.showPlayerProfile('player_id').then(() => {
    console.log('Player profile shown');
});
// Returns: Promise<void>

// Show player search
GPGS.showPlayerSearch().then(() => {
    console.log('Player search shown');
});
// Returns: Promise<void>
```

### Cloud Saves

```javascript
// Save game data
GPGS.saveGame('save_name', 'description', {
    level: 1,
    score: 1000
}).then(() => {
    console.log('Game saved');
});
// Returns: Promise<void>

// Load game data
GPGS.loadGame('save_name').then(data => {
    console.log('Game loaded:', data);
});
// Returns: Promise<Object> - The saved game data

// Show saved games UI
GPGS.showSavedGames({
    title: 'Saved Games',
    allowAddButton: true,
    allowDelete: true,
    maxSnapshots: 5
}).then(() => {
    console.log('Saved games UI shown');
});
// Returns: Promise<void>

// Delete a snapshot
GPGS.deleteSnapshot('save_name').then(snapshotId => {
    console.log('Snapshot deleted:', snapshotId);
});
// Returns: Promise<string>

// Load all snapshots
GPGS.loadAllSnapshots(false).then(snapshots => {
    console.log('All snapshots:', snapshots);
});
// Returns: Promise<Array<Object>>
```

### Player Stats

```javascript
// Get player info
GPGS.getPlayerInfo('player_id', true).then(info => {
    console.log('Player info:', info);
});
// Returns: Promise<{
//   id: string,
//   displayName: string,
//   title: string,
//   levelInfo?: {
//     currentLevel: number,
//     maxXp: number,
//     minXp: number
//   }
// }>

// Get player stats
GPGS.getPlayerStats().then(stats => {
    console.log('Player stats:', stats);
});
// Returns: Promise<{
//   averageSessionLength: number,
//   daysSinceLastPlayed: number,
//   numberOfPurchases: number,
//   numberOfSessions: number,
//   sessionPercentile: number,
//   spendPercentile: number,
//   spendProbability: number
// }>
```

### Events

```javascript
// Increment an event
GPGS.incrementEvent('event_id', 1).then(() => {
    console.log('Event incremented');
});
// Returns: Promise<void>

// Get all events
GPGS.getAllEvents().then(events => {
    console.log('All events:', events);
});
// Returns: Promise<Array<{
//   id: string,
//   name: string,
//   description: string,
//   value: number
// }>>

// Get specific event
GPGS.getEvent('event_id').then(event => {
    console.log('Event:', event);
});
// Returns: Promise<{
//   id: string,
//   name: string,
//   description: string,
//   value: number
// }>
```

## Events

The plugin emits the following events:

### `gpgs.signin`
Emitted when sign-in state changes.
```javascript
{
    isSignedIn: boolean,
    error?: string  // Present if sign-in failed
}
```

### `gpgs.signout`
Emitted when user signs out (including background sign-out).
```javascript
{
    isSignedIn: false,
    reason: string  // e.g., "background_signout"
}
```

### `gpgs.availability`
Emitted when Google Play Services availability changes.
```javascript
{
    available: boolean,
    errorCode?: number,
    errorString?: string,
    isUserResolvable?: boolean
}
```

## Error Handling

The plugin uses promises for all operations. Errors are passed to the catch handler:
```javascript
GPGS.login().catch(error => {
    console.error('Error:', error.message, 'Status Code:', error.statusCode);
});
```

The error object contains:
- `message`: A descriptive error message
- `statusCode`: The status code from the underlying Google Play Games SDK (if available)

Common error codes from the SDK can be found in the official documentation.

## Debug Mode

Enable debug mode in `config.xml` to see detailed logs:
```xml
<preference name="GPGS_DEBUG" value="true" />
```

## License

This project is licensed under the GPL-3.0-or-later License - see the [LICENSE](LICENSE) file for details.

## Author

Exelerus AB - [https://exelerus.com](https://exelerus.com)

## Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## Examples

- [Listening to sign-in events](examples/events.md)
- [Working with leaderboards & scores](examples/scores.md)