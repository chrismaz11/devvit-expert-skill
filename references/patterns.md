# Devvit Common Patterns

## 1. User profile — load or create

```typescript
async function loadOrCreateUser(context: Devvit.Context): Promise<UserProfile> {
  const userId = context.userId;
  const username = context.username;
  if (!userId || !username) throw new Error('User not logged in');

  const existing = await context.redis.get(`user:${userId}`);
  if (existing) return JSON.parse(existing) as UserProfile;

  const profile: UserProfile = {
    userId,
    username,
    karma: 100,
    streak: 0,
    remindersEnabled: false,
    joined: new Date().toISOString(),
  };
  await context.redis.set(`user:${userId}`, JSON.stringify(profile));
  return profile;
}
```

## 2. Full useWebView wiring

```typescript
// src/main.tsx
type WebToServer =
  | { type: 'INIT_REQUEST' }
  | { type: 'VOTE'; choice: string }
  | { type: 'GET_LEADERBOARD' };

type ServerToWeb =
  | { type: 'INIT'; user: UserProfile; poll: PollData | null }
  | { type: 'VOTE_CONFIRMED'; choice: string; user: UserProfile }
  | { type: 'LEADERBOARD'; entries: Entry[] }
  | { type: 'ERROR'; message: string };

Devvit.addCustomPostType({
  name: 'MyPoll',
  height: 'tall',
  render: (context) => {
    const webView = useWebView<WebToServer, ServerToWeb>({
      onMessage: async (msg, hook) => {
        try {
          if (msg.type === 'INIT_REQUEST') {
            const [user, poll] = await Promise.all([
              loadOrCreateUser(context),
              getPoll(context),
            ]);
            hook.postMessage({ type: 'INIT', user, poll });
          } else if (msg.type === 'VOTE') {
            const user = await saveVote(context, msg.choice);
            hook.postMessage({ type: 'VOTE_CONFIRMED', choice: msg.choice, user });
          } else if (msg.type === 'GET_LEADERBOARD') {
            const entries = await getLeaderboard(context);
            hook.postMessage({ type: 'LEADERBOARD', entries });
          }
        } catch (err) {
          hook.postMessage({ type: 'ERROR', message: String(err) });
        }
      },
    });
    return <webview url="index.html" grow />;
  },
});
```

```javascript
// webroot/index.html — receiving messages
window.addEventListener('message', (event) => {
  if (event.data.type !== 'devvit-message') return;
  const msg = event.data.data.message;  // nested twice!

  if (msg.type === 'INIT') {
    state.user = msg.user;
    state.poll = msg.poll;
    render();
  } else if (msg.type === 'ERROR') {
    showError(msg.message);
  }
});

function send(msg) {
  window.parent.postMessage(msg, '*');
}

document.addEventListener('DOMContentLoaded', () => {
  send({ type: 'INIT_REQUEST' });
});
```

## 3. Leaderboard with sorted sets

```typescript
const LEADERBOARD_KEY = 'leaderboard:alltime';

async function updateLeaderboard(context: Devvit.Context, userId: string, delta: number) {
  await context.redis.zIncrBy(LEADERBOARD_KEY, userId, delta);
}

async function getTopN(context: Devvit.Context, n: number): Promise<LeaderboardEntry[]> {
  const members = await context.redis.zRange(LEADERBOARD_KEY, 0, n - 1, {
    reverse: true,
    withScores: true,
  }) as { member: string; score: number }[];

  return Promise.all(members.map(async ({ member, score }, i) => {
    const userJson = await context.redis.get(`user:${member}`);
    const user = userJson ? JSON.parse(userJson) : { username: 'Unknown' };
    return { rank: i + 1, userId: member, username: user.username, score };
  }));
}

async function getUserRank(context: Devvit.Context, userId: string) {
  return await context.redis.zRank(LEADERBOARD_KEY, userId, { reverse: true });
  // returns 0 for #1, undefined if not on board
}
```

## 4. Scheduled jobs: full pattern

```typescript
// Step 1: Define the job handler
Devvit.addSchedulerJob({
  name: 'generateDailyCase',
  onRun: async (event, context) => {
    const today = getToday();  // YYYY-MM-DD in ET

    // Idempotency check — don't run twice
    const existing = await context.redis.get(`case:${today}`);
    if (existing) {
      console.log('Case already exists for', today);
      return;
    }

    const dailyCase = {
      date: today,
      subreddit: 'funny',
      created: new Date().toISOString(),
    };
    await context.redis.set(`case:${today}`, JSON.stringify(dailyCase));
    console.log('Created case for', today);
  },
});

// Step 2: Schedule it on AppInstall
Devvit.addTrigger({
  event: 'AppInstall',
  onEvent: async (_event, context) => {
    await context.scheduler.runJob({
      name: 'generateDailyCase',
      cron: '0 0 * * *',  // midnight ET
    });
    console.log('Scheduled generateDailyCase');
  },
});
```

## 5. Atomic counter with transactions

```typescript
async function atomicIncrement(context: Devvit.Context, key: string, amount: number) {
  let attempts = 0;
  while (attempts < 5) {
    const txn = await context.redis.watch(key);
    const current = parseInt(await context.redis.get(key) ?? '0');
    await txn.multi();
    await txn.set(key, String(current + amount));
    const result = await txn.exec();
    if (result !== null) return current + amount;  // success
    attempts++;
    await new Promise(r => setTimeout(r, 50));  // backoff
  }
  throw new Error('Transaction failed after retries');
}
```

## 6. ET timezone handling

```typescript
function getETDateString(date = new Date()): string {
  const ETOffset = isEDT(date) ? -4 : -5;
  const etMs = date.getTime() + ETOffset * 3600 * 1000;
  const etDate = new Date(etMs);
  return etDate.toISOString().slice(0, 10);  // YYYY-MM-DD
}

function isEDT(date: Date): boolean {
  const year = date.getUTCFullYear();
  const start = getNthSundayOfMonth(year, 2, 2);  // March, 2nd Sunday
  const end = getNthSundayOfMonth(year, 10, 1);   // November, 1st Sunday
  return date >= start && date < end;
}
// Or use date-fns-tz: formatInTimeZone(date, 'America/New_York', 'yyyy-MM-dd')
```

## 7. Sending DMs with rate limiting

```typescript
async function sendReminderDM(context: Devvit.Context, username: string, message: string) {
  try {
    await context.reddit.sendPrivateMessage({
      to: username,
      subject: 'Daily Docket Reminder',
      text: message,
    });
  } catch (err) {
    // DMs can fail if user has messaging disabled — don't crash the whole job
    console.error(`Failed to DM ${username}:`, err);
  }
}

// In a job processing many users, add a small delay to avoid rate limits:
for (const userId of userIds) {
  await processUser(context, userId);
  await new Promise(r => setTimeout(r, 100));  // 100ms between users
}
```

## 8. Custom post creation from menu item

```typescript
Devvit.addMenuItem({
  label: '⚖️ Create Daily Docket Post',
  location: 'subreddit',
  forUserType: 'moderator',
  onPress: async (_event, context) => {
    const post = await context.reddit.submitPost({
      subredditName: context.subredditName!,
      title: `Daily Docket — ${getETDateString()}`,
      preview: (
        <vstack backgroundColor="#1a1a2e" padding="large" alignment="center middle">
          <text color="white" size="xlarge" weight="bold">⚖️ Loading...</text>
        </vstack>
      ),
    });
    context.ui.navigateTo(post);
    context.ui.showToast({ text: 'Post created!', appearance: 'success' });
  },
});
```

## 9. Storing lists of IDs

**Option A: Redis list (ordered, allows duplicates)**
```typescript
await context.redis.rPush(`votes:${date}`, userId);
const allVoters = await context.redis.lRange(`votes:${date}`, 0, -1);
```

**Option B: Redis sorted set (ordered by score, unique members)**
```typescript
await context.redis.zAdd(`votes:${date}`, { member: userId, score: Date.now() });
const allVoters = await context.redis.zRange(`votes:${date}`, 0, -1);
const count = await context.redis.zCard(`votes:${date}`);
```

## 10. Error handling strategy

```typescript
onMessage: async (msg, hook) => {
  try {
    // ... handle message
  } catch (err) {
    console.error('Error handling', msg.type, err);
    hook.postMessage({ type: 'ERROR', message: 'Something went wrong. Try again.' });
  }
},
```

In scheduled jobs, don't let one user's failure crash the whole job:
```typescript
const results = await Promise.allSettled(userIds.map(id => processUser(context, id)));
const failures = results.filter(r => r.status === 'rejected');
if (failures.length > 0) console.error(`${failures.length} users failed`);
```

## 11. Settings-based configuration

```typescript
// Define:
Devvit.addSettings([
  { name: 'maxStake', type: 'number', label: 'Max stake per bet', scope: 'installation', defaultValue: 500 },
]);

// Read in any context:
const maxStake = (await context.settings.get<number>('maxStake')) ?? 500;
```

## 12. Pagination with Redis cursors

```typescript
async function getAllUserKeys(context: Devvit.Context): Promise<string[]> {
  const keys: string[] = [];
  let cursor = 0;
  do {
    const result = await context.redis.scan(cursor, 'user:*', 100);
    cursor = result.cursor;
    keys.push(...result.keys);
  } while (cursor !== 0);
  return keys;
}
```

## 13. Blocks JSX quick reference

```tsx
// vstack / hstack = vertical / horizontal flex containers
<vstack width="100%" height="100%" alignment="center middle" backgroundColor="#1a1a2e">
  <image url="logo.png" width="120px" height="120px" imageWidth={120} imageHeight={120} />
  <text size="xlarge" color="white" weight="bold">Title</text>
  <spacer size="medium" />
  <button onPress={() => doThing()} appearance="primary">Click me</button>
  <webview url="index.html" grow />
</vstack>

// Text sizes: 'xsmall' | 'small' | 'medium' | 'large' | 'xlarge' | 'xxlarge'
// button appearance: 'primary' | 'secondary' | 'plain' | 'bordered' | 'destructive'
// padding/gap: 'none' | 'xsmall' | 'small' | 'medium' | 'large'
```
