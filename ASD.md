# KGS01 Authorization State Desynchronization(ASD)

## What is happening

The root cause is in `GuiServerDevice.cc`. The method `readAsyncHash` registers a persistent asynchronous callback loop for each client connection. At registration time, the current authorization level is bound into the callback as a captured `readOnly` boolean:

```
channel->readAsyncHash(
    bind_weak(&GuiServerDevice::onRead, this, _1, channel, _2, readOnly)
);
```

At the end of each invocation, `onRead` re-registers itself with the same captured value, creating a self-perpetuating loop where `readOnly` is fixed for the lifetime of the connection:

```
chan->readAsyncHash(
    bind_weak(&GuiServerDevice::onRead, this, _1, channel, _2, readOnly)
);
```

Write-level commands are gated on this value:

```
if (readOnly && violatesReadOnly(command)) {
    reject();
}
```

If `readOnly` was `false` at initial login, this condition never triggers, regardless of what happens to the session afterward.

## Why login-over-login breaks enforcement

When a client sends a second login request on the same open connection, the server sets a flag and skips re-registration of the loop entirely:

```
if (!isLoginOverLogin) {
    channel->readAsyncHash(..., readOnly);
}
```

The reasoning is that a loop is already running, so a new one does not need to be created.
This is correct in terms of avoiding a duplicate loop, but it means the new `readOnly` value computed for the second login is never propagated anywhere the running loop can see it. The new state is stored in session metadata and returned to the client, but the execution context that actually enforces commands keeps using the original captured value.

The two things that were supposed to agree, what the server tells the client its permissions are, and what the server actually enforces are now different.

## Why the existing session state does not help

The updated authorization level is written to the session record in `m_channels` during the login-over-login flow. That record is accurate. The problem is that `onRead` never reads from it. It uses the captured parameter instead. The session record and the enforcement context are updated independently, and only one of them is wired into the decision that matters.

## Proof of concept

The behavior can be reproduced without a live system by modeling the captured state directly:

```
def make_loop(read_only):
    def on_read(cmd):
        writes = {"reconfigure", "execute", "initDevice", "killDevice", "killServer"}
        if read_only and cmd in writes:
            return "blocked"
        return "executed"
    return on_read

# client logs in with write access
loop = make_loop(read_only=False)

# client issues a second login without credentials
# server computes new_read_only=True and tells the client
# server does not update the loop
new_read_only = True

print(loop("execute"))       # executed
print(loop("killDevice"))    # executed
print(loop("reconfigure"))   # executed
```

Against a live deployment, the sequence is:

1. authenticate with a valid one-time token. server registers the loop with `readOnly=False`
2. send a second login on the same connection without a token. server responds with a read-only session confirmation
3. send `execute`, `reconfigure`, `killDevice`, or `killServer`  commands are accepted and forwarded to the backend

Audit logs record a read-only session throughout step 3. The backend sees write operations.

## What this does and does not allow

This does not allow a client to acquire write access it was never granted. It allows write access that was legitimately granted to persist after the server has explicitly revoked it. The window is indefinite — it lasts until the connection is closed.

The audit log discrepancy is a secondary consequence. A forensic review of the logs would show a read-only session that somehow caused hardware or device state changes, with no record of how.

## Fix

`readOnly` should not be a captured parameter at all. `onRead` should retrieve the current authorization level from `m_channels` at the start of each invocation:

```
bool readOnly;
{
    std::lock_guard<std::mutex> lock(m_channelMutex);
    auto it = m_channels.find(channel);
    if (it == m_channels.end()) {
        return; // channel already gone
    }
    readOnly = it->second.readOnly;
}
```

This makes `m_channels` the single source of truth for authorization state. Any change to the session — login-over-login, token expiry, explicit downgrade — is immediately reflected in what `onRead` enforces, without needing to touch the callback registration logic.
