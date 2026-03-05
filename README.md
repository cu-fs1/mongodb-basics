# MongoDB Basics

This README is a practical, beginner-to-intermediate guide to MongoDB.

## 1) What MongoDB Is

MongoDB is a **NoSQL document database**. Instead of rows and columns, it stores data as flexible JSON-like documents (BSON under the hood).

- **Database**: Top-level container (like a project namespace).
- **Collection**: Group of related documents (like a table).
- **Document**: A single record (like a row), represented as key-value pairs.

Example document:

```json
{
	"_id": "64ef...",
	"name": "Pritam",
	"email": "pritam@example.com",
	"age": 27,
	"skills": ["javascript", "mongodb"],
	"address": {
		"city": "Kolkata",
		"country": "India"
	}
}
```

## 2) Why MongoDB

- Flexible schema: Documents in the same collection can vary.
- Fast development: JSON-like model maps naturally to app objects.
- Rich query language and aggregation framework.
- Built for scale with replication and sharding.

## 3) Core MongoDB Terms You Must Know

- **BSON**: Binary JSON format used internally by MongoDB.
- **`_id`**: Unique primary key for every document (auto-created if not provided).
- **ObjectId**: Default `_id` type.
- **Index**: Data structure that speeds reads.
- **Replica Set**: Group of nodes for high availability.
- **Shard**: Horizontal partition of data for scale.
- **Aggregation Pipeline**: Stage-based data processing.

## 4) SQL vs MongoDB Mapping

- Database (SQL) -> Database (MongoDB)
- Table -> Collection
- Row -> Document
- Column -> Field
- JOIN -> `$lookup` (or denormalized design)
- `GROUP BY` -> `$group`

## 5) Installation and Setup (Mongoose + MongoDB)

If you are using Mongoose, install both Node dependencies and a MongoDB server (local or Atlas).

### Step 1: Install Node.js

Install Node.js LTS (includes `npm`).

Check:

```bash
node -v
pnpm -v
```

### Step 2: Create Node Project

```bash
pnpm init
```

### Step 3: Install Dependencies

```bash
pnpm add mongoose dotenv
```

### Step 4: Host on MongoDB Atlas (Recommended)

Use **MongoDB Atlas** — the official cloud-hosted MongoDB service. It has a free tier and requires no local installation.

1. Go to [https://www.mongodb.com/cloud/atlas](https://www.mongodb.com/cloud/atlas) and sign up.
2. Create a free **M0 cluster** (free forever, no credit card needed).
3. Under **Database Access**, create a database user with a username and password.
4. Under **Network Access**, add your current IP address (or `0.0.0.0/0` for development).
5. Go to **Clusters → Connect → Drivers**, select Node.js, and copy the connection string.

The connection string looks like:

```
mongodb+srv://<username>:<password>@cluster0.xxxxx.mongodb.net/<dbname>?retryWrites=true&w=majority
```

Replace `<username>`, `<password>`, and `<dbname>` with your values.

### Step 5: Add Environment Variable

Create `.env`:

```env
MONGODB_URI=mongodb://127.0.0.1:27017/school
```

For Atlas, use your Atlas URI in `MONGODB_URI`.

### Step 6: Minimal Connection Test

Create `index.js`:

```javascript
import "dotenv/config";
import mongoose from "mongoose";

async function start() {
	try {
		await mongoose.connect(process.env.MONGODB_URI);
		console.log("MongoDB connected with Mongoose");
		await mongoose.connection.close();
	} catch (error) {
		console.error("Connection failed:", error.message);
		process.exit(1);
	}
}

start();
```

Set package type (for `import` syntax):

```json
{
	"type": "module"
}
```

Run:

```bash
node index.js
```

If it prints `MongoDB connected with Mongoose`, installation is correct.

## 6) Basic Query Workflow with Mongoose

Assume you already created a `Student` model.

Insert:

```javascript
await Student.create({ name: "Asha", age: 20, course: "CS", marks: 89 });

await Student.insertMany([
	{ name: "Ravi", age: 21, course: "Math", marks: 76 },
	{ name: "Meera", age: 19, course: "CS", marks: 92 }
]);
```

Read:

```javascript
await Student.find();
await Student.find({ course: "CS" });
await Student.find({ marks: { $gte: 80 } });
await Student.findOne({ name: "Asha" });
```

Projection (pick fields):

```javascript
await Student.find({ course: "CS" }).select("name marks -_id");
```

Update:

```javascript
await Student.updateOne({ name: "Asha" }, { $set: { marks: 91 } });
await Student.updateMany({ course: "CS" }, { $inc: { marks: 2 } });
```

Delete:

```javascript
await Student.deleteOne({ name: "Ravi" });
await Student.deleteMany({ marks: { $lt: 40 } });
```

## 7) Query Operators

### Comparison

- `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`

Example (Mongoose):

```javascript
await Student.find({ marks: { $gte: 80, $lte: 95 } });
```

### Logical

- `$and`, `$or`, `$not`, `$nor`

Example (Mongoose):

```javascript
await Student.find({
	$or: [{ course: "CS" }, { marks: { $gt: 90 } }]
});
```

### Element / Array

- `$exists`, `$type`, `$all`, `$elemMatch`, `$size`

Example (Mongoose):

```javascript
await User.find({ skills: { $all: ["mongodb", "nodejs"] } });
```

## 8) Cursor Operations

```javascript
await Student.find().sort({ marks: -1 }).limit(5).skip(0);
```

- `sort`: Asc `1`, Desc `-1`
- `limit`: max docs returned
- `skip`: pagination offset

### Advanced Mongoose Query Patterns

```javascript
// Case-insensitive search
await Student.find({ name: { $regex: "^ash", $options: "i" } });

// Count documents
const totalCS = await Student.countDocuments({ course: "CS" });

// Exists / missing field
await Student.find({ phone: { $exists: false } });

// Lean query (faster read, plain JS objects)
const rows = await Student.find({ marks: { $gte: 80 } }).lean();

// Pagination helper
const page = 1;
const pageSize = 10;
const items = await Student.find()
	.sort({ createdAt: -1 })
	.skip((page - 1) * pageSize)
	.limit(pageSize);
```

## 9) Indexing (Very Important)

Without indexes, MongoDB scans many documents.

Create index:

```javascript
studentSchema.index({ email: 1 }, { unique: true });
studentSchema.index({ course: 1, marks: -1 });
```

Check indexes:

```javascript
await Student.collection.getIndexes();
```

Inspect query plan:

```javascript
await Student.find({ course: "CS" }).explain("executionStats");
```

## 10) Aggregation Pipeline

Aggregation transforms and analyzes data in stages.

Common stages:

- `$match` filter
- `$group` aggregate
- `$project` reshape
- `$sort`
- `$limit`
- `$lookup` join-like
- `$unwind` flatten arrays

Example: average marks by course.

```javascript
await Student.aggregate([
	{ $match: { marks: { $gte: 50 } } },
	{
		$group: {
			_id: "$course",
			avgMarks: { $avg: "$marks" },
			count: { $sum: 1 }
		}
	},
	{ $sort: { avgMarks: -1 } }
]);
```

## 11) Data Modeling (Schema Design)

MongoDB schema is flexible, but design still matters.

### Embed vs Reference

- **Embed** when related data is read together and bounded in size.
- **Reference** when relation is large, shared, or updated separately.

Embedded example:

```json
{
	"name": "Order-1001",
	"customer": { "id": 1, "name": "Asha" },
	"items": [
		{ "sku": "P1", "qty": 2, "price": 500 },
		{ "sku": "P2", "qty": 1, "price": 900 }
	]
}
```

Referenced example:

```json
{
	"orderId": "1001",
	"customerId": "c001",
	"itemIds": ["i1", "i2"]
}
```

### Practical schema tips

- Model around **query patterns first**.
- Keep documents under 16 MB.
- Avoid unbounded arrays in hot documents.
- Add validation rules where possible.

## 12) Validation

With Mongoose, enforce structure directly in the schema definition.

```javascript
import mongoose from "mongoose";

const studentSchema = new mongoose.Schema({
	name: { type: String, required: true },
	email: { type: String, required: true },
	age: { type: Number, min: 0 }
});

const Student = mongoose.model("Student", studentSchema);
```

## 13) Transactions

MongoDB supports multi-document ACID transactions (on replica sets/sharded clusters).

Use transactions when one operation depends on another and all must succeed together.

## 14) Replication and High Availability

- Replica set has **primary** and **secondary** nodes.
- Writes go to primary; secondaries replicate data.
- On primary failure, automatic election picks new primary.

Benefits:

- High availability
- Fault tolerance
- Read scaling (read preferences)

## 15) Sharding and Horizontal Scaling

Sharding splits data across multiple servers by a **shard key**.

- Needed when one server cannot handle data size or throughput.
- Choose shard key carefully to avoid hot shards.

## 16) Security Essentials

- Enable authentication and role-based access control.
- Use TLS for encryption in transit.
- Use encryption at rest where possible.
- Restrict network access with firewall/VPC.
- Never expose an unauthenticated database publicly.

## 17) Backup and Restore

Common tools:

- `mongodump` for logical backup.
- `mongorestore` for restore.
- Atlas offers managed snapshots and point-in-time recovery options.

## 18) Useful MongoDB Tools

- **MongoDB Compass**: GUI for browsing data and running queries.
- **mongosh**: Command line shell.
- **MongoDB Atlas**: Managed cloud MongoDB.

## 19) MongoDB in Applications

Popular drivers:

- Node.js (`mongodb`, `mongoose`)
- Python (`pymongo`)
- Java (`mongodb-driver-sync`)

If you will use **Mongoose** with Node.js, start like this.

Install:

```bash
pnpm add mongoose
```

### Connect to MongoDB

```javascript
import mongoose from "mongoose";

const uri = process.env.MONGODB_URI || "mongodb://127.0.0.1:27017/school";

await mongoose.connect(uri);
console.log("MongoDB connected");
```

### Define Schema and Model

```javascript
import mongoose from "mongoose";

const studentSchema = new mongoose.Schema(
	{
		name: { type: String, required: true, trim: true },
		email: { type: String, required: true, unique: true, lowercase: true },
		marks: { type: Number, min: 0, max: 100, default: 0 },
		course: { type: String, index: true }
	},
	{ timestamps: true }
);

const Student = mongoose.model("Student", studentSchema);
export default Student;
```

### CRUD with Mongoose

```javascript
import Student from "./models/student.model.js";

// Create
const asha = await Student.create({
	name: "Asha",
	email: "asha@example.com",
	marks: 88,
	course: "CS"
});

// Read
const topStudents = await Student.find({ marks: { $gte: 80 } })
	.select("name marks course -_id")
	.sort({ marks: -1 })
	.limit(5);

// Update
await Student.updateOne({ email: "asha@example.com" }, { $inc: { marks: 2 } });

// Delete
await Student.deleteOne({ email: "asha@example.com" });
```

### Relations with `ref` and `populate`

```javascript
import mongoose from "mongoose";

const userSchema = new mongoose.Schema({
	name: String
});

const postSchema = new mongoose.Schema({
	title: String,
	author: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true }
});

const User = mongoose.model("User", userSchema);
const Post = mongoose.model("Post", postSchema);

const user = await User.create({ name: "Pritam" });
await Post.create({ title: "MongoDB Basics", author: user._id });

const posts = await Post.find().populate("author", "name");
```

### Transactions in Mongoose

```javascript
const session = await mongoose.startSession();

try {
	await session.withTransaction(async () => {
		await Student.create([{ name: "Meera", email: "meera@example.com" }], { session });
		await Student.updateOne({ email: "asha@example.com" }, { $inc: { marks: 1 } }, { session });
	});
} finally {
	await session.endSession();
}
```

Note: Transactions require replica set or Atlas cluster.

### Mongoose Middleware and Validation

```javascript
studentSchema.pre("save", function (next) {
	this.name = this.name.trim();
	next();
});
```

Use Mongoose validation for app-level checks and MongoDB indexes for uniqueness/performance.

## 20) Best Practices Checklist

- Create indexes for real query patterns.
- Use projections to reduce payload.
- Avoid frequent full collection scans.
- Keep schema consistent even if flexible.
- Use aggregation for server-side data processing.
- Monitor performance with `explain()` and metrics.
- Plan for backup, security, and scaling from day one.
- Keep Mongoose schemas strict and explicit for predictable data.
- Handle DB connection and query errors with centralized middleware.

## 21) Common Beginner Mistakes

- Treating MongoDB exactly like SQL without rethinking model.
- Overusing references when embedding is better.
- Missing indexes on filter/sort fields.
- Storing huge unbounded arrays in one document.
- Ignoring validation and security in development.

## 22) Quick Practice Commands

```javascript
import mongoose from "mongoose";

const productSchema = new mongoose.Schema({
	name: String,
	category: String,
	price: Number,
	stock: Number
});

const Product = mongoose.model("Product", productSchema);

await Product.insertMany([
	{ name: "Laptop", category: "Electronics", price: 70000, stock: 12 },
	{ name: "Mouse", category: "Electronics", price: 800, stock: 100 },
	{ name: "Book", category: "Education", price: 450, stock: 50 }
]);

await Product.find({ category: "Electronics" });
await Product.updateOne({ name: "Mouse" }, { $inc: { stock: -1 } });

await Product.aggregate([
	{ $group: { _id: "$category", totalStock: { $sum: "$stock" } } }
]);
```

## 23) How MongoDB Works Internally (Low Level)

### Storage Engine: WiredTiger

MongoDB uses **WiredTiger** as its default storage engine (since v3.2).

- Data is stored in a compressed, columnar format on disk inside `.wt` files.
- WiredTiger uses a **B-tree** data structure to organize documents and indexes on disk.
- Compression is applied per block (snappy by default, zlib/zstd available), reducing disk usage significantly.

### BSON on Disk

When you insert a document, MongoDB serializes it to **BSON** (Binary JSON) before writing it to disk.

- BSON is a binary-encoded format that adds type information and length prefixes to each field.
- This makes field traversal and size calculation fast without parsing the entire document.
- Each document lives inside a **collection file**, identified by its `_id`.

### Write-Ahead Journal (WAJ / Journaling)

MongoDB guarantees durability through a **write-ahead log**.

1. Every write is first appended to the journal (a `.journal` file) before being applied to the data files.
2. If the server crashes mid-write, on restart MongoDB replays the journal to recover to a consistent state.
3. WiredTiger flushes the journal to disk every 50 ms by default (`storage.journal.commitIntervalMs`).

### Index Structure: B-Trees

Every index in MongoDB is stored as a **B-tree** on disk.

- The leaf nodes of the B-tree hold the indexed field value and a pointer (RecordId) to the actual document.
- For a compound index `{ course: 1, marks: -1 }`, the B-tree is sorted first by `course` ascending, then by `marks` descending within each course.
- When a query matches the index prefix, MongoDB traverses the B-tree instead of scanning the full collection — this is why indexes matter so much.

### Document-Level Locking

WiredTiger uses **document-level (optimistic) concurrency control**, not collection-level locks.

- Multiple writes to different documents in the same collection can proceed concurrently.
- Uses MVCC (Multi-Version Concurrency Control): readers see a consistent snapshot of data without blocking writers.
- Conflicts on the **same document** are serialized.

### WiredTiger Cache

WiredTiger maintains an in-memory cache (default: 50% of RAM − 1 GB).

- Frequently accessed pages (B-tree nodes, documents, index entries) are kept in the cache.
- Dirty pages (modified but not yet flushed) are written to disk during checkpoints (every 60 seconds by default).
- This is why the first query after startup is slower — the cache is cold.

### Query Planning and Execution

When you run a query, MongoDB's **query planner** picks an execution strategy:

1. **Candidate plan selection**: The planner considers all applicable indexes and generates candidate plans.
2. **Trial run**: Candidate plans are run concurrently for a small number of documents to measure performance.
3. **Winning plan**: The fastest plan is selected and cached for that query shape.
4. **Execution**: The winning plan is carried out — either an index scan (`IXSCAN`) or a collection scan (`COLLSCAN`).

You can inspect this with:

```javascript
await Student.find({ course: "CS" }).explain("executionStats");
```

Key fields to look at in the output:

- `winningPlan.stage`: `IXSCAN` (good) vs `COLLSCAN` (potentially slow)
- `totalDocsExamined`: how many documents were scanned
- `totalKeysExamined`: how many index entries were scanned
- `executionTimeMillis`: how long the query took

### Write Path Summary

```
Client write
    ↓
MongoDB driver (BSON serialization)
    ↓
mongod process receives operation
    ↓
WiredTiger appends to journal (WAL)
    ↓
Document written / updated in WiredTiger cache
    ↓
B-tree indexes updated in cache
    ↓
Background checkpoint flushes dirty pages to .wt data files
```

## 24) What to Learn Next

1. Advanced indexing (TTL, text, wildcard, partial indexes)
2. Aggregation deep dive (`$facet`, `$bucket`, `$setWindowFields`)
3. Atlas monitoring and performance tuning
4. Change streams and event-driven architecture
5. Schema versioning and migration strategies
