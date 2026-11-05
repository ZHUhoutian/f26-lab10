
# RACE.md

One section per milestone. Fill each one in order as you go.

The prediction in milestone 1 is written BEFORE you run your race test. A TA
may ask about it. Commit it on its own so the order is visible.

Keep it short and specific. Point at methods, call sites, and the invariant.

---

## Milestone 1: from the ticket to a test that fails every time

### Suspect list

Everything your review turned up that touches shared state, with a one-line
verdict for each. Add or remove rows as needed.

| # | What you found (method, field, or class) | Verdict, in one line |
|---|---|---|
| 1 | `BookingService.book()`: `hasOverlap` then `insert` | **Race, explains the ticket.** Check-then-act: the store lock is released between the two calls, so the check can go stale. |
| 2 | `book()` against `cancelBooking()` | Same root cause, not this ticket: a promotion can land between the check and the insert, but no ticketed booking was waitlisted or cancelled. |
| 3 | `BookingStats` counters | Data race, intended: documented as approximate and nothing decides on them. The ticket's bad state is in the booking data. |
| 4 | `InMemoryBookingStore` methods, `catalog`, `sequence`, `joinWaitlist()`, `listBookings()` | Safe: each is one atomic store call, a read of data that never changes, or an `AtomicLong`. |

**Which one explains the ticket, and how you know.** Suspect 1. The invariant is
"at most one CONFIRMED booking covering any given minute." Only `book()` and
promotion write a CONFIRMED booking, and the ticket rules out promotion. In any
serial order, the second `book()` sees the first and logs `booking.rejected`. The
log instead shows two `booking.confirmed` lines 1 ms apart with no rejection, so
both checks ran before either insert.

### Predicted interleaving (write this before you run your race test)

Two threads, one row per step, in the form from lecture. Describe what each
thread does at that step and leave the other column empty. Add or remove rows
as your prediction needs.

| Step | Thread A | Thread B |
|---|---|---|
| t1 | `book()`: `hasOverlap` (call 1) returns false; parked before returning | |
| t2 | | `book()`: `hasOverlap` (call 2) returns false; not parked (one-shot gate) |
| t3 | | inserts `bk-1` CONFIRMED and logs `booking.confirmed` |
| t4 | released, still holding the stale false; inserts `bk-2` CONFIRMED and logs `booking.confirmed` | |
| | **End state:** two CONFIRMED bookings for WEH-5409 `[540, 600)`, no `booking.rejected`; the test fails with expected 1, was 2 | |

If the run turned out different from your prediction, add one line on what
you missed. Leave the table as you first wrote it.

### Your race test

**How you aimed `PausingStore`, and the result.** Method name and call index, why
that pair pins the schedule you predicted, and the output of two separate
runs (assertion failure and log lines).

Aim: `new PausingStore(real, "hasOverlap", 1)`. Call 1 is customer A's check, so
A is parked after it read "free" and before it can insert. That holds A inside
the gap while B runs `book()` start to finish.

The run matched the prediction, including `gate fired=true` with 2 `hasOverlap`
calls.

Run 1:

```
2026-11-05T21:13:12.504Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-05T21:13:12.505Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
gate fired=true hasOverlap calls=2
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.027 s <<< FAILURE! -- in edu.cmu.cs214.lab10.harness.RaceTest
org.opentest4j.AssertionFailedError: CONFIRMED bookings for WEH-5409 [540, 600) ==> expected: <1> but was: <2>
```

Run 2:

```
2026-11-05T21:13:13.416Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-05T21:13:13.418Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
gate fired=true hasOverlap calls=2
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.025 s <<< FAILURE! -- in edu.cmu.cs214.lab10.harness.RaceTest
org.opentest4j.AssertionFailedError: CONFIRMED bookings for WEH-5409 [540, 600) ==> expected: <1> but was: <2>
```

---

## Milestone 2: the patch a teammate sent

**Your milestone-1 test on the patched code.** Green or red. Paste the result.

Green. A parks before its re-check, so the re-check sees B's booking and A is
rejected.

```
2026-11-05T21:21:43.906Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-05T21:21:43.910Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
gate fired=true hasOverlap calls=4
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
```

**The re-aimed run.** Method name, call index, the pasted result, and one line
on why that aim tests the patch rather than sidestepping it.

Aim: `new PausingStore(real, "hasOverlap", 2)`, in
`twoCustomersInTheRecheckToInsertGapLeaveOneConfirmedBooking`. Call 2 is A's
re-check, so A is parked after its last look at the store and before `insert`.
That is the window the patch claims to close, so the aim goes through the patch
rather than around it. Red on both runs. `bk-2` logs first because A took `bk-1`
before its re-check.

Run 1:

```
2026-11-05T21:21:41.970Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
2026-11-05T21:21:41.972Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
gate fired=true hasOverlap calls=4
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.027 s <<< FAILURE! -- in edu.cmu.cs214.lab10.harness.RaceTest
org.opentest4j.AssertionFailedError: CONFIRMED bookings for WEH-5409 [540, 600) ==> expected: <1> but was: <2>
```

Run 2:

```
2026-11-05T21:21:42.969Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
2026-11-05T21:21:42.971Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
gate fired=true hasOverlap calls=4
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.025 s <<< FAILURE! -- in edu.cmu.cs214.lab10.harness.RaceTest
org.opentest4j.AssertionFailedError: CONFIRMED bookings for WEH-5409 [540, 600) ==> expected: <1> but was: <2>
```

**Your verdict.** Accept or reject, and what your milestone-1 result does and
does not show about the patch.

**Reject.** The patch adds another check but still releases the store lock before
`insert`, so check-then-act remains and the gap only moves. The milestone 1 green
shows the patch handles one ordering, where B inserts before A's re-check. It
does not show the race is gone, because that test never parks a thread after the
re-check. The re-aimed run does, and it breaks the invariant every time.

The stress runner agrees, though it only supports the verdict: the patched code
still had bad rounds in every run (2-4 per 100, 3-5 per 2000, 14 per 20000),
against the teammate's report of none. The patch makes the race rarer and so
harder to catch, which is not a fix.

---

## Milestone 3: the fix you would ship

**Which option, and what you changed.** Name the lock or the atomic operation
and the two operations it now covers. Paste the diff of your change and the
passing run of your milestone-1 test and the full suite.

Option 2. `BookingStore.insertIfFree(Booking)` is the atomic operation. In
`InMemoryBookingStore` it holds the store's monitor across the two operations, the
overlap check and the insert. `book()` calls it instead of `hasOverlap` then
`insert`.

```diff
diff --git a/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java b/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java
index 5e6b639..9c65544 100644
--- a/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java
+++ b/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java
@@ -53,8 +53,8 @@ public final class PausingStore implements BookingStore {
     /**
      * @param delegate     the real store, which does all the work
      * @param targetMethod the {@link BookingStore} method to park inside, by
-     *     name: {@code hasOverlap}, {@code insert}, {@code bookingsForRoom}, or
-     *     {@code cancelAndPromote}
+     *     name: {@code hasOverlap}, {@code insert}, {@code insertIfFree},
+     *     {@code bookingsForRoom}, or {@code cancelAndPromote}
      * @param targetCall   which invocation of that method to park, counting
      *     from 1 across all threads
      */
@@ -109,6 +109,11 @@ public final class PausingStore implements BookingStore {
         });
     }
 
+    @Override
+    public boolean insertIfFree(Booking booking) {
+        return gate("insertIfFree", () -> delegate.insertIfFree(booking));
+    }
+
     @Override
     public List<Booking> bookingsForRoom(String roomId) {
         return gate("bookingsForRoom", () -> delegate.bookingsForRoom(roomId));
diff --git a/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java b/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java
index 514c3e1..75faa4f 100644
--- a/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java
+++ b/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java
@@ -22,10 +22,12 @@ import java.util.concurrent.atomic.AtomicLong;
  * <p>Thread safety: safe for concurrent use. The service owns no mutable
  * booking state of its own. The room catalog is built once in the constructor
  * and never written again, and every read and write of booking data goes
- * through {@link BookingStore}, whose methods are each synchronized. Since the
- * store is thread safe, the operations built on top of it are too. The only
- * other field that changes after construction is {@link BookingStats}, and its
- * numbers are documented as approximate.
+ * through {@link BookingStore}, whose methods are each synchronized. That makes
+ * one store call atomic, not a sequence of them, so an operation that depends on
+ * a check staying true must do its check and write in a single store call, as
+ * {@link #book} does with {@link BookingStore#insertIfFree}. The only other field
+ * that changes after construction is {@link BookingStats}, and its numbers are
+ * documented as approximate.
  */
 public final class BookingService {
 
@@ -67,11 +69,12 @@ public final class BookingService {
         if (startMinute < 0 || endMinute <= startMinute) {
             return reject(roomId, startMinute, endMinute, "empty or reversed slot");
         }
-        if (store.hasOverlap(roomId, startMinute, endMinute)) {
+        Booking booking = new Booking(nextId(), roomId, startMinute, endMinute, BookingStatus.CONFIRMED);
+        // One store call on purpose: a separate check and insert would release
+        // the store's lock between them and let two bookings through.
+        if (!store.insertIfFree(booking)) {
             return reject(roomId, startMinute, endMinute, "slot already held");
         }
-        Booking booking = new Booking(nextId(), roomId, startMinute, endMinute, BookingStatus.CONFIRMED);
-        store.insert(booking);
         stats.recordConfirmed();
         BookingLog.confirmed(booking);
         return BookingResult.confirmed(booking);
diff --git a/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java b/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java
index 3ae3544..92f936c 100644
--- a/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java
+++ b/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java
@@ -28,6 +28,17 @@ public interface BookingStore {
     /** Stores the booking. Ids are assumed unique; the store does not check. */
     void insert(Booking booking);
 
+    /**
+     * Stores a booking only if the room has no CONFIRMED booking touching its
+     * range, checking and writing as one indivisible step. No other thread can
+     * change the room between the two.
+     *
+     * @param booking the booking to store
+     * @return true when it was stored, false when the range was taken and
+     *     nothing changed
+     */
+    boolean insertIfFree(Booking booking);
+
     /**
      * Every booking held for the room, in insertion order, in every status.
      *
diff --git a/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java b/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java
index f7fd85f..d7c8833 100644
--- a/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java
+++ b/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java
@@ -36,6 +36,15 @@ public final class InMemoryBookingStore implements BookingStore {
         bookings.add(booking);
     }
 
+    @Override
+    public synchronized boolean insertIfFree(Booking booking) {
+        if (overlapsConfirmed(booking)) {
+            return false;
+        }
+        bookings.add(booking);
+        return true;
+    }
+
     @Override
     public synchronized List<Booking> bookingsForRoom(String roomId) {
         List<Booking> forRoom = new ArrayList<>();
```

Passing run (in order: the new `insertIfFree` test, the milestone 2 test, then the milestone 1 test):

```
2026-11-05T21:30:07.143Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
2026-11-05T21:30:07.148Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
gate fired=true insertIfFree calls=2
2026-11-05T21:30:07.152Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-05T21:30:09.153Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
gate fired=false hasOverlap calls=0
2026-11-05T21:30:09.155Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-05T21:30:11.155Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
gate fired=false hasOverlap calls=0
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.028 s -- in edu.cmu.cs214.lab10.harness.RaceTest
[INFO] Tests run: 14, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

**Is your milestone-1 test still testing anything?** After your fix, describe
what that test now exercises. Did `PausingStore` pause a call during the green run? How do
you know?

Not the race. `PausingStore` paused nothing: the test printed `gate fired=false
hasOverlap calls=0`, because `book()` no longer calls `hasOverlap`. The
confirmation and rejection are 2 s apart, which is the `awaitPaused` timeout, so A
finished before B started. The run is sequential.

It is still a regression guard. If someone brings back a check and a separate
insert, the gate fires again and the test goes red, as it did before the fix. I
left it unchanged for that reason.

Adjustment: I added `secondCustomerIsRejectedWhileFirstIsParkedAfterTheAtomicInsert`,
aimed at `insertIfFree` call 1. Its gate does fire (`gate fired=true`), and B is
rejected while A is parked. No test can park inside `insertIfFree`, so its
atomicity rests on `synchronized`. The stress runner supports that but does not
prove it: 0 bad rounds in 26,300.

**What runs one at a time now that ran in parallel before.** Name concretely which callers
queue behind which. "Nothing, it is just correct now" does not count as an
answer.

Every `book()`, for any room and from any service sharing the store, now holds the
store's monitor for its check and its write together. Before, other calls could run
between the two. Now `book()` callers queue behind each other, and behind
`cancelAndPromote`, `joinWaitlist`'s insert and `listBookings`, and those queue
behind `book()`.

The cost is that bookings for unrelated rooms queue too, because one lock covers
the whole list, and the check scans that whole list while holding it. Validation,
`nextId()`, stats and logging stay outside the lock.

**What you left unfixed.** For anything your review flagged that you chose not
to fix, one line each on why.

- `BookingStats`: lost updates are documented and accepted, and nothing decides on
  the counters.
- `nextId()` counts per service: two services on one store could issue the same id.
  This is latent and did not cause the ticket, whose ids differ.
- `book()` now takes an id before the store check, so a rejected attempt skips a
  number. Ids only need to be unique.
- `hasOverlap` stays on the interface: the service no longer calls it, but pairing
  it with `insert` would reopen this race.
- Suspect 2 needed no separate change: `insertIfFree` and `cancelAndPromote` share
  the same monitor, so a promotion cannot land inside `book()`. That comes from
  reading the code; I did not run a test for it.
