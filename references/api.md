# Devvit API Reference
Version: @devvit/public-api 0.12.x

## Table of Contents
1. [Devvit class](#devvit-class)
2. [useWebView](#usewebview)
3. [Context object](#context-object)
4. [Redis client](#redis-client)
5. [Scheduler client](#scheduler-client)
6. [Reddit API client](#reddit-api-client)
7. [UIClient](#uiclient)
8. [Triggers](#triggers)
9. [Forms](#forms)
10. [Settings](#settings)
11. [Realtime](#realtime)
12. [devvit.yaml](#devvityaml)

---

## Devvit class

### Devvit.configure(config)
Must be called before registering any jobs/triggers that need these capabilities.
```typescript
Devvit.configure({
  redis: true,         // context.redis
  redditAPI: true,     // context.reddit
  http: true,          // fetch() / HTTP requests
  media: true,         // context.media
  kvStore: true,       // context.kvStore (simpler KV, not full Redis)
});
```

### Devvit.addCustomPostType(opts)
```typescript
Devvit.addCustomPostType({
  name: 'MyApp',
  description: 'Short description shown in mod tools',
  height: 'tall',  // 'regular' | 'tall' | 'fullscreen'
  render: (context) => {
    return <webview url="index.html" grow />;
  },
});
```

### Devvit.addSchedulerJob(job)
```typescript
Devvit.addSchedulerJob({
  name: 'myJobName',
  onRun: async (event, context) => {
    const { data } = event;
    // context has: redis, reddit, scheduler, settings (no ui)
  },
});
```

### Devvit.addTrigger(trigger)
```typescript
Devvit.addTrigger({
  event: 'AppInstall',  // AppInstall | AppUpgrade | PostSubmit | PostCreate | PostDelete | PostReport | ModAction | etc.
  onEvent: async (event, context) => { ... },
});
```

### Devvit.addMenuItem(item)
```typescript
Devvit.addMenuItem({
  label: '📊 Create Post',
  location: 'subreddit',  // 'subreddit' | 'post' | 'comment' | 'user'
  forUserType: 'moderator',
  onPress: async (event, context) => {
    const post = await context.reddit.submitPost({
      subredditName: context.subredditName!,
      title: 'My Post',
      preview: <vstack padding="medium"><text>Loading...</text></vstack>,
    });
    context.ui.navigateTo(post);
    context.ui.showToast('Post created!');
  },
});
```

### Devvit.addSettings(fields)
```typescript
Devvit.addSettings([
  {
    type: 'string',
    name: 'api_key',
    label: 'API Key',
    scope: 'app',         // 'app' (dev only) | 'installation' (per subreddit)
    secret: true,
  },
  {
    type: 'boolean',
    name: 'feature_enabled',
    label: 'Enable Feature',
    scope: 'installation',
    defaultValue: true,
  },
]);
// Read: await context.settings.get('api_key')
```

---

## useWebView

The primary hook for Devvit Web apps. Call inside `render` of `addCustomPostType`.

```typescript
import { useWebView } from '@devvit/public-api';

const webView = useWebView<WebToServer, ServerToWeb>({
  url: 'index.html',
  onMessage: async (msg, hook) => {
    // hook.postMessage(msg: ServerToWeb) — send back to web
    // hook.mount() / hook.unmount() — control visibility
  },
  onUnmount: async (hook) => { },
});
// webView.postMessage(msg) — send from render scope
```

Web-side receiving pattern:
```javascript
window.addEventListener('message', (event) => {
  // CRITICAL: messages are nested two levels deep
  if (event.data.type !== 'devvit-message') return;
  const msg = event.data.data.message;  // <-- two .data levels!
  switch (msg.type) {
    case 'INIT': handleInit(msg); break;
    case 'ERROR': showError(msg.message); break;
  }
});
```

Web-side sending:
```javascript
window.parent.postMessage({ type: 'SUBMIT', payload: data }, '*');
```

---

## Context object

```typescript
context.userId          // string | undefined — undefined if logged out
context.username        // string | undefined
context.subredditId     // string
context.subredditName   // string | undefined
context.postId          // string | undefined
context.appSlug         // string

context.redis           // RedisClient
context.reddit          // RedditAPIClient
context.scheduler       // Scheduler
context.settings        // SettingsClient
context.ui              // UIClient (Blocks/render context only)
context.kvStore         // KVStore
context.realtime        // RealtimeClient
```

**Important:** In scheduler/trigger `onRun` / `onEvent`, context is `JobContext` — it omits `ui`, `dimensions`, `modLog`.

---

## Redis client

### Strings
```typescript
await context.redis.get(key)                          // → string | undefined
await context.redis.set(key, value)
await context.redis.set(key, value, { ex: 3600 })     // with TTL in seconds
await context.redis.set(key, value, { nx: true })     // only if not exists
await context.redis.del(key1, key2, ...)
await context.redis.exists(key1, key2, ...)           // → number
await context.redis.expire(key, seconds)
await context.redis.mGet([key1, key2])                // → (string | null)[]
await context.redis.mSet({ key1: val1, key2: val2 })
await context.redis.incrBy(key, amount)               // → number
await context.redis.keys(pattern)                     // → string[]

// Store/retrieve objects:
await context.redis.set(key, JSON.stringify(obj));
const obj = JSON.parse(await context.redis.get(key) ?? 'null');
```

### Hashes
```typescript
await context.redis.hSet(key, { field1: 'v1', field2: 'v2' })
await context.redis.hGet(key, field)                  // → string | undefined
await context.redis.hGetAll(key)                      // → Record<string, string>
await context.redis.hMGet(key, [field1, field2])      // → (string | null)[]
await context.redis.hDel(key, [field1])
await context.redis.hIncrBy(key, field, amount)       // → number
await context.redis.hLen(key)
await context.redis.hKeys(key)                        // → string[]
```

### Sorted Sets (leaderboards, ranked data)
```typescript
await context.redis.zAdd(key, { member: 'user1', score: 100 }, { member: 'user2', score: 200 })
await context.redis.zScore(key, member)               // → number | undefined
await context.redis.zRank(key, member)                // → number | undefined (0-based, ascending)
await context.redis.zRank(key, member, { reverse: true })  // rank from top

await context.redis.zRange(key, 0, 9)                 // → string[] (ascending)
await context.redis.zRange(key, 0, 9, { reverse: true })
await context.redis.zRange(key, 0, 9, { withScores: true })  // → ZMember[]
await context.redis.zRange(key, minScore, maxScore, { byScore: true })

await context.redis.zCard(key)                        // → number
await context.redis.zIncrBy(key, member, amount)      // → new score
await context.redis.zRem(key, [member1, member2])
await context.redis.zRemRangeByScore(key, min, max)
```

### Lists
```typescript
await context.redis.lPush(key, value1, value2)        // prepend
await context.redis.rPush(key, value1, value2)        // append
await context.redis.lRange(key, 0, -1)                // → string[] (all items)
await context.redis.lLen(key)
await context.redis.lPop(key)
await context.redis.rPop(key)
```

### Transactions
```typescript
const txn = await context.redis.watch(watchedKey);
await txn.multi();
await txn.incrBy('counter', 1);
await txn.set('flag', 'done');
const results = await txn.exec();  // null if watched key changed (retry needed)
```

### Global scope (cross-installation)
```typescript
await context.redis.global.set(key, value);
await context.redis.global.get(key);
```

---

## Scheduler client

```typescript
// Register first:
Devvit.addSchedulerJob({ name: 'myJob', onRun: async (event, context) => { ... } });

// One-time job:
const jobId = await context.scheduler.runJob({
  name: 'myJob',
  data: { userId: '123' },
  runAt: new Date(Date.now() + 60_000),
});

// Recurring cron job:
const jobId = await context.scheduler.runJob({
  name: 'dailyReset',
  cron: '0 0 * * *',   // midnight ET every day
});
```

Cron format: `minute hour dayOfMonth month dayOfWeek`
All times are **America/New_York (ET)**.

```
'0 0 * * *'      every day at midnight
'0 9 * * *'      every day at 9 AM
'0 9 * * 1-5'    weekdays at 9 AM
'*/15 * * * *'   every 15 minutes
'0 20 * * 0'     every Sunday at 8 PM
```

```typescript
await context.scheduler.cancelJob(jobId);
const jobs = await context.scheduler.listJobs();
```

---

## Reddit API client

### Posts
```typescript
const post = await context.reddit.getPostById('t3_abc123');
const posts = await context.reddit.getNewPosts({ subredditName: 'sub', limit: 10 }).all();
const posts = await context.reddit.getTopPosts({ subredditName: 'sub', time: 'day', limit: 10 }).all();
// time: 'hour' | 'day' | 'week' | 'month' | 'year' | 'all'

const newPost = await context.reddit.submitPost({
  subredditName: context.subredditName!,
  title: 'Title',
  text: 'Body text',
  // OR preview: <JSX/>,
});

await post.remove();
await post.approve();
await post.lock();
await post.sticky({ position: 1 });
```

### Comments
```typescript
const comment = await context.reddit.getCommentById('t1_abc123');
const comments = await post.getComments({ sort: 'top', limit: 20 }).all();
await post.reply({ text: 'comment text' });
await comment.reply({ text: 'reply' });
await comment.remove();
await comment.distinguish({ how: 'yes', sticky: true });
```

### Users
```typescript
const user = await context.reddit.getUserById('t2_abc123');
const user = await context.reddit.getUserByName('username');
user.id / user.username / user.linkKarma / user.commentKarma / user.createdAt
```

### Private messages
```typescript
await context.reddit.sendPrivateMessage({
  to: 'username',
  subject: 'Subject line',
  text: 'Message body (markdown ok)',
});
```

### Subreddit
```typescript
const sub = await context.reddit.getSubredditByName('name');
const sub = await context.reddit.getCurrentSubreddit();
await sub.banUser({ username: 'spammer', reason: 'Spam', duration: 7 });
```

---

## UIClient

Only available in Blocks render context. Not in scheduler jobs or triggers.

```typescript
context.ui.showToast('Message');
context.ui.showToast({ text: 'Message', appearance: 'success' });  // 'neutral' | 'success' | 'danger'
context.ui.showForm(formKey, optionalDefaultValues);
context.ui.navigateTo(post);
context.ui.navigateTo('https://url');
```

---

## Triggers

```typescript
// Events: AppInstall, AppUpgrade, PostSubmit, PostCreate, PostDelete, PostReport,
//         CommentSubmit, CommentCreate, CommentDelete, CommentReport, ModAction

Devvit.addTrigger({
  event: 'PostSubmit',
  onEvent: async (event, context) => {
    // event.post, event.author
  },
});
```

---

## Realtime

```typescript
// Server: publish
await context.realtime.send('my-channel', { type: 'UPDATE', value: 42 });

// Client (in render): subscribe
const channel = useChannel({
  name: 'my-channel',
  onMessage: (msg) => { /* update UI */ },
});
channel.subscribe();
```

---

## devvit.yaml

```yaml
name: my-app
version: 0.1.0
capabilities:
  - redis
  - redditAPI
  - http
scheduledJobs:
  - name: dailyReset
    description: Resets daily state
```
