# RoomReserve Critique

Fill in each section. One section per milestone. Keep it short and specific. Point at
files and methods, not adjectives.

---

## Milestone 1: The design as it is

Describe the system as the code actually builds it.

**Data model.** What is a booking, in the code? What types hold it, and what has to stay
in agreement for a booking to make sense?

There is no `Booking` type. A booking is four separate facts, held in three different
places, that have to be kept in sync by hand:

- The room and date exist only as fragments of a string key, `roomId + "|" + date`, used
  to look up `List<long[]>` in `InMemoryStore.slotsByRoomDate` (`InMemoryStore.java:13,17`).
- The interval is a raw two-element `long[]` (`{start, end}`) inside that list. Nothing
  gives the two positions names; every reader has to remember `slot[0]` is start and
  `slot[1]` is end.
- The booker's name lives in a second, independent map, `bookerBySlot`
  (`InMemoryStore.java:14`), keyed by a *different* composite string,
  `roomId + "|" + date + "|" + start + "|" + end` (line 29).

So one logical booking is really three independent writes (`slots.add(...)` and
`bookerBySlot.put(...)` in `addSlot`, lines 28-29) that must agree on room, date, start,
and end for `bookerFor` to ever find the right name again. Cancel and reschedule
(`InMemoryStore.removeSlot`, lines 33-47) have to redo the same key-string construction
and remove from both maps separately — there is no single method or object whose job is
"delete this booking," only "delete this row from this map, and also that row from that
other map, and don't typo the key."

**Operations.** What can a caller do, and what goes in and out?

`RequestHandler` exposes four operations, all string-in/string-out:

- `createBooking(room, date, start, end, user)` → `"OK: ..."` or `"ERROR: ..."`.
- `cancelBooking(room, date, start, end)` → `"OK: ..."` or `"ERROR: ..."`.
- `rescheduleBooking(room, date, oldStart, oldEnd, newStart, newEnd)` → `"OK: ..."` or
  `"ERROR: ..."`. Note it takes no `user` — the booker is looked up from the old slot.
- `listBookings(room, date)` → a formatted multi-line string, or a "no bookings" string.

Each of `createBooking`, `cancelBooking`, and `rescheduleBooking` independently re-parses
`"HH:MM"` strings into minutes with its own copy of the same `split(":")` /
`Long.parseLong` block (`RequestHandler.java:13-25`, `46-58`, `69-88`). There is no
`parseTime` helper; the parsing logic is triplicated.

**Structure.** What classes exist, what does each own, and who holds a reference to whom?

- `ReservationApp` — demo driver; owns one `RequestHandler`.
- `RequestHandler` — owns one `InMemoryStore` (`RequestHandler.java:10`). Does the string
  parsing, the overlap check for `createBooking`, and the response formatting itself.
- `InMemoryStore` — owns the two `HashMap`s described above. Held only by
  `RequestHandler`.
- `BookingPolicy` — a standalone class implementing business hours, max length, and
  overlap (`validate`, lines 19-35). **Nothing holds a reference to it, and it is never
  instantiated.** A repo-wide search for `BookingPolicy` turns up exactly one hit: its own
  class declaration. `RequestHandler` never constructs one and never calls `validate`,
  `isWithinBusinessHours`, or `isWithinMaxLength`.

Reference graph is actually `ReservationApp → RequestHandler → InMemoryStore`, a
three-class chain, not the four-box pipeline the diagram in DESIGN.md draws.
`BookingPolicy` sits off to the side, referenced by no one — the design document's
central claim ("every booking request is validated by `BookingPolicy` before it reaches
storage") is false of the code as it stands.

**The no-double-booking invariant.** Where is it enforced? Name every place a check
happens, say what each one actually checks, and trace one reschedule request through the
code from the entry point to storage.

Three places look like they check this; only one of them runs, and it doesn't cover every
path:

1. `BookingPolicy.validate` (`BookingPolicy.java:29-33`) loops the existing slots and
   calls `overlaps`. This is the "real" overlap check the design describes — and it is
   dead code. Nothing calls it.
2. `RequestHandler.createBooking` (`RequestHandler.java:30-36`) has its own inline
   overlap loop, `startMinutes < slot[1] && slot[0] < endMinutes`, checked against
   `store.slotsFor(room, date)` before calling `store.addSlot`. This is a second,
   independent reimplementation of the same test as `BookingPolicy.overlaps`, and it is
   the only overlap check that actually executes — and only for `createBooking`.
3. `InMemoryStore.addSlot` (`InMemoryStore.java:23-27`) checks only for an **exact**
   duplicate interval (`slot[0] == start && slot[1] == end`), not overlap. This runs on
   every insert, including from `rescheduleBooking`.

Tracing one `rescheduleBooking(room, date, "10:00", "11:30", "13:00", "14:30")` call
(`RequestHandler.java:67-101`):

1. Entry: the six strings arrive at `rescheduleBooking`.
2. Lines 69-88: `oldStart`/`oldEnd`/`newStart`/`newEnd` are each split and parsed to
   minutes — the same parsing block duplicated from `createBooking`.
3. Line 89: the only validation performed is `newEnd > newStart`. No business-hours
   check, no max-length check, and **no overlap check against any other booking in the
   room** — `BookingPolicy` is not consulted and the inline loop from `createBooking` is
   not repeated here.
4. Line 93: `store.bookerFor(room, date, oldStartMinutes, oldEndMinutes)` looks up the
   booker by reconstructing the composite key; if the old interval isn't an exact match,
   this returns `null` and the request is rejected.
5. Line 98: `store.removeSlot(...)` deletes the old interval from `slotsByRoomDate` and
   the old entry from `bookerBySlot`.
6. Line 99: `store.addSlot(...)` inserts the new interval. Inside `addSlot`
   (`InMemoryStore.java:23-27`), the only check is for an exact duplicate of the *new*
   interval — not for overlap with any third booking already sitting in that room's list.
7. Line 100: an unconditional `"OK: moved ..."` is returned.

So a reschedule that lands exactly on top of a third, unrelated existing booking is
accepted and stored — the invariant DESIGN.md promises ("a booking may not overlap
another booking for the same room on the same day") is not enforced on this path at all.
`RequestHandlerTest.rescheduleMovesABooking` (`RequestHandlerTest.java:59-67`) only moves
a booking into an empty slot, so this gap is untested and the suite stays green.

---

## Milestone 2: Two design problems

Two problems. For each one, fill in all three parts.

### Problem 1

**The problem.** Representational gap. "A booking" is not a type anywhere in the code —
it is three unlinked artifacts that happen to agree: a raw `long[2]` in a list, and a
name in a *second* map keyed by a *different* hand-built string, with the room and date
never stored on the booking at all, only smeared across both keys.

**Where in the code.** `InMemoryStore.java`: the two fields `slotsByRoomDate` and
`bookerBySlot` (lines 13-14), and `addSlot` (16-31), `removeSlot` (33-47), and
`bookerFor` (57-59), each of which reconstructs the composite key string from scratch
and has to touch both maps in lockstep. `RequestHandler.rescheduleBooking`
(lines 93, 98-99) depends on this: it finds the booker by rebuilding the old interval's
key, deletes from both maps, then inserts fresh rows into both maps.

**What it makes expensive.** Already goes wrong today: because a booking is only
findable by an exact string match on `room|date|start|end`, `bookerFor` (`InMemoryStore
.java:57-59`) has no notion of the booking's *identity* independent of its current
interval — it can only be found by knowing that interval byte-for-byte. Concretely, a
future feature already on the roadmap in `DESIGN.md`'s "Planned next" — recurring
bookings — needs a per-booking attribute (a series id) that outlives a single interval.
Adding it means adding a *third* parallel map keyed by yet another hand-assembled
string, and updating `addSlot`/`removeSlot`/`rescheduleBooking` to keep three maps
consistent instead of two, with no compiler check that a change to one key format was
mirrored in the others.

### Problem 2

**The problem.** Missing boundary. `DESIGN.md`'s diagram claims "storage is only ever
reached from the handler, and only after the policy has approved the request" — but
nothing in the code makes that true. There is no single gate a booking write must pass
through; each operation decides for itself whether to validate, and one of them
decided not to.

**Where in the code.** `RequestHandler.rescheduleBooking` (lines 89-99) checks only
`newEndMinutes <= newStartMinutes` and then calls `store.removeSlot(...)` /
`store.addSlot(...)` directly — no call to `BookingPolicy` (which nothing in the
codebase ever calls) and no repeat of the overlap loop that `createBooking`
(lines 30-36) hand-rolls for itself. Compare the two methods side by side: `createBooking`
and `rescheduleBooking` both end at the same `InMemoryStore`, but only one of them
stands anything in front of it.

**What it makes expensive.** Already goes wrong today: a reschedule can move a booking
directly on top of a third, unrelated existing booking in the same room, and the call
returns `"OK: moved ..."` — silently violating the one invariant `BookingPolicy` exists
to name. This ships green because the only reschedule test,
`RequestHandlerTest.rescheduleMovesABooking` (`RequestHandlerTest.java:59-67`), moves
into an empty slot. Going forward, every operation `DESIGN.md`'s "Planned next" promises
— recurring bookings, midnight-crossing bookings, per-building hours — needs a human to
remember to re-insert the same checks, in the same order, at the top of a new method.
Nothing catches it if they don't; this bug is proof that at least one contributor
already forgot, and the next one has no reason to notice they're about to do it again.

---

## Milestone 3: Two alternative decompositions

Two different ways to carve up this system. A different split of responsibility, not a
list of local code fixes. Read the handout's appendix before writing this section.

### Alternative A

**The decomposition.** What are the pieces, what does each own, and where do the rules
live?

**One tradeoff.** Something this option actually costs. "No real downside" is not a
tradeoff.

### Alternative B

**The decomposition.**

**One tradeoff.**

### Preference

Which one, and under what conditions? Say what the choice depends on, and what would
make you pick the other one instead.
