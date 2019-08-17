Van Jacobson, Sun Jul 28 19:24:28 PDT 2019

NFD patches to fix multicast Interests and improve Interest satisfaction
time.

These are two small patches to fix misbehavior we have observed in the
current version NFD (github.com/named-data/NFD v0.6.6+ commit 07f2e2f
July 22, 2019). They should "git apply" cleanly to most recent
versions of NFD.

Once NFD is running, it will be necessary to set up a route to the multicast face. Run "nfdc status report" and look for the the udp4 multicast face_id (e.g., 259) and then "nfdc route add localnet face_id"

**no-nacks-on-multicast-faces.patch** disables NACK processing on multicast faces. Currently an NFD will send a NACK in response to an arriving Interest when it has no faces (including the faces created by local apps' prefix registrations) to forward it on. This makes multicast Interests extremely unreliable. Consider host A's multicast Interest "/localnet/foo" being received by hosts B and C. B has a local process registered for "/localnet/foo" so B will forward the Interest to it but C does not so it immediately multicasts a NACK which causes A to discard the Interest. Sometime later the process on B sends a Data responding to the Interest which B's NFD multicasts (since the NACK didn't kill the pending interest on B) but A discards because it has no pending Interest. Clearly *no* communication between A & B is possible since C's NACKs always arrive before A or B's Data's.
Since link-level NACKs can often do harm by killing off Interests that might otherwise be satisfied before they expire, if deployed, there should be a per-face flag to enable them and the flag should default to "disabled".

**ship-pending-interests-on-register.patch** fixes an NFD implementation error that causes communication involving ephemeral processes to take time on the order of Interest lifetimes (>1-2s) rather than Interest propagation times (<500uS). The NDN abstract forwarder model says that Interests should stay in the forwarder's PIT until they are satisfied or time out. When new Interest satisfaction opportunities arise, such as new prefix registrations or new faces getting added to existing registrations, matching pending Interests should be immediately forwarded to take advantage of them. NFD doesn't do this which means an ephemeral process doesn't see interests from a longer-lived peer until the peer's existing Interest times out and is refreshed. This patch makes all pending Interests covered by a new registration be delivered immediately upon completion of the new registration. That means ephemeral processes get peer interests ~5ms after they start rather than ~2s after they start (5ms is the time it takes NFD to complete a prefix registration).
