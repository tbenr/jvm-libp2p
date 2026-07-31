# QUIC Dial Cancellation Design

## Problem

`QuicTransport.dial()` binds an ephemeral UDP channel and then starts the QUIC
handshake. The returned `CompletableFuture<Connection>` is a dependent future,
so cancelling it does not cancel the bind or handshake futures upstream.

The current cancellation handler waits for `quicConnFuture` to complete
successfully before closing the established `QuicChannel`. This handles
cancellation against a responsive QUIC peer, but it does not promptly release
the UDP channel while a handshake is still pending. A non-responsive peer can
therefore retain the socket and its Netty resources until QUIC fails or times
out.

The existing regression test does not observe this state. It cancels
immediately and waits only while `activeConnections` is greater than zero.
Because only established `QuicChannel` instances are counted, the assertion can
run before the pending UDP channel is visible and pass without testing its
lifecycle.

This defect is independently actionable. Whether it explains the production
symptoms in Consensys/teku#11001 requires separate validation after the library
fix.

## Chosen Approach

Keep the Netty UDP bind future in `QuicTransport.dial()`. When the returned dial
future is cancelled, close the bind future's datagram channel immediately.

Netty creates the channel before asynchronous bind completion, so this handles
cancellation during both UDP bind and QUIC handshake. Closing the parent
datagram channel also terminates an established or establishing QUIC child,
matching the immediate channel cleanup used by `PlainNettyTransport`.

No new transport state, public API, or cancellation wrapper is required.

## Alternatives

### Cancel the CompletableFuture chain

Propagating `CompletableFuture.cancel()` upstream would not reliably cancel the
underlying Netty operations or close their channels. The futures describe
completion but do not own all network resources.

### Track pending QUIC dials

A transport-level registry could retain every pending datagram channel and
remove it on completion. This would also support transport shutdown cleanup,
but adds synchronization and lifecycle state for a cancellation path that can
be handled directly by the existing bind future.

## Characterization And Regression Tests

Use a loopback UDP endpoint that receives packets but never responds as a QUIC
peer:

1. Start a datagram socket on loopback.
2. Dial it with `QuicTransport`.
3. Receive the first QUIC datagram and record the client's source port. This
   proves the dial socket is bound and the handshake is pending.
4. Cancel the returned dial future.
5. Poll until the applicable deadline while trying to bind another socket to
   the recorded client port.

Before changing production code, run this as a characterization test with a
35-second deadline. Record the elapsed release time and confirm that the
current implementation releases the socket around Netty's 30-second QUIC
connect timeout. This distinguishes bounded retention from an unbounded leak.

Then tighten the same test to require release within five seconds. It must fail
against the current implementation because cancellation does not release the
socket promptly. After the fix, cancellation closes the channel and the port
becomes bindable within that short deadline.

Only the prompt-release assertion remains in the committed test suite. The
30-second characterization run is evidence gathered during development, not a
permanent addition to test duration.

The existing established-connection cancellation test remains useful for the
post-handshake race and will be retained.

## Scope And Success Criteria

- Cancelling a pending QUIC dial promptly closes its ephemeral UDP channel.
- A pre-fix characterization run confirms current retention is bounded at
  approximately 30 seconds.
- Cancellation after QUIC establishment remains safe.
- The new regression test fails against the current implementation and passes
  with the fix.
- Existing focused QUIC transport tests continue to pass.
- No Teku dependency update is included in this change.
