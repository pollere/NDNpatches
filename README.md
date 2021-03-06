# NFD patches to fix multicast Interests and improve Interest satisfaction time

### Van Jacobson, Mon Nov 18 10:00:24 PST 2019

These are some small patches to fix misbehavior we have observed in the
current version NFD (github.com/named-data/NFD v0.6.6+ commit 07f2e2f
July 22, 2019). They should "git apply" cleanly to most recent
versions of NFD.

**no-nacks-on-multicast-faces.patch** disables NACK processing on multicast faces. Currently an NFD will send a NACK in response to an arriving Interest when it has no faces (including the faces created by local apps' prefix registrations) to forward it on. This makes multicast Interests extremely unreliable. Consider host A's multicast Interest "/localnet/foo" being received by hosts B and C. B has a local process registered for "/localnet/foo" so B will forward the Interest to it but C does not so it immediately multicasts a NACK which causes A to discard the Interest. Sometime later the process on B sends a Data responding to the Interest which B's NFD multicasts (since the NACK didn't kill the pending interest on B) but A discards because it has no pending Interest. Clearly *no* communication between A & B is possible since C's NACKs always arrive before A or B's Data's.
Since link-level NACKs can often do harm by killing off Interests that might otherwise be satisfied before they expire, if deployed, there should be a per-face flag to enable them and the flag should default to "disabled".

**ship-pending-interests-on-register.patch** fixes an NFD implementation error that causes communication involving ephemeral processes to take time on the order of Interest lifetimes (>1-2s) rather than Interest propagation times (<500uS). The NDN abstract forwarder model says that Interests should stay in the forwarder's PIT until they are satisfied or time out. When new Interest satisfaction opportunities arise, such as new prefix registrations or new faces getting added to existing registrations, matching pending Interests should be immediately forwarded to take advantage of them. NFD doesn't do this which means an ephemeral process doesn't see interests from a longer-lived peer until the peer's existing Interest times out and is refreshed. This patch makes all pending Interests covered by a new registration be delivered immediately upon completion of the new registration. That means ephemeral processes get peer interests ~5ms after they start rather than ~2s after they start (5ms is the time it takes NFD to complete a prefix registration).

**build-system.patch** This is an *optional* patch to fix NFD or ndn-cxx build problems on systems using a modern c++ compiler. In [May 2018](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0754r2.pdf) the C++ standards committee added a new standard header, `<version>`, to hold library properties and implementation-defined library meta-information. *Many* core library routines now include it.  Unfortunately, the ndn-cxx and NFD projects' `wscript` build file creates a file called VERSION in the top-level directory then makes the top-level directory the first entry in the c++ include path via an `-I.` in `CXXFLAGS`. On case-independent file systems like MacOS, this bogus VERSION file is always chosen in place of the library's <version> and nothing will compile. This patch adds a `.nfd` extension to VERSION so it doesn't conflict.  It also turns off some `-W` warnings that result in thousands of warning messages from boost includes, obscuring real issues.

To test using multicast-based tools (like the DNMP suite), it will be necessary to set up a a prefix that routes to a multicast face.  For example, DNMP uses prefix "localnet" for local network(s) multicast.  To set it up, once NFD is running, run `nfdc status report` and look for the the udp4 (or 6) multicast face_id (e.g., 259) and then `nfdc route add localnet face_id`. (Note: Google WiFi will only do IPv6 multicast, so use the udp6 multicast face_id if you have Google WiFi.) It is also necessary to make sure that NFD really uses the multicast strategy instead of the "best route" strategy. This can be ensured by adding the following line to your nfd.conf file:
    /localnet /localhost/nfd/strategy/multicast

**patch.key-impl** This patch to ndn-cxx is necessary to stop the ndnsec tool from aborting when it encounters a key format it doesn't a key type it doesn't recognize (such as a schema or an EdDSA key). This is necessary to use the Data-Centric Toolkit in the DCT repo.

**patch.ndn-ind** patch against [https://github.com/operantnetworks](https://github.com/operantnetworks) ndn-ind HEAD commit b72bbf7e that does two things:

-  adds an 'ndn-ind/async-face.hpp' object that supports single-threaded, asynchronous (callback-driven) i/o and timers without the locking and serialization overhead of threadsafe-face.
-  adds /opt/local/{include,lib} to libndn-ind.pc to support open source packages installed by MacPorts in addition to Homebrew packages from /usr/local.
  