# 09. Spark Catalyst Optimizer & Tungsten

*Source references: `apache/spark/sql/catalyst` and `apache/spark/sql/core/src/main/java/org/apache/spark/sql/execution`*

Apache Spark 1.0 (2014) was fast because it processed data in memory.
Apache Spark 2.0+ (2016-2026) is fast because it acts like a literal **Database Compiler**. 

To reach extreme performance, Databricks had to rewrite everything in "Project Tungsten," completely destroying the standard JVM limits.

---

## 1. The Catalyst AST Compiler

When a data engineer writes this PySpark code:
```python
df.join(users, "id").filter(col("age") > 30).select("name")
```
Spark does not execute a join, a filter, and a select. 
Instead, it builds an **Abstract Syntax Tree (AST)**—a logical representation of the query.

Catalyst comprises four phases:
1. **Analysis:** Resolves column names (Is `age` actually an integer in the schema?).
2. **Logical Optimization:** Applies mathematical rules. For example, **Predicate Pushdown**: Catalyst physically rewrites the AST to execute the `filter(age > 30)` *before* the `join`. 
3. **Physical Planning:** Generates multiple execution strategies (e.g., Should I use a SortMergeJoin or a BroadcastHashJoin?).
4. **Cost Model:** Estimates network shuffle costs and picks the cheapest Physical Plan.

### Rule Evaluation Source Code
If you look at the Catalyst optimizer rules (e.g., `PushDownPredicates.scala`), the code uses pattern matching to recursively rewrite the AST tree:

```scala
// Conceptual rule rewriting the AST tree
object PushPredicateThroughJoin extends Rule[LogicalPlan] {
  def apply(plan: LogicalPlan): LogicalPlan = plan transform {
    
    // If we see a Filter sitting on top of an Inner Join...
    case Filter(condition, Join(left, right, Inner, None)) =>
      
      // We push the condition down directly into the left or right table
      val newLeft = if (canEvaluate(condition, left)) Filter(condition, left) else left
      val newRight = if (canEvaluate(condition, right)) Filter(condition, right) else right

      Join(newLeft, newRight, Inner, None)
  }
}
```

---

## 2. Project Tungsten: Defeating the JVM

After Catalyst optimizes the query plan, Spark has to physically execute it on a billion rows of data. 

In Spark 1.0, every row was an actual Java Object. If your table had 3 string columns, every single row had 3 Java String object headers, 8 bytes of padding, and GC tracking overhead. A 100MB Parquet file would inflate to 500MB of Java RAM, immediately triggering Garbage Collection pauses.

**Tungsten Memory Format** fixes this. It bypasses the Java Object model entirely. Spark asks the OS for a massive chunk of raw memory (using `sun.misc.Unsafe`) and packs the rows in as raw bytes, exactly like a C++ `struct`.

### Whole-Stage Code Generation

If you have a pipeline that Filters, then Maps, then Aggregates, Spark 1.0 used "Volcano" execution: calling functional `iterator.next()` over and over, millions of times. 

Tungsten uses **Whole-Stage Code Generation**. At runtime, Catalyst converts your entire PySpark pipeline into a single, massive string of highly unrolled Java source code, compiles it into bytecode on the fly (using the Janino compiler), and executes it.

It fuses all loops into a single CPU-optimized `while` loop:

```java
// What Spark 1.0 did (Millions of virtual method calls)
for (Row r : filter.run(table)) {
    for (Row r2 : map.run(r)) {
      agg.run(r2);
    }
}

// What Tungsten Generates and Compiles AT RUNTIME (Fused Loop)
int count = 0;
while (batch.hasNext()) {
    long memoryPointer = batch.nextPointer();
    
    // Read raw bytes using unsafe Memory pointers (No Java Objects!)
    int age = Platform.getInt(memoryPointer + 16); 
    if (age > 30) {
        count++; // 0 method calls. Pure CPU registers.
    }
}
```

By keeping the inner loops tight enough to fit inside the CPU's **L1/L2 hardware Cache**, Tungsten ensures that the CPU never has to stall waiting for Main Memory (RAM).
