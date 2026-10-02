# MessageChannel message posted while entering bfcache is never delivered (Firefox)

A message posted to a `MessagePort` while a page goes into the back/forward cache - during `pagehide` with `persisted: true`, or during the `focusout` Firefox fires right after it - is never delivered, neither before the page is frozen nor after it is restored. A `setTimeout` scheduled at the same moment does fire after the restore, and new messages posted on the same channel after the restore are delivered as usual.

Chromium, with bfcache, delivers the message.

Live testcase: <https://stefanprobst.github.io/firefox-bfcache-messagechannel/>

## Steps to reproduce

1. Open <https://stefanprobst.github.io/firefox-bfcache-messagechannel/> in Firefox (or serve this directory, e.g. `python3 -m http.server 8000`, and open <http://localhost:8000/>).
2. Click "Leave to another page (same origin)" (the cross-origin link to example.com shows the same).
3. Press the browser's Back button.
4. Wait a second for the result.

### Expected

> PASS: every message posted before entering bfcache was delivered.

The log shows `message #1 (posted during "pagehide") delivered`, either before the page is frozen or after it is restored.

### Actual

> FAIL: message(s) #1, #2 posted before entering bfcache were never delivered.

```
    31 ms  pageshow (persisted: false)
  1254 ms  pagehide (persisted: true)
  1254 ms  message #1 posted during "pagehide" (document.visibilityState: visible)
  1254 ms  message #2 posted during "focusout" (document.visibilityState: hidden)
  5268 ms  pageshow (persisted: true)
  5271 ms  setTimeout(0) scheduled during "pagehide" fired
  6283 ms  after restore: 2 posted, 0 delivered
```

The "Post another message on the same channel" button shows that the channel itself still works after the restore - only the messages in flight when the page was frozen are lost.

Tested with Firefox 156.0.1 on Linux, fresh profile with default settings. Message #2 (`focusout`) only appears when the clicked link had focus, which is the case when clicking it with a mouse.

## Real-world impact

React's scheduler queues its work loop by posting a message to a `MessageChannel`, and only posts again once that message has been received. When a React app updates state in a `blur`/`focusout` handler - which react-aria components do, to track focus - the update lands after `pagehide`, the scheduler's message is lost, and after the restore the scheduler believes its work loop is still running. It never runs scheduled work again: synchronous updates still render, but transitions never do, which in Next.js means every client-side navigation silently does nothing until the page is reloaded.

- Reported first to react-aria: <https://github.com/adobe/react-spectrum/issues/10459> (repro: <https://github.com/stefanprobst/issue-rac-next-link-bfcache-firefox>)
- React's scheduler: <https://github.com/facebook/react/blob/main/packages/scheduler/src/forks/Scheduler.js> (`schedulePerformWorkUntilDeadline`, `isMessageLoopRunning`)
