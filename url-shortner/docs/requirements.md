# Functional Requirements
- Given a long URL, generate a unique short URL.
- Short URL must be combination of numbers (0-9) and characters (a-z, A-Z)
- When a short URL is accessed, redirect the user to the original URL.
- User must be authenticated before they can generate short URL.
- Allow users to create custom short URL (if available based on the uniqueness).
- Short URL expires with default and custom TTLs.
- Rate limit on number of short URL an account can have
- Track click count for each URL.

# Non-Functional Requirements
- High availability: The system must be highly available (e.g., 99.99%)
- Low Latency: Redirect request must complete under 50ms (p99).
- Uniqueness: Each short code maps to exactly one long URL, and short code never collide.
- Scalability: Should handle a large number of read and write requests (100:1 read-to-write ratio).
- Durability: Once a short URL is created, the mapping must not be lost.

---

# Back-of-the-envelop Estimation
- New link created - **10 million/day**
- 100:1 read/write ratio = **1 billion redirects/day**

## Writes (URL shortening):
- Average write QPS = 10,000,000 / (24 * 3600) ≈ **115 QPS** (steady state)
- peak write load (3x factor) = **350 QPS**

## Reads (Redirect):
- Average read QPS = 1,000,000,000 / (24 * 3600) ≈ **11,500 QPS** (steady state)
- Peak read load (3x factor) ≈ **35,000 QPS**

## Storage (5 years):
Each URL mapping stores:
- Short code: ~7 character = 7 bytes
- Long URL: average 200 character = 200 bytes
- user ID: UUID = 36 bytes
- Timestamp, metadata: ~50 bytes

**Total per record: ~300 bytes**

- URLs per year: 10M/day * 365 days = 3.65 billion URLs/year
- Storage per year = 3.65B * 300 bytes = ~1TB/year
- **5-year storage = ~5TB**

## Short Code Length:
Using Base62 encoding (a-z, A-Z, 0-9):
- 6 characters: 62^6 = 56.8 billion unique codes
- 7 characters: 62^7 = 3.5 trillion unique code

With 3.65 billion URLs per year, a **7 character code** provides ample head-room for decades of growth.
