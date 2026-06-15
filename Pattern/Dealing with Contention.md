 - **Contention** occurs when multiple processes compete for the same resource at the same time, like booking the last concert ticket or bidding on an auction item. Without proper handling, you get race conditions, double-bookings, and inconsistent state.


**# The Race condition (PROBLEM)**
- Example: Consider buying concert tickets online. There's one seat left for The Weeknd, and Alice and Bob both want it. They each hit "Buy Now" in the same instant. The obvious way to handle a purchase is the one most of us would reach for first. Read the current seat count, check that it's above zero, and if it is, decrement it and sell the ticket.

-- Read the current count
SELECT available_seats FROM concerts WHERE concert_id = 'weeknd_tour';

-- The app checks available_seats > 0, then writes the new value back
UPDATE concerts
SET available_seats = available_seats - 1
WHERE concert_id = 'weeknd_tour';

- For a single buyer this is exactly right. The trouble starts when Alice and Bob run it at the same moment. Alice's request reads one seat available. A fraction of a millisecond later, before Alice has written anything back, Bob's request reads the same count and also sees one seat. Both check the number they just read, both conclude there's a seat to sell, and both move on to charge a card. Alice's update commits first and the count drops to zero. Bob's update commits right after, decrements again, and the count slides to negative one.

- By the time the dust settles, both cards have been charged $500 and both buyers have a  confirmation email for the same seat. Alice and Bob show up to the concert each thinking they own Row 5, Seat 12. One of them is getting kicked out, and the venue is stuck issuing a refund while dealing with two very angry customers.

	![[Screenshot 2026-06-14 at 1.45.53 AM.png]]

- The real culprit is the gap between two steps the naive code treats as one. Reading the count and writing the new value back aren't a single action, and in between them the world can change. Bob slips into that gap, reads a seat count Alice is about to invalidate, and acts on a number that's already stale. The window is tiny, microseconds in memory and milliseconds over a network, but it's all it takes for both requests to decide they've won the same seat.

- With 10,000 concurrent users hitting the same resource, even small race condition windows create massive conflicts. As you continue to grow, it's likely that you'll need to coordinate across multiple nodes which adds to the complexity.

To get this right, we need some form of synchronization.

**# Problems Breakdowns with Dealing with contention Problem**
1. Ticketmaster
2. Rate limiter
3. Online auction


**# The SOLUTION**
- It's a **lost update**, the classic failure of a **read-modify-write** cycle. Two requests read the same value, both act on it, and both write back, so one silently overwrites the other.
- Every fix below is a way to close the gap between reading a value and acting on it.wdfsda

1. Conditional Writes
- You decrement a count if a seat if left, or mark the order shipped if it has not been cancelled. When your rule is an if about the current data, the database can check it and make the change in a single statement, no locks or version numbers required.

- Booking a seat is exactly this kind of rule. "Decrement the count, but only if a seat is left."

	UPDATE concerts
	SET available_seats = available_seats - 1
	WHERE concert_id = 'weeknd_tour'
	  AND available_seats > 0;

- This is already safe under concurrency. No matter how many people try to grab the last ticket at once, only one will succeed. The reason is that the database won't let two updates change the same row at the same time. It makes them take turns, and the one that has to wait doesn't get to act on the value it first read
- Example: 





