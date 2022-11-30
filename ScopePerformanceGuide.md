### Avoid outputting intermediate results in final production scripts. 
it is very common case to output the intermidate results while initial debugging, but it would have negative impact on query performance, as it 1) breaks the execution pipeline and 2) forces the system to introduce an intermediate stage.
### Break independent commands into separte scripts
Scope job writes all the results at the end of the entire successfuly execution phase. One won't get results for some portion of the master script until all the computations are complete. Moreover, the increase of vertexes may negatively impact vertext scheduling
### Avoid Data Skew
data skew is when there may be a large number of distinct values in a column, but a small number of those distinct values have a high frequency(Pareto principle in data). This often happens for GROUP BY, REDUCE, COMBINE and JOIN, as the system will send all the rows with same value for the aggregation key to the same vertext, which would lead to the time out of the vertext.
### Avoid Low Distinctness
It occurs when the number of distinct values in a column is very low, as mentioned above, the system would send the rows with same key into one vertext, if there were few distinct values, it would send to few vertexes with huge data. To deal with low distinctness, there are two options, 1) use a different key - selecting a different key entirely or by creating a compound key consisting of several columns. 2)  use a recursive aggregate operator, this allows the system to do the aggregation in multiple rounds, like SUM and COUNT are recursive.
### Physical design of Structured Streams
When writing data into a structured stream, the choices of CLUSTERED BY and SORTED BY are crucial for desirable performance of jobs.
### RANGE partitioning versus HASH partitioning
HASH partitioning has better write performance than RANGE, as it gives the user more control of read parallelism, since the number of hash buckets and the number of vertices reading them are tightly coupled. If left unspecified SCOPE will use RANGE CLUSTERED BY
### Use sytem built-in operators instead of User-Defined Operators
Use GROUP BY instead of REDUCE; use JOIN instead of COMBINE; use system aggregate functions instead of user-defined aggregate functions if possible.
### Don't do these things in UDO
1. Do not use UDOs if there are system built-in equivalents
2. Do not use the column names to access the columns from the Row object, as the string based indexer on the Row object is much slower than the int based indexer
3. Do not block the pipeline, if the UDO caches all/some of the input rows before yield return any row, you are blocking the pipeline
4. Do not add extra instructions into the loop, e.g. try to cache the result of StringColumn.String, instead of calling those functions many times
5. Do not create row objects for yield return inside the loop; row object should be reused and just copy column values
6. Do not add sort requirement in the UDO implementation 
7. Do not use static variables inside the UDOs
8. Avoid data explosion, as the system does not know the fact that the output of the UDO is much larger than its input
### Use INNER JOIN or SEMIJOIN instead of OUTER JOIN
1. INNER JOIN produces less data than OUTER JOIN
2. if one side of join is small, the system will be able to choose broadcast join graph without repartioning the bigger input, repartitioning is a costly operation
3. LEFT OUTER and RIGHT OUTER JOINs allow broadcast join only one way, as the LEFT  OUTER JOIN can only get the broadcast join optimization provided that the "small" side of the join is the right hand side
4. As a result of the possibility the column of output becomes NULL rather than a value in some input row, any partition property or sortedness on those columns is destroyed.
5. For INNER JOIN, most of the structural properties are preserved, which avoids additional sorting, repartitioning.
### Do not use Resource files in PROCESS to simulate INNER JOIN
Automatically broadcast cannot be achieved if resources are used. Resources will need to be sent to every single vertex in the current job, even though it is only used by a particular operator.

Distributiing large resource files to all the vertex has impacts on overall cluster network utilization, also on the job latency. Cosmos is designed for Single-Instruction Multiple Data model, and use data partitioning and pipelineing to achieve parallel computation.

### Degree of Parallelism is not a magic bullet
Increasing the DOP will improve the performance to some extent, but eventually it will hit a plateau and the performance will start to drop.

One of the most important factors is that cluster resources are limited.

### Use UNION ALL instead of UNION, if appropriate
### Pass-through columns in UDO could avoid unnecessary repartitioning