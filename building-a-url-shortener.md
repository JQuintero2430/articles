# Building a URL Shortener

This design is intended to provide a solution within the context of a job interview, which is why we will leave aside some considerations that could affect the design in a real production environment.

With this in mind, the problem statement is as follows:
*"You are asked to design a URL shortener that returns the address `fake-company.io/XXXXXX`, so that we can provide a short URL service to our users, who in turn can share this short URL if they want to distribute the original address of their website."*

Given the context, the first thing worth asking is:

* How many characters will this shortener return to append to the path?
* What will be the encoding/hash base for those characters?
* Will the shortened URLs have a TTL, or will they remain indefinitely?
* Should we perform validations before returning the shortened URL?

Since validations are a design in themselves, we’ll leave them out of scope. For this case, we’ll use **6 characters** and work with **base 36 (a–z, 0–9)** to simplify the problem as much as possible and focus on the reasoning behind the shortener’s design, imagining that paths never expire.

---

## Calculating the possibility space

Each character has 36 possible values, and they are independent of each other.

Therefore, we must calculate `36^6`, which gives us **2,176,782,336 possible combinations**.

This means that, in the worst case, our database could end up holding just over **two billion records**.

---

## Database design

For this scenario, we could use an **SQL database** with at least the following columns:

* `short_path` → the generated hash.
* `original_url` → the original URL.
* `created_at` → the creation/assignment timestamp.
* `is_assigned` (boolean/flag) → indicates whether that path has already been assigned.

Since many short URLs will be queried more than once, it’s also worth considering:

* Applying a **CQRS pattern**, moving the read side to a NoSQL database for better scalability.
* Using a **key–value database on disk** (e.g., RocksDB) to handle simple lookups.
* Or, keeping everything in SQL but relying on an **in-memory database like Redis**, with a configurable TTL, to cache the most frequently queried paths.

---

## Path generation strategy

The first idea might be to use a randomizer to generate the path based on the original URL (which we’ll call `source`). However, this quickly introduces **collisions**, forcing us to implement a retry mechanism that can degenerate into almost infinite loops.

Therefore, relying solely on randomness is not a good idea.

Instead, a better strategy is to **precompute all possible combinations**, shuffle them with a *shuffle* algorithm, and finally persist them in our database.

This way:

* We avoid collisions.
* We can safely assign paths in order.
* We free up compute resources during critical production usage, since the heavy lifting is done in batches.

---

## Assignment flow

The system flow would look like this:

1. Prepare the database with all possible combinations, shuffled and persisted in a table.
2. Each record has a flag `is_assigned = false`.
3. When a user submits a URL to shorten:

   * Perform necessary validations (discussion for another time).
   * Lock the next free record using `SELECT ... FOR UPDATE` (SQL with locking).
   * Update the record:

     * Save the `source`.
     * Mark `is_assigned = true`.
     * Record the `created_at`.
   * Release the lock and return the `short_path` to the user.
4. When the user accesses `fake-company.io/{short_path}`, we run a `SELECT` using that value and redirect them to the `source`.

---

## Additional interview notes

* **Scalability:** this design can scale horizontally if we partition the combinations across multiple database nodes.
* **Consistency:** using `SELECT FOR UPDATE` ensures no collisions occur in concurrent environments.
* **Security:** it’s better to avoid assigning combinations in natural order (`000000`, `000001`, …) because it exposes predictability. This is why the shuffle is important.
* **Monitoring:** we can expose metrics to track how much of the namespace has already been consumed.

---

## Conclusion

With this design, we solve the problem of unique path assignment and ensure that each shortened URL has a consistent and scalable flow.

I hope this guide serves as a clear starting point to tackle, from a simple perspective, one of the most common questions in technical interviews, while also opening your mind to the various possibilities in system design.
