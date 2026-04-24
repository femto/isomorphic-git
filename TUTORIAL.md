[English](./TUTORIAL.md) | [中文](#中文教程)

# Learn isomorphic-git -- Git in JavaScript, Everywhere

## Git is a Data Structure. isomorphic-git Gives You the API.

Before we write code, let's get one thing straight.

**Git is not a command-line tool -- it's a content-addressable filesystem with a version control system built on top.** Every commit is a snapshot. Every branch is a pointer. Every remote is just another repository. isomorphic-git gives you programmatic access to all of it -- in Node.js, in the browser, anywhere JavaScript runs.

### Why isomorphic-git?

The traditional git binary is a native executable. It works great on servers and developer machines. But what about:

- **Browser-based IDEs?** No native binaries in the browser.
- **Serverless functions?** Cold starts penalize heavy dependencies.
- **Electron apps?** Ship one binary, not two.
- **Edge runtimes?** V8 isolates don't run native code.
- **WebContainers?** Pure JavaScript environments.

isomorphic-git solves all of these. It's a pure JavaScript implementation of git that:

```
- Runs in Node.js (with node:fs)
- Runs in browsers (with LightningFS or any IndexedDB wrapper)
- Runs in workers, service workers, edge functions
- Speaks the git wire protocol over HTTP
- Supports custom filesystems and HTTP clients
```

The API is the same everywhere. The backends are pluggable. **Bring your own fs, bring your own http.**

### What You'll Learn

This tutorial teaches isomorphic-git through 7 progressive sessions:

```
s01  Clone & Init      -- get repositories onto your filesystem
s02  Status & Staging  -- track changes, stage files
s03  Commits & History -- create snapshots, read the log
s04  Branches          -- create, switch, merge
s05  Remotes           -- fetch, pull, push
s06  Custom Backends   -- plug in your own fs and http
s07  Browser           -- full git in the browser with LightningFS
```

Each session builds on the last. Each concept is demonstrated in both Node.js and browser contexts.

---

```
                    THE ISOMORPHIC-GIT PATTERN
                    ===========================

    fs + http + dir --> git.* functions --> repository state

    Required for most operations:
    - fs: filesystem (node:fs or LightningFS)
    - dir: working directory path
    - http: HTTP client (for network operations)

    The fs and http are injected, not assumed.
    This is what makes it isomorphic.
```

**7 progressive sessions, from clone to browser.**
**Each session has one motto. Each concept builds on the last.**

> **s01** &nbsp; *"One fs + one clone = a local repo"* &mdash; clone, init, and the fs injection pattern
>
> **s02** &nbsp; *"Track changes, not just files"* &mdash; status, statusMatrix, add, remove
>
> **s03** &nbsp; *"Commits are immutable snapshots"* &mdash; commit, log, readCommit
>
> **s04** &nbsp; *"Branches are just pointers"* &mdash; branch, checkout, merge, currentBranch
>
> **s05** &nbsp; *"Sync with remotes"* &mdash; fetch, pull, push, authentication
>
> **s06** &nbsp; *"Bring your own backends"* &mdash; custom fs, custom http, virtual filesystems
>
> **s07** &nbsp; *"Git runs everywhere JavaScript runs"* &mdash; LightningFS, browser integration, service workers

---

## s01 -- Clone & Init

### *"One fs + one clone = a local repo"*

#### What You'll Learn

- The `fs` injection pattern -- why isomorphic-git doesn't import fs
- Cloning a remote repository
- Initializing a new repository
- The minimal parameters every git operation needs

#### The Pattern

Every isomorphic-git function takes an options object. Most operations require at least:

```javascript
{
  fs,      // filesystem implementation
  dir,     // working directory path
  // ... operation-specific options
}
```

The `fs` is never assumed. You inject it. This is what makes the library isomorphic.

#### Node.js Example

```javascript
import git from 'isomorphic-git';
import http from 'isomorphic-git/http/node';
import fs from 'node:fs';

// Clone a repository
await git.clone({
  fs,
  http,
  dir: '/tmp/my-repo',
  url: 'https://github.com/isomorphic-git/isomorphic-git',
  depth: 1,                    // shallow clone -- faster
  singleBranch: true,          // only fetch default branch
  onProgress: (event) => {
    console.log(`${event.phase}: ${event.loaded}/${event.total}`);
  },
});

console.log('Cloned!');

// Initialize a new repository
await git.init({
  fs,
  dir: '/tmp/new-repo',
  defaultBranch: 'main',       // set default branch name
});

console.log('Initialized!');
```

#### Browser Example

```javascript
import git from 'isomorphic-git';
import http from 'isomorphic-git/http/web';
import LightningFS from '@isomorphic-git/lightning-fs';

// Create a filesystem backed by IndexedDB
const fs = new LightningFS('my-app');

// Clone a repository
await git.clone({
  fs,
  http,
  dir: '/repo',
  url: 'https://github.com/isomorphic-git/isomorphic-git',
  corsProxy: 'https://cors.isomorphic-git.org',  // needed for cross-origin
  depth: 1,
  singleBranch: true,
});

console.log('Cloned in browser!');
```

#### Key Takeaways

```
✓ Always inject fs -- never import it inside the library
✓ Use isomorphic-git/http/node for Node.js, /http/web for browsers
✓ dir is the working directory path (absolute or relative)
✓ depth: 1 + singleBranch: true = fast shallow clones
✓ Browser needs corsProxy for cross-origin clones
```

---

## s02 -- Status & Staging

### *"Track changes, not just files"*

#### What You'll Learn

- Check file status (modified, staged, untracked)
- The statusMatrix -- batch status for entire repo
- Add files to the staging area (index)
- Remove files from tracking

#### Understanding Status

Git tracks files in three trees:

```
HEAD (last commit) <--> Index (staging area) <--> Workdir (filesystem)

Status tells you the relationship between these three trees.
```

#### Node.js Example

```javascript
import git from 'isomorphic-git';
import fs from 'node:fs';

const dir = '/tmp/my-repo';

// Check status of a single file
const status = await git.status({
  fs,
  dir,
  filepath: 'README.md',
});

console.log('README.md status:', status);
// Possible values: 'ignored', 'unmodified', '*modified', '*deleted',
//                  '*added', 'absent', 'modified', 'deleted', 'added'
// * prefix means: staged for that change

// Get status matrix for entire repo (much faster than checking each file)
const matrix = await git.statusMatrix({
  fs,
  dir,
});

// statusMatrix returns: [filepath, HEAD, WORKDIR, STAGE]
// Values: 0 = absent, 1 = present unchanged, 2 = present changed
for (const [filepath, head, workdir, stage] of matrix) {
  if (head !== workdir || head !== stage) {
    console.log(`${filepath}: HEAD=${head} WORKDIR=${workdir} STAGE=${stage}`);
  }
}

// Add a file to the index
await git.add({
  fs,
  dir,
  filepath: 'newfile.txt',
});

// Add all files matching a pattern
const files = await fs.promises.readdir(dir);
for (const file of files) {
  if (file.endsWith('.js')) {
    await git.add({ fs, dir, filepath: file });
  }
}

// Remove a file from the index (and working directory)
await git.remove({
  fs,
  dir,
  filepath: 'oldfile.txt',
});
```

#### Browser Example

```javascript
import git from 'isomorphic-git';
import LightningFS from '@isomorphic-git/lightning-fs';

const fs = new LightningFS('my-app');
const dir = '/repo';

// Write a new file
await fs.promises.writeFile('/repo/hello.txt', 'Hello, World!');

// Check its status
const status = await git.status({
  fs,
  dir,
  filepath: 'hello.txt',
});
console.log('hello.txt:', status);  // '*added' (new file, not staged)

// Stage it
await git.add({
  fs,
  dir,
  filepath: 'hello.txt',
});

// Check again
const newStatus = await git.status({
  fs,
  dir,
  filepath: 'hello.txt',
});
console.log('hello.txt:', newStatus);  // 'added' (staged)
```

#### The statusMatrix Decoded

```
statusMatrix returns: [filepath, HEAD, WORKDIR, STAGE]

HEAD    = status in last commit
WORKDIR = status in working directory
STAGE   = status in staging area (index)

Values:
  0 = absent
  1 = identical to HEAD
  2 = different from HEAD

Common patterns:
  [file, 1, 1, 1] = unmodified
  [file, 1, 2, 1] = modified, not staged
  [file, 1, 2, 2] = modified, staged
  [file, 0, 2, 0] = new file, not staged
  [file, 0, 2, 2] = new file, staged
  [file, 1, 0, 1] = deleted, not staged
  [file, 1, 0, 0] = deleted, staged
```

#### Key Takeaways

```
✓ status() checks one file; statusMatrix() checks everything
✓ statusMatrix is much faster for repos with many files
✓ add() stages changes; remove() unstages AND deletes
✓ The three trees: HEAD, Index (stage), Workdir
✓ Status values tell you what changed and whether it's staged
```

---

## s03 -- Commits & History

### *"Commits are immutable snapshots"*

#### What You'll Learn

- Create commits with author/committer info
- Read commit history with git.log
- Inspect individual commits with git.readCommit
- Understand commit objects and their structure

#### The Commit Object

```
Every commit contains:
  - tree:      SHA of the root tree (directory snapshot)
  - parent:    SHA(s) of parent commit(s)
  - author:    name, email, timestamp
  - committer: name, email, timestamp
  - message:   commit message

Commits are immutable. Once created, they never change.
The SHA is a hash of all this content.
```

#### Node.js Example

```javascript
import git from 'isomorphic-git';
import fs from 'node:fs';

const dir = '/tmp/my-repo';

// Create a commit
const sha = await git.commit({
  fs,
  dir,
  message: 'Add new feature',
  author: {
    name: 'Your Name',
    email: 'you@example.com',
  },
  // committer defaults to author if not specified
});

console.log('Created commit:', sha);

// Read commit history
const commits = await git.log({
  fs,
  dir,
  depth: 10,  // limit to 10 commits
});

for (const commit of commits) {
  console.log(`${commit.oid.slice(0, 7)} ${commit.commit.message.split('\n')[0]}`);
  console.log(`  Author: ${commit.commit.author.name} <${commit.commit.author.email}>`);
  console.log(`  Date: ${new Date(commit.commit.author.timestamp * 1000).toISOString()}`);
}

// Read a specific commit
const commitData = await git.readCommit({
  fs,
  dir,
  oid: sha,
});

console.log('Commit details:', {
  tree: commitData.commit.tree,
  parent: commitData.commit.parent,
  message: commitData.commit.message,
});

// Log from a specific branch or ref
const mainCommits = await git.log({
  fs,
  dir,
  ref: 'main',
  depth: 5,
});
```

#### Browser Example

```javascript
import git from 'isomorphic-git';
import LightningFS from '@isomorphic-git/lightning-fs';

const fs = new LightningFS('my-app');
const dir = '/repo';

// Stage and commit a file
await fs.promises.writeFile('/repo/app.js', 'console.log("Hello");');
await git.add({ fs, dir, filepath: 'app.js' });

const sha = await git.commit({
  fs,
  dir,
  message: 'Initial commit',
  author: {
    name: 'Browser User',
    email: 'browser@example.com',
  },
});

console.log('Committed:', sha);

// Show history
const log = await git.log({ fs, dir });
log.forEach(entry => {
  console.log(`${entry.oid.slice(0, 7)} - ${entry.commit.message}`);
});
```

#### Advanced: Walking the Commit Graph

```javascript
// Get all commits reachable from HEAD
const allCommits = await git.log({
  fs,
  dir,
  // no depth limit = all commits
});

// Find commits by a specific author
const myCommits = allCommits.filter(
  c => c.commit.author.email === 'me@example.com'
);

// Find merge commits (commits with multiple parents)
const merges = allCommits.filter(
  c => c.commit.parent.length > 1
);
```

#### Key Takeaways

```
✓ commit() requires staged changes and author info
✓ log() returns commits from newest to oldest
✓ readCommit() gives you the full commit object
✓ oid is the commit SHA (40 hex characters)
✓ author.timestamp is Unix time (seconds since epoch)
✓ Use depth to limit how many commits to fetch
```

---

## s04 -- Branches

### *"Branches are just pointers"*

#### What You'll Learn

- List, create, and delete branches
- Switch between branches with checkout
- Get the current branch name
- Merge branches together

#### What Is a Branch?

```
A branch is a pointer to a commit. That's it.

main -----> abc123 (commit)
feature --> def456 (commit)

When you commit, the current branch moves forward.
When you create a branch, you create a new pointer.
When you merge, you may create a merge commit or fast-forward.
```

#### Node.js Example

```javascript
import git from 'isomorphic-git';
import fs from 'node:fs';

const dir = '/tmp/my-repo';

// Get current branch
const current = await git.currentBranch({
  fs,
  dir,
  fullname: false,  // 'main' not 'refs/heads/main'
});
console.log('Current branch:', current);

// List all branches
const branches = await git.listBranches({
  fs,
  dir,
});
console.log('Branches:', branches);

// Create a new branch
await git.branch({
  fs,
  dir,
  ref: 'feature-x',
  // checkout: true,  // optionally switch to it immediately
});

// Switch to the new branch
await git.checkout({
  fs,
  dir,
  ref: 'feature-x',
});

// Make some changes and commit on the feature branch
await fs.promises.writeFile(`${dir}/feature.txt`, 'New feature code');
await git.add({ fs, dir, filepath: 'feature.txt' });
await git.commit({
  fs,
  dir,
  message: 'Add feature',
  author: { name: 'Dev', email: 'dev@example.com' },
});

// Switch back to main
await git.checkout({
  fs,
  dir,
  ref: 'main',
});

// Merge feature-x into main
const mergeResult = await git.merge({
  fs,
  dir,
  ours: 'main',
  theirs: 'feature-x',
  author: { name: 'Dev', email: 'dev@example.com' },
});

console.log('Merge result:', mergeResult);
// { oid: '...', alreadyMerged: false, fastForward: true/false, ... }

// Delete the feature branch (after merge)
await git.deleteBranch({
  fs,
  dir,
  ref: 'feature-x',
});
```

#### Browser Example

```javascript
import git from 'isomorphic-git';
import LightningFS from '@isomorphic-git/lightning-fs';

const fs = new LightningFS('my-app');
const dir = '/repo';

// Create and switch to a branch in one step
await git.branch({
  fs,
  dir,
  ref: 'experiment',
  checkout: true,
});

console.log('Now on:', await git.currentBranch({ fs, dir }));

// Do work...
await fs.promises.writeFile('/repo/experiment.js', '// experimental code');
await git.add({ fs, dir, filepath: 'experiment.js' });
await git.commit({
  fs,
  dir,
  message: 'Experimental changes',
  author: { name: 'Tester', email: 'test@example.com' },
});

// Switch back
await git.checkout({ fs, dir, ref: 'main' });
```

#### Handling Merge Conflicts

```javascript
try {
  await git.merge({
    fs,
    dir,
    ours: 'main',
    theirs: 'feature',
    author: { name: 'Dev', email: 'dev@example.com' },
  });
} catch (err) {
  if (err.code === 'MergeConflictError') {
    console.log('Conflicts in:', err.data.filepaths);
    // You need to resolve these manually:
    // 1. Read the conflicted files
    // 2. Edit them to resolve conflicts
    // 3. Stage the resolved files
    // 4. Commit the merge
  } else {
    throw err;
  }
}
```

#### Key Takeaways

```
✓ Branches are lightweight -- just pointers to commits
✓ currentBranch() tells you where HEAD points
✓ checkout() moves HEAD and updates the working directory
✓ merge() can fast-forward or create a merge commit
✓ Merge conflicts throw MergeConflictError with file list
✓ Delete branches after merging to keep things clean
```

---

## s05 -- Remotes

### *"Sync with remotes"*

#### What You'll Learn

- Fetch changes from remote repositories
- Pull (fetch + merge) in one step
- Push local commits to remotes
- Authentication with tokens and credentials

#### The Remote Model

```
Local repo <---fetch/pull--- Remote repo
Local repo ----push-------> Remote repo

Remote = URL + name (usually 'origin')
Fetch = download commits and refs, don't merge
Pull = fetch + merge into current branch
Push = upload commits and update remote refs
```

#### Node.js Example

```javascript
import git from 'isomorphic-git';
import http from 'isomorphic-git/http/node';
import fs from 'node:fs';

const dir = '/tmp/my-repo';

// Add a remote (if not already from clone)
await git.addRemote({
  fs,
  dir,
  remote: 'origin',
  url: 'https://github.com/user/repo',
});

// List remotes
const remotes = await git.listRemotes({ fs, dir });
console.log('Remotes:', remotes);
// [{ remote: 'origin', url: 'https://...' }]

// Fetch from remote
await git.fetch({
  fs,
  http,
  dir,
  remote: 'origin',
  ref: 'main',
  depth: 10,
  onProgress: (e) => console.log(`Fetching: ${e.phase}`),
  onAuth: () => ({ username: process.env.GITHUB_TOKEN }),
});

// Pull (fetch + merge)
await git.pull({
  fs,
  http,
  dir,
  remote: 'origin',
  ref: 'main',
  author: { name: 'Dev', email: 'dev@example.com' },
  onAuth: () => ({ username: process.env.GITHUB_TOKEN }),
});

// Push to remote
const pushResult = await git.push({
  fs,
  http,
  dir,
  remote: 'origin',
  ref: 'main',
  onAuth: () => ({ username: process.env.GITHUB_TOKEN }),
});

console.log('Push result:', pushResult);
// { ok: true, refs: { 'refs/heads/main': { ... } } }
```

#### Authentication Patterns

```javascript
// Pattern 1: GitHub token (recommended)
onAuth: () => ({
  username: process.env.GITHUB_TOKEN,
  // password not needed for token auth
})

// Pattern 2: Username + password
onAuth: () => ({
  username: 'your-username',
  password: 'your-password',  // or personal access token
})

// Pattern 3: Interactive prompt
onAuth: (url) => {
  // Return credentials or prompt user
  return {
    username: prompt('Username for ' + url),
    password: prompt('Password:'),
  };
}

// Pattern 4: Cancel auth
onAuth: () => ({ cancel: true })

// Pattern 5: Handle auth failure and retry
onAuthFailure: ({ url, auth }) => {
  console.log('Auth failed for', url);
  // Return new credentials to retry, or undefined to fail
  return {
    username: 'new-token',
  };
}
```

#### Browser Example

```javascript
import git from 'isomorphic-git';
import http from 'isomorphic-git/http/web';
import LightningFS from '@isomorphic-git/lightning-fs';

const fs = new LightningFS('my-app');
const dir = '/repo';

// Fetch with CORS proxy (required for cross-origin)
await git.fetch({
  fs,
  http,
  dir,
  corsProxy: 'https://cors.isomorphic-git.org',
  url: 'https://github.com/user/repo',
  ref: 'main',
  onAuth: () => ({
    username: localStorage.getItem('github_token'),
  }),
});

// Push (requires auth)
await git.push({
  fs,
  http,
  dir,
  corsProxy: 'https://cors.isomorphic-git.org',
  remote: 'origin',
  ref: 'main',
  onAuth: () => ({
    username: localStorage.getItem('github_token'),
  }),
});
```

#### Key Takeaways

```
✓ fetch() downloads but doesn't merge
✓ pull() = fetch() + merge() in one call
✓ push() uploads local commits to remote
✓ onAuth callback provides credentials on demand
✓ GitHub tokens work as username with empty password
✓ Browser needs corsProxy for cross-origin requests
✓ onAuthFailure lets you retry with different credentials
```

---

## s06 -- Custom Backends

### *"Bring your own backends"*

#### What You'll Learn

- Why isomorphic-git uses dependency injection
- Creating custom filesystem implementations
- Creating custom HTTP clients
- Virtual filesystems and in-memory git

#### The Injection Pattern

```
isomorphic-git doesn't import 'fs' or 'http'.
You inject them. This is the key to portability.

Why?
- Node.js has node:fs, browsers have IndexedDB
- Node.js has node:http, browsers have fetch
- Edge runtimes have neither
- You might want in-memory, or S3, or a database

Solution: dependency injection via options object.
```

#### Custom Filesystem Interface

```javascript
// Your fs must implement these methods (Promise-based):
const customFs = {
  promises: {
    readFile(path, options) { /* return contents */ },
    writeFile(path, data, options) { /* write data */ },
    unlink(path) { /* delete file */ },
    readdir(path) { /* return array of names */ },
    mkdir(path, options) { /* create directory */ },
    rmdir(path) { /* remove directory */ },
    stat(path) { /* return { isFile(), isDirectory(), ... } */ },
    lstat(path) { /* like stat but for symlinks */ },
    readlink(path) { /* read symlink target */ },
    symlink(target, path) { /* create symlink */ },
    chmod(path, mode) { /* change permissions */ },
  },
};
```

#### Node.js Example: In-Memory Filesystem

```javascript
import git from 'isomorphic-git';
import http from 'isomorphic-git/http/node';
import { Volume, createFsFromVolume } from 'memfs';

// Create an in-memory filesystem
const vol = new Volume();
const fs = createFsFromVolume(vol);

// Use it with isomorphic-git
await git.clone({
  fs,
  http,
  dir: '/repo',
  url: 'https://github.com/user/repo',
  depth: 1,
});

// The entire repo is in memory!
const files = await fs.promises.readdir('/repo');
console.log('Files in memory:', files);

// Useful for:
// - Testing without touching disk
// - Serverless functions (fast cold starts)
// - Temporary operations
```

#### Custom HTTP Client

```javascript
// Your http must implement the request function:
const customHttp = {
  async request({ url, method, headers, body, onProgress }) {
    // Make the HTTP request however you want
    const response = await fetch(url, {
      method,
      headers,
      body,
    });

    return {
      url: response.url,
      method,
      statusCode: response.status,
      statusMessage: response.statusText,
      headers: Object.fromEntries(response.headers),
      body: [new Uint8Array(await response.arrayBuffer())],
    };
  },
};

// Use it
await git.clone({
  fs,
  http: customHttp,
  dir: '/repo',
  url: 'https://github.com/user/repo',
});
```

#### Browser Example: Custom Cache Layer

```javascript
import git from 'isomorphic-git';
import http from 'isomorphic-git/http/web';
import LightningFS from '@isomorphic-git/lightning-fs';

const fs = new LightningFS('my-app');

// Wrap HTTP with caching
const cachedHttp = {
  async request(options) {
    const cacheKey = `git-cache:${options.url}:${options.method}`;

    // Check cache first (for GET requests)
    if (options.method === 'GET') {
      const cached = localStorage.getItem(cacheKey);
      if (cached) {
        const { timestamp, data } = JSON.parse(cached);
        if (Date.now() - timestamp < 60000) {  // 1 minute cache
          return data;
        }
      }
    }

    // Make the actual request
    const response = await http.request(options);

    // Cache the response
    if (options.method === 'GET' && response.statusCode === 200) {
      localStorage.setItem(cacheKey, JSON.stringify({
        timestamp: Date.now(),
        data: response,
      }));
    }

    return response;
  },
};

await git.fetch({
  fs,
  http: cachedHttp,
  dir: '/repo',
  corsProxy: 'https://cors.isomorphic-git.org',
  url: 'https://github.com/user/repo',
});
```

#### Key Takeaways

```
✓ isomorphic-git uses dependency injection for portability
✓ fs must implement the node:fs/promises interface
✓ http must implement a request() function
✓ memfs gives you in-memory git for testing
✓ LightningFS gives you IndexedDB-backed git for browsers
✓ You can wrap/extend the default implementations
✓ This is how git works in WebContainers, StackBlitz, etc.
```

---

## s07 -- Browser Integration

### *"Git runs everywhere JavaScript runs"*

#### What You'll Learn

- Setting up LightningFS for persistent browser storage
- Handling CORS for remote operations
- Progress reporting in UI
- Service worker integration

#### LightningFS Deep Dive

```
LightningFS = IndexedDB + filesystem API

- Persistent across page reloads
- Multiple tabs can share the same fs
- Each fs instance has a name (database name)
- Supports all operations isomorphic-git needs
```

#### Complete Browser Example

```html
<!DOCTYPE html>
<html>
<head>
  <title>Browser Git</title>
</head>
<body>
  <div id="output"></div>
  <button id="clone">Clone Repo</button>
  <button id="log">Show Log</button>
  <button id="status">Show Status</button>

  <script type="module">
    import git from 'https://unpkg.com/isomorphic-git@1.25.0';
    import http from 'https://unpkg.com/isomorphic-git@1.25.0/http/web/index.js';
    import LightningFS from 'https://unpkg.com/@isomorphic-git/lightning-fs@4.6.0';

    // Initialize filesystem
    const fs = new LightningFS('my-git-app');
    const dir = '/repo';

    // Helper to log to page
    const log = (msg) => {
      document.getElementById('output').innerHTML += msg + '<br>';
    };

    // Clone button
    document.getElementById('clone').onclick = async () => {
      log('Cloning...');

      try {
        await git.clone({
          fs,
          http,
          dir,
          url: 'https://github.com/isomorphic-git/isomorphic-git',
          corsProxy: 'https://cors.isomorphic-git.org',
          depth: 5,
          singleBranch: true,
          onProgress: (event) => {
            log(`${event.phase}: ${event.loaded || 0}/${event.total || '?'}`);
          },
        });

        log('Clone complete!');
      } catch (err) {
        log('Error: ' + err.message);
      }
    };

    // Log button
    document.getElementById('log').onclick = async () => {
      try {
        const commits = await git.log({ fs, dir, depth: 5 });

        log('<h3>Recent Commits:</h3>');
        for (const commit of commits) {
          log(`${commit.oid.slice(0, 7)} - ${commit.commit.message.split('\n')[0]}`);
        }
      } catch (err) {
        log('Error: ' + err.message);
      }
    };

    // Status button
    document.getElementById('status').onclick = async () => {
      try {
        const files = await fs.promises.readdir(dir);
        log('<h3>Files:</h3>');
        for (const file of files) {
          const status = await git.status({ fs, dir, filepath: file });
          log(`${file}: ${status}`);
        }
      } catch (err) {
        log('Error: ' + err.message);
      }
    };
  </script>
</body>
</html>
```

#### Service Worker Integration

```javascript
// sw.js - Service Worker with git capabilities
importScripts('https://unpkg.com/isomorphic-git@1.25.0/index.umd.min.js');
importScripts('https://unpkg.com/@isomorphic-git/lightning-fs@4.6.0/dist/lightning-fs.min.js');

const fs = new LightningFS('sw-git');

self.addEventListener('message', async (event) => {
  const { type, data } = event.data;

  if (type === 'clone') {
    await git.clone({
      fs,
      http: git.http,
      dir: '/repo',
      url: data.url,
      depth: 1,
    });
    event.ports[0].postMessage({ success: true });
  }

  if (type === 'readFile') {
    const content = await fs.promises.readFile(data.path, 'utf8');
    event.ports[0].postMessage({ content });
  }
});

// main.js - Communicate with service worker
async function cloneInWorker(url) {
  const channel = new MessageChannel();

  return new Promise((resolve) => {
    channel.port1.onmessage = (event) => resolve(event.data);
    navigator.serviceWorker.controller.postMessage(
      { type: 'clone', data: { url } },
      [channel.port2]
    );
  });
}
```

#### Progress Reporting UI

```javascript
// React/Vue/Svelte style progress component
const progress = {
  phase: '',
  loaded: 0,
  total: 0,
};

await git.clone({
  fs,
  http,
  dir: '/repo',
  url: 'https://github.com/user/repo',
  corsProxy: 'https://cors.isomorphic-git.org',
  onProgress: (event) => {
    progress.phase = event.phase;
    progress.loaded = event.loaded;
    progress.total = event.total;

    // Update UI
    updateProgressBar(progress);
  },
  onMessage: (message) => {
    // Server messages (from git server)
    console.log('Server:', message);
  },
});
```

#### Handling Large Repositories

```javascript
// For large repos, use streaming and shallow clones

// 1. Shallow clone with limited depth
await git.clone({
  fs,
  http,
  dir: '/repo',
  url: 'https://github.com/large/repo',
  depth: 1,
  singleBranch: true,
});

// 2. Fetch more history later if needed
await git.fetch({
  fs,
  http,
  dir: '/repo',
  depth: 10,  // fetch 10 more commits
  deepen: true,
});

// 3. Sparse checkout (only certain paths)
// Note: This requires manual implementation with read/write operations
```

#### Key Takeaways

```
✓ LightningFS provides persistent IndexedDB storage
✓ corsProxy is required for cross-origin git operations
✓ onProgress callback enables progress bars
✓ Service workers can run git in the background
✓ Shallow clones (depth: 1) are essential for browser
✓ singleBranch: true reduces download size
✓ The same API works in main thread and workers
```

---

## Quick Reference

### Installation

```bash
# Node.js
npm install isomorphic-git

# Browser (CDN)
# See s07 for script tag imports
```

### Import Patterns

```javascript
// Node.js (ESM)
import git from 'isomorphic-git';
import http from 'isomorphic-git/http/node';
import fs from 'node:fs';

// Node.js (CJS)
const git = require('isomorphic-git');
const http = require('isomorphic-git/http/node');
const fs = require('fs');

// Browser
import git from 'isomorphic-git';
import http from 'isomorphic-git/http/web';
import LightningFS from '@isomorphic-git/lightning-fs';
const fs = new LightningFS('my-app');
```

### Common Options Object

```javascript
{
  fs,                    // required: filesystem
  dir,                   // required: working directory
  http,                  // required for network ops
  url,                   // remote URL
  ref,                   // branch or tag name
  remote,                // remote name (default: 'origin')
  depth,                 // shallow clone depth
  singleBranch,          // only fetch one branch
  corsProxy,             // CORS proxy URL (browser)
  onProgress,            // progress callback
  onMessage,             // server message callback
  onAuth,                // auth callback
  onAuthFailure,         // auth failure callback
  author,                // { name, email, timestamp? }
  committer,             // { name, email, timestamp? }
  message,               // commit message
  filepath,              // path relative to dir
}
```

### Error Handling

```javascript
import { Errors } from 'isomorphic-git';

try {
  await git.push({ fs, http, dir, remote: 'origin' });
} catch (err) {
  if (err instanceof Errors.PushRejectedError) {
    console.log('Push rejected - pull first');
  } else if (err instanceof Errors.HttpError) {
    console.log('Network error:', err.statusCode);
  } else if (err instanceof Errors.MergeConflictError) {
    console.log('Conflicts:', err.data.filepaths);
  } else {
    throw err;
  }
}
```

---

## Architecture

```
isomorphic-git/
│
├── Core Git Operations
│   ├── clone, init          # Repository creation
│   ├── add, remove          # Staging
│   ├── commit, merge        # Writing commits
│   ├── log, readCommit      # Reading history
│   ├── status, statusMatrix # Working tree status
│   └── branch, checkout     # References
│
├── Network Operations
│   ├── fetch                # Download objects
│   ├── pull                 # Fetch + merge
│   └── push                 # Upload objects
│
├── HTTP Backends
│   ├── http/node            # Uses node:http
│   └── http/web             # Uses fetch API
│
└── Pluggable Filesystem
    ├── node:fs              # Native Node.js
    ├── LightningFS          # IndexedDB (browser)
    ├── memfs                # In-memory
    └── Your custom fs       # Anything you want
```

---

## Learning Path

```
Phase 1: LOCAL OPERATIONS              Phase 2: NETWORK
========================               =================
s01  Clone & Init             [2]      s05  Fetch/Pull/Push     [6]
     fs injection pattern                   remote operations
     |                                      authentication
     +-> s02  Status/Staging    [4]
              statusMatrix, add
              |
         s03  Commits & Log     [3]
              snapshots, history
              |
         s04  Branches          [5]
              pointers, merge

Phase 3: ADVANCED
=================
s06  Custom Backends          [7]
     dependency injection
     |
s07  Browser Integration      [8]
     LightningFS, CORS
     service workers

     [N] = number of APIs introduced
```

---

## Links

- **GitHub**: https://github.com/isomorphic-git/isomorphic-git
- **Documentation**: https://isomorphic-git.org
- **LightningFS**: https://github.com/isomorphic-git/lightning-fs
- **CORS Proxy**: https://cors.isomorphic-git.org

---

<a name="中文教程"></a>

# 中文教程

## isomorphic-git 学习指南 -- JavaScript 中的 Git

### 核心理念

**Git 不是命令行工具 -- 它是一个内容寻址文件系统，上面构建了版本控制系统。** isomorphic-git 让你在 Node.js 和浏览器中都能使用 Git。

### 七个渐进式章节

```
s01  "一个 fs + 一次 clone = 本地仓库"    -- clone, init, fs 注入模式
s02  "追踪变更，而非文件"                 -- status, statusMatrix, add, remove
s03  "提交是不可变的快照"                 -- commit, log, readCommit
s04  "分支只是指针"                       -- branch, checkout, merge
s05  "与远程同步"                         -- fetch, pull, push, 认证
s06  "自带后端"                           -- 自定义 fs 和 http
s07  "Git 在 JavaScript 能运行的地方都能运行" -- LightningFS, 浏览器集成
```

### 核心模式

```javascript
// Node.js
import git from 'isomorphic-git';
import http from 'isomorphic-git/http/node';
import fs from 'node:fs';

await git.clone({ fs, http, dir: '/repo', url: 'https://...' });

// 浏览器
import git from 'isomorphic-git';
import http from 'isomorphic-git/http/web';
import LightningFS from '@isomorphic-git/lightning-fs';

const fs = new LightningFS('my-app');
await git.clone({
  fs, http,
  dir: '/repo',
  url: 'https://...',
  corsProxy: 'https://cors.isomorphic-git.org'
});
```

### 要点总结

| 章节 | 主题 | 格言 |
|------|------|------|
| s01 | Clone 与 Init | *一个 fs + 一次 clone = 本地仓库* |
| s02 | Status 与 Staging | *追踪变更，而非文件* |
| s03 | Commits 与 Log | *提交是不可变的快照* |
| s04 | Branches | *分支只是指针* |
| s05 | Remotes | *与远程同步* |
| s06 | 自定义后端 | *自带后端* |
| s07 | 浏览器集成 | *Git 在 JavaScript 能运行的地方都能运行* |

### 快速开始

```bash
npm install isomorphic-git
npm install @isomorphic-git/lightning-fs  # 浏览器使用
```

### 关键 API

```javascript
// 仓库操作
git.clone({ fs, http, dir, url })
git.init({ fs, dir })

// 暂存区
git.status({ fs, dir, filepath })
git.statusMatrix({ fs, dir })
git.add({ fs, dir, filepath })
git.remove({ fs, dir, filepath })

// 提交
git.commit({ fs, dir, message, author })
git.log({ fs, dir, depth })

// 分支
git.branch({ fs, dir, ref })
git.checkout({ fs, dir, ref })
git.merge({ fs, dir, ours, theirs, author })

// 远程
git.fetch({ fs, http, dir, remote })
git.pull({ fs, http, dir, remote, author })
git.push({ fs, http, dir, remote })
```

---

**isomorphic-git: 依赖注入让 Git 无处不在。**

**fs 和 http 是你注入的，不是导入的。这就是同构的秘密。**
