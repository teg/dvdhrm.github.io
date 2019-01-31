---
layout: post
name: Rethinking the D-Bus Message Bus
categories: [fedora]
tags: [dbus, fedora, bus1, dbus-broker, dbus-daemon]
permalink: /:title/
---

Later this year, on November 21, 2017, D-Bus will see its
[15th birthday](https://cgit.freedesktop.org/dbus/dbus/commit/?id=93cff3d69fb705806d2af4fd6f29c497ea3192e0).
An impressive age, only shy of the KDE and GNOME projects, whose collaboration
inspired the creation of this independent IPC system. While still relied upon
by the most recent KDE and GNOME releases, D-Bus is not free of criticism.
Despite its age and mighty advocates, it never gained traction outside of its
origins. On the contrary, it has long been criticized as bloated,
over-engineered, and orphaned. Though, when looking into those claims, you're
often left with unsubstantiated ranting about the environment D-Bus is used in.
If you rather want a glimpse into the deeper issues, the best place to look is
the
[D-Bus bug-tracker](https://bugs.freedesktop.org/buglist.cgi?bug_status=__open__&product=dbus),
including the assessments of the D-Bus developers
themselves. The bugs range from uncontrolled memory usage, over silent dropping
of messages, to dead-locks by design, unsolved for up to 7 years. Looking
closer, most of them simply cannot be solved without breaking guarantees long
given by *dbus-daemon(1)*, the reference implementation. Hence, workarounds
have been put in place to keep them under control.

Nevertheless, these issues still bugged us!
Which is, why we rethought some of the fundamental concepts behind the shared
Message Buses defined by the D-Bus Specification. We developed a new
architecture that is designed particularly for the use-cases of modern D-Bus,
and it allows us to solve several long standing issues with *dbus-daemon(1)*.
With this in mind, we set out to implement an alternative D-Bus Message Bus.
Half a year later, we hereby announce the
[**dbus-broker project**](https://www.github.com/bus1/dbus-broker/wiki)!

But before we dive into the project, lets first have a look at some of the long
standing open bug reports on D-Bus. A selection:

 * [Bug #33606](https://bugs.freedesktop.org/show_bug.cgi?id=33606#c11):
   *"stop dbus-daemon memory usage ballooning if a client is slow to read"*

   The bug-report describes a situation where the memory-usage of
   *dbus-daemon(1)* grows in an uncontrolled manner, if inflight messages keep
   piling up in the incoming and outgoing queues of the daemon. Despite
   being reported more than 6 years ago, there is no satisfying solution
   to the issue.

   What it boils down to is the fact that *dbus-daemon(1)* does not
   judge messages based on their message type. Hence, whether a message
   was triggered by a peer itself (e.g., a method call), or triggered by
   another peer (e.g., a method reply), the message is always accounted on
   the sender of the message. Hence, if those messages are piled up in
   outgoing queues in *dbus-daemon(1)*, the sender of those messages is
   accounted and punished for them.
   This can be misused by malicious applications that simply trigger a
   target peer to send messages (like method replies and signals), but
   they never read those messages but leave them queued. As a result,
   there is still no agreed upon way to decide who to punish for excessive
   buffering.

 * [Bug #80817](https://bugs.freedesktop.org/show_bug.cgi?id=80817):
   *"messages with abusive recursion are silently dropped"*

   Depending on the linux kernel you use, consecutively queued
   unix-domain-sockets may be rejected by **sendmsg(2)**. This can have the
   effect of *dbus-daemon(1)* being unable to forward a message. The message
   will be silently dropped, without notifying anyone.

   There is no known workaround for this issue, since the time of
   **sendmsg(2)** might be too late for proper error-handling, due to output
   buffering or short writes.

   Similarly,
   [Bug #52372](https://bugs.freedesktop.org/show_bug.cgi?id=52372)
   describes another situation where messages are
   silently dropped, if they are queued on an activatable name but their
   sender disconnects before the destination is activated.

   Lastly, *dbus-daemon(1)* might fail any message and reply with an error
   message. That is, method-calls but also method-replies, signals, and
   error-messages can all be rejected for arbitrary reason by
   *dbus-daemon(1)* and trigger an error-reply. Nearly no application is
   ready to expect asynchronous error-replies to their attempt to send a
   method reply or signal.
   Again, this stems from *dbus-daemon(1)* never judging messages by their
   type. Despite method-transactions being stateful, there is no reliable
   way for a peer to cancel a message transaction. Any attempt to do so
   might fail. Same is true for a signal-subscription.

   There are some more similar scenarios where *dbus-daemon(1)* has to
   silently drop messages, or unexpectedly rejects messages, thus breaking
   the rule of reliability. This is not about catching errors in client
   libraries, but this is about either messages being silently discarded
   or asynchronously rejected.

 * [Bug #28355](https://bugs.freedesktop.org/show_bug.cgi?id=28355):
   *"dbus-daemon hangs while starting if users are in LDAP/NIS/etc."*

   Additionally to client-side policies, *dbus-daemon(1)* implements a
   mandatory access control mechanism, based on uids, gids, and message
   content. This, however, required D-Bus to resolve user-names and
   group-names to IDs, which will involve NSS, and as such LDAP/NIS/etc.
   This has long been a source of deadlocks, when using D-Bus to implement
   those NSS modules themselves. Workarounds are available, but the
   problem itself is not solved.

 * [Bug #83938](https://bugs.freedesktop.org/show_bug.cgi?id=83938):
   *"improve data structures for pending replies"*

   This bug-report concerns the method-call tracking in *dbus-daemon(1)*,
   which is used to allow exactly one reply per method-call, but not more.
   A list of _open reply windows_ is kept to track pending method-calls.
   In *dbus-daemon(1)*, this is a global, linked list, searched whenever a
   reply is sent. By queuing up too many replies on too many connections,
   lookups on this list will consume a considerable amount of time,
   slowing down the entire bus.

   While the issue at hand can be solved, and has been solved, there
   remain many similar global data-structures in *dbus-daemon(1)*, that are
   shared across all users. Some of them can be fixed, some cannot, since
   D-Bus defines some global behavior (like broadcast matching and
   name-ownership/handover). This prevents D-Bus from scaling nicely with
   more processors being added to a system.

   In fact, the name-registry of D-Bus, and the atomic hand-over of queued
   name owners, requires huge global state-tracking without any known
   efficient, parallel solution.

   Furthermore, many of the employed workarounds simply introduce per-peer
   limits for those global resources. By setting them low enough, their
   scope has been kept under control. However, history shows that those
   limits have had violated application expectations
   [several](https://bugs.freedesktop.org/show_bug.cgi?id=50264)
   [times](https://github.com/NetworkManager/NetworkManager/commit/2c299ba65c51e9c407090dc83929d692c74ee3f2).

None of the issues mentioned here is critical enough for D-Bus to become
unbearable. On the contrary, D-Bus is still popular and no serious replacement
is even close to be considered a contender. Furthermore, suitable workarounds
have often been put in place to control those issues.

But we kept being annoyed by these fundamental problems, so we set forth to
solve them in *dbus-broker(1)*. What we came up with is a set of theoretical
rules and concepts for a different message bus:

 1. **No Shared Medium**

    This is a rather theoretical change. Previously, the D-Bus Message Bus
    followed the model of actual physically wired buses, where peers place
    messages on a shared medium for others to fetch. The problem here is to
    guarantee fairness, and to make peers accountable for excessive use.
    In D-Bus the problem can be reduced to outgoing queues in the message
    broker. Whenever many peers send messages to the same destination, they
    fill the same message queue. If that queue runs full, someone needs to
    be held accountable. Was the destination too slow reading messages and
    should be disconnected? Did a sender flood the destination with an
    unreasonable amount of messages? Or did an innocent 3rd party just send
    a single message, but happened to be the final straw?

    We decided to overcome this by throwing the model of a shared medium
    overboard. We no longer consider a D-Bus Message Bus a global medium
    that all peers
    are connected to and submit messages to. We rather consider a bus a set
    of distinct peers with no global state. Whenever a peer sends a message,
    we consider this a transaction between the sender and the destination
    (or multiple destinations in case of multicasts). We try to avoid any
    global state or context. We want every action taken by a peer to only
    affect the source and target of the action, but nothing else.

    While nice in theory, D-Bus does not allow this. There is global state,
    and it is hard-coded in the D-Bus specification with many existing
    applications relying on it. However, we still tried to stick to this as
    close as possible. In particular, this means:

    * Whenever a peer creates an object in the bus manager, it must be
      linked and indexed on a specific peer. There must not be any
      global lists or maps. Whenever the bus manager performs a
      transaction, it must be able to collect all objects that affect
      it by just looking at the involved peers.

      This rule is, in some corner-cases, violated to keep compatibility
      to the specification. That is, if applications rely on global
      behavior, it will still work. However, anything that can be
      indexed, is indexed, and as long as applications don't rely on
      obscure D-Bus features, they will never end up in those global
      data-structures.

    * We now judge messages by their message types. We implement proper
      message transactions and always know who to account for for
      inflight
      messages. Moreover, every peer now has a limited incoming queue,
      which every other peer gets a fair share of. Whenever a peer
      exceeds their share on another peer's queue, one of both exceeded
      their configured limits and the message must be rejected.
      Details on how we dynamically adjust those shares can be found in
      the online
      [documentation](https://github.com/bus1/dbus-broker/wiki/Accounting).

      We still need to decide who is at fault. Is the sender to blame
      or the receiver? Our solution is to base this on the question
      whether a message is unsolicited. That is, for unsolicited
      messages, the sender is to blame. For solicited messages, the
      receiver is to blame. Effectively, this means whenever you send a
      method call, you are to blame if you did not account for the
      reply. In case of signals, we simply treat a subscription as the
      intention of the subscriber to receive an unlimited stream of
      signals, thus making subscribed signals solicited.

      Lastly, in case of unsolicited messages, we reply with an error,
      and expect every peer to be able to deal with asynchronous errors
      to unsolicited messages. By contrast, solicited messages
      never yield an error. Instead, we always consider the receiver of
      solicited messages to be at fault, thus throw them off the bus.

 2. **No IPC to implement IPC**

    D-Bus is an IPC mechanism to allow other processes to communicate. We
    strictly believe that the implementation of an IPC mechanism should not
    use IPC itself. Otherwise, deadlocks are a steady threat.

    This means, the transaction of a message (whatever kind) should not
    depend on any other means but local data. We do not read files, we do
    not invoke NSS, we do not call into D-Bus. Instead, the operation of the
    bus manager regarding message transactions is a self-contained process
    without any external hooks or callbacks.

 3. **User-based Accounting**

    Any resource and any object allocated in the bus must be accounted on a
    user. We do not account based on peers, but always account based on
    users.

    In particular, this means we never have stacked accounting. We have
    limits for specific resources, but all those limits only ever affect the
    user accounting. That is, you can no longer exceed limits by simply
    connecting multiple times to the bus, or by creating objects that have
    separate accounting. Instead, whenever an action is accounted, it will
    be accounted on the calling user, regardless through which peer or object
    the action is performed.

 4. **Reliability**

    Never ever shall a message be silently dropped! Any error condition must
    be caught and handled, and must never put peers into unexpected
    situations.

    If a situation arises where we cannot gracefully handle an error
    condition, we exit. We never put the burden on the peers, nor do we
    silently ignore it.

With these in mind, we implemented an independent D-Bus Message Bus and named
it **dbus-broker**. It is available on
[GitHub](https://www.github.com/bus1/dbus-broker) and already capable of
booting a full Fedora Desktop System. Some of its properties are:

 * **Pure Bus Implementation**

   One of our aims was to make the bus manager a pure implementation with as
   little policy as possible. Furthermore, following our rule of *"No IPC to
   implement IPC"*, we stripped all external communication from it. The
   result is a standalone program we call *dbus-broker*, which implements a
   Message Bus as defined by the D-Bus specification. The only external
   control channel is a private socketpair that must be passed down by the
   parent process that spawns *dbus-broker(1)*. This channel is used to control
   the broker at runtime, as well as get notified about specific events like
   name activation.

   On top of this, we implemented a launcher compatible to *dbus-daemon(1)*,
   employing *dbus-broker(1)*. This *dbus-broker-launch(1)* program implements
   the *dbus-daemon(1)* semantics of a system and session/user message bus.

 * **Local Only**

   We only implement local IPC. No remote transports are supported. We
   believe that this is beyond the realm of D-Bus. You can always employ ssh
   tunneling to get remote D-Bus working, just like most projects do
   already.

 * **No Legacy**

   We do not implement legacy D-Bus features. Anything that is marked as
   deprecated was dropped, as long as it is not relied upon by crucial
   infrastructure. We are compatible to *dbus-daemon(1)*, so use it if your
   system still requires those legacy features.

   All those deviations are documented in our online
   [wiki](https://github.com/bus1/dbus-broker/wiki/Deviations). Each case comes
   with a rationale why we decided to drop support for it.

 * **Linux Only**

   A lot of functionality we rely on is simply not available on other
   operating systems. *dbus-daemon(1)* is still around (and will stay around),
   so there will always be a working D-Bus Message Bus for other operating
   systems.

   Note that we rely on several peculiar features of the linux kernel to
   implement a secure message broker (including its accounting for inflight
   FDs, its output queueing on *AF_UNIX* including the *IOCOUTQ* ioctl,
   edge-triggered event notification, *SO_PEERGROUPS* ioctl, and more). We
   fixed
   [several](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/net/core/sock.c?id=28b5ba2aa0f55d80adb2624564ed2b170c19519e)
   [bugs](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/net/unix/af_unix.c?id=27eac47b00789522ba00501b0838026e1ecb6f05)
   upstream just few weeks ago, and we will continue to
   do so. But we are not in a position to review other kernels for the same
   guarantees.

 * **Pipelining**

   We support SASL pipelining for fast connection attempts. This means, all
   SASL D-Bus authentication requests can be queued up without waiting for
   their replies, including any following *Hello()* call or other D-Bus
   Message.

   This allows connecting to the message broker without waiting for a single
   roundtrip.

 * **No Spec-Deviation**

   We do not intend to add features not standardized in the D-Bus
   Specification, nor do we intend to deviate. However, we do sometimes
   deviate from the behavior of the reference implementation. All those
   deviations are carefully considered and
   [documented](https://github.com/bus1/dbus-broker/wiki/Deviations).

   Our intention is to base this
   implementation on the ideas described above, and thus fix some of the
   fundamental issues we see in D-Bus. We report all our findings back and
   recommend solutions to upstream *dbus-daemon(1)*. Discussion and development
   of the D-Bus specification still happens upstream. We are not the persons
   to contact for extensions of the specification, but we will happily
   collaborate on the upstream mailing-list and bug-tracker with whoever
   wants to discuss D-Bus.

 * **Runtime Broker Control**

   The message broker process provides a control API to its parent process
   via a private connection. It allows to feed initial
   state, but also control the broker at runtime.

   While it can and is used to implement compatibility to the dbus-daemon
   configuration files, it is also possible to modify the broker at runtime,
   if necessary. This includes adding and removing listener sockets and
   activatable names at runtime. Thus, appearance of activatable names can
   now be scheduled arbitrarily.

Please be aware that the *dbus-broker* project is still experimental. While we
successfully use it on our machines to run the system and session/user bus, we
do not recommend deploying it on production machines at this time. We are
not aware of any critical bugs, but we do want more testing before recommending
its deployment.

If you are curious and want to try it out, there are packages available for
Fedora and Arch Linux. Other distributions will follow. The online
documentation also contains information on how to compile and deploy it
manually.

 * [Project Wiki](https://github.com/bus1/dbus-broker/wiki)
 * [Issue Tracker](https://github.com/bus1/dbus-broker/issues)
 * Last Release: [v3](https://github.com/bus1/dbus-broker/archive/v3/dbus-broker-v3.tar.gz)
 * Fedora Packages in [Copr](https://copr.fedorainfracloud.org/coprs/g/bus1/dbus/package/dbus-broker/)
 * Arch Linux Packages in [AUR](https://aur.archlinux.org/packages/dbus-broker)
