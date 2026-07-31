# QUIC Dial Cancellation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Release the ephemeral UDP channel immediately when a pending QUIC dial is cancelled.

**Architecture:** Preserve the UDP bind `ChannelFuture` created by `QuicTransport.dial()` and close its channel from the returned connection future's cancellation handler. Characterize the current timeout-bounded retention with a loopback UDP blackhole before tightening the same probe into a prompt-release regression test.

**Tech Stack:** Kotlin, Java, Netty 4.2 QUIC, JUnit 5, Gradle

## Global Constraints

- Do not change public transport APIs.
- Keep the established-QUIC cancellation behavior and its existing test.
- Confirm current retention is bounded at approximately 30 seconds before changing production code.
- Keep only the five-second prompt-release assertion in the final test suite.
- Do not include a Teku dependency update.

---

### Task 1: Characterize Current Pending-Socket Retention

**Files:**
- Modify: `libp2p/src/test/java/io/libp2p/transport/quic/QuicServerTestJava.java`

**Interfaces:**
- Consumes: `QuicTransport.dial(Multiaddr, ConnectionHandler, ChannelVisitor)`
- Produces: `awaitUdpPortReusable(int, Duration): long`, returning elapsed milliseconds

- [ ] **Step 1: Add imports and the UDP port-reuse helper**

Add these imports:

```java
import java.net.BindException;
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.time.Duration;
```

Add this helper near the end of `QuicServerTestJava`:

```java
private static long awaitUdpPortReusable(int port, Duration timeout) throws Exception {
  long startedAt = System.nanoTime();
  long deadline = startedAt + timeout.toNanos();
  BindException lastBindFailure = null;

  while (System.nanoTime() < deadline) {
    try (DatagramSocket probe = new DatagramSocket(null)) {
      probe.setReuseAddress(false);
      probe.bind(new InetSocketAddress(port));
      return TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startedAt);
    } catch (BindException e) {
      lastBindFailure = e;
      Thread.sleep(50);
    }
  }

  throw new AssertionError(
      "UDP port " + port + " was not released within " + timeout, lastBindFailure);
}
```

- [ ] **Step 2: Add the 35-second characterization test**

Add this test before the helper:

```java
@Test
void cancelledPendingDialReleasesUdpSocketWithinQuicConnectTimeout() throws Exception {
  Pair<PrivKey, PubKey> clientKeyPair = KeyKt.generateKeyPair(KeyType.ED25519);
  List<io.libp2p.core.multistream.ProtocolBinding<?>> emptyProtocols = new ArrayList<>();
  QuicTransport clientTransport = QuicTransport.ECDSA(clientKeyPair.component1(), emptyProtocols);
  clientTransport.initialize();

  try (DatagramSocket blackhole =
      new DatagramSocket(new InetSocketAddress(InetAddress.getLoopbackAddress(), 0))) {
    blackhole.setSoTimeout(5_000);
    String targetAddress =
        "/ip4/127.0.0.1/udp/" + blackhole.getLocalPort() + "/quic-v1";

    CompletableFuture<Connection> dial =
        clientTransport.dial(new Multiaddr(targetAddress), conn -> {}, null);

    DatagramPacket firstPacket = new DatagramPacket(new byte[2_048], 2_048);
    blackhole.receive(firstPacket);
    int clientPort = firstPacket.getPort();

    Assertions.assertTrue(dial.cancel(true));

    long releaseMillis = awaitUdpPortReusable(clientPort, Duration.ofSeconds(35));
    System.out.println("Cancelled pending QUIC dial released UDP port after " + releaseMillis + " ms");
    Assertions.assertTrue(
        releaseMillis >= Duration.ofSeconds(25).toMillis(),
        "expected current cleanup to wait for the approximately 30-second QUIC connect timeout");
  } finally {
    clientTransport.close().get(5, TimeUnit.SECONDS);
  }
}
```

- [ ] **Step 3: Run the characterization test**

Run:

```bash
./gradlew :libp2p:test \
  --tests io.libp2p.transport.quic.QuicServerTestJava.cancelledPendingDialReleasesUdpSocketWithinQuicConnectTimeout \
  --info
```

Expected: PASS after approximately 30 seconds. Preserve the printed elapsed time in the implementation notes.

- [ ] **Step 4: Commit the characterization evidence**

```bash
git add libp2p/src/test/java/io/libp2p/transport/quic/QuicServerTestJava.java
git commit -m "test(quic): characterize cancelled dial socket timeout"
```

### Task 2: Require Prompt Cancellation Cleanup

**Files:**
- Modify: `libp2p/src/test/java/io/libp2p/transport/quic/QuicServerTestJava.java`
- Modify: `libp2p/src/main/kotlin/io/libp2p/transport/quic/QuicTransport.kt`

**Interfaces:**
- Consumes: `awaitUdpPortReusable(int, Duration): long` from Task 1
- Produces: cancellation behavior that closes the UDP bind channel immediately

- [ ] **Step 1: Tighten and rename the test**

Rename the test to:

```java
void cancelledPendingDialPromptlyReleasesUdpSocket() throws Exception
```

Replace the characterization assertions with:

```java
long releaseMillis = awaitUdpPortReusable(clientPort, Duration.ofSeconds(5));
Assertions.assertTrue(
    releaseMillis < Duration.ofSeconds(5).toMillis(),
    "cancelled pending QUIC dial did not promptly release its UDP socket");
```

- [ ] **Step 2: Run the test to verify RED**

Run:

```bash
./gradlew :libp2p:test \
  --tests io.libp2p.transport.quic.QuicServerTestJava.cancelledPendingDialPromptlyReleasesUdpSocket
```

Expected: FAIL after five seconds with `UDP port ... was not released within PT5S`.

- [ ] **Step 3: Preserve the UDP bind future**

In `QuicTransport.dial()`, split the bind operation from the QUIC future with
this exact diff:

```diff
-        val quicConnFuture: CompletableFuture<QuicChannel> = client.clone()
+        val udpBindFuture = client.clone()
             .handler(requestsHandler)
             .bind(InetSocketAddress(0))
+
+        val quicConnFuture: CompletableFuture<QuicChannel> = udpBindFuture
             .toCompletableFuture()
             .thenCompose { udpChannel ->
```

Do not change the existing `thenCompose` body.

- [ ] **Step 4: Close the parent UDP channel on cancellation**

Change the existing cancellation block to retain established-channel cleanup
and immediately close the parent channel:

```kotlin
connectionFuture.whenComplete { _, _ ->
    if (connectionFuture.isCancelled) {
        quicConnFuture.thenAccept { it.close() }
        udpBindFuture.channel().close()
    }
}
```

- [ ] **Step 5: Run the prompt-release test to verify GREEN**

Run:

```bash
./gradlew :libp2p:test \
  --tests io.libp2p.transport.quic.QuicServerTestJava.cancelledPendingDialPromptlyReleasesUdpSocket
```

Expected: PASS in substantially less than five seconds.

- [ ] **Step 6: Run the existing post-handshake cancellation test**

Run:

```bash
./gradlew :libp2p:test \
  --tests io.libp2p.transport.quic.QuicServerTestJava.cancelledDialClosesEstablishedQuicConnection
```

Expected: PASS.

- [ ] **Step 7: Commit the fix and final regression**

```bash
git add \
  libp2p/src/main/kotlin/io/libp2p/transport/quic/QuicTransport.kt \
  libp2p/src/test/java/io/libp2p/transport/quic/QuicServerTestJava.java
git commit -m "fix(quic): close UDP channel when dial is cancelled"
```

### Task 3: Verify The QUIC Transport And Module

**Files:**
- Verify: `libp2p/src/main/kotlin/io/libp2p/transport/quic/QuicTransport.kt`
- Verify: `libp2p/src/test/java/io/libp2p/transport/quic/QuicServerTestJava.java`

**Interfaces:**
- Consumes: the final implementation and regression test from Task 2
- Produces: fresh verification evidence for completion

- [ ] **Step 1: Check formatting and the final diff**

Run:

```bash
git diff origin/develop --check
git diff origin/develop --stat
```

Expected: no whitespace errors; only the design, plan, QUIC transport, and QUIC test files differ.

- [ ] **Step 2: Run all QUIC transport tests**

Run:

```bash
./gradlew :libp2p:test --tests 'io.libp2p.transport.quic.*'
```

Expected: BUILD SUCCESSFUL with no failing QUIC tests.

- [ ] **Step 3: Run the complete libp2p test task**

Run:

```bash
./gradlew :libp2p:test
```

Expected: BUILD SUCCESSFUL with no failing tests.

- [ ] **Step 4: Confirm repository state**

Run:

```bash
git status --short --branch
git log --oneline origin/develop..HEAD
```

Expected: clean worktree on `codex/quic-udp-dial-cancellation` with the design,
characterization, fix, and plan commits ahead of `origin/develop`.
