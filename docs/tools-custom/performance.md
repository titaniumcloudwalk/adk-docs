# Increase tool performance with parallel execution

<div class="language-support-tag">
  <span class="lst-supported">Supported in ADK</span><span class="lst-python">Python v1.10.0</span><span class="lst-typescript">TypeScript v0.2.0</span>
</div>

Starting with Agent Development Kit (ADK) version 1.10.0 for Python and version 0.2.0 for TypeScript, the framework
attempts to run any agent-requested 
[function tools](/adk-docs/tools-custom/function-tools/) 
in parallel. This behavior can significantly improve the performance and
responsiveness of your agents, particularly for agents that rely on multiple
external APIs or long-running tasks. For example, if you have 3 tools that each
take 2 seconds, by running them in parallel, the total execution time will be
closer to 2 seconds, instead of 6 seconds. The ability to run tool functions
parallel can improve the performance of your agents, particularly in the
following scenarios:

-   **Research tasks:** Where the agent collects information from multiple
    sources before proceeding to the next stage of the workflow.
-   **API calls:** Where the agent accesses several APIs independently, such
    as searching for available flights using APIs from multiple airlines.
-   **Publishing and communication tasks:** When the agent needs to publish
    or communicate through multiple, independent channels or multiple recipients.

However, your custom tools must be built with asynchronous execution support to
enable this performance improvement. This guide explains how parallel tool
execution works in the ADK and how to build your tools to take full advantage of
this processing feature.

!!! warning
    Any ADK Tools that use synchronous processing in a set of tool function
    calls will block other tools from executing in parallel, even if the other
    tools allow for parallel execution.

## Build parallel-ready tools

Enable parallel execution of your tool functions by defining them as
asynchronous functions. In Python code, this means using `async def` and `await`
syntax which allows the ADK to run them concurrently in an `asyncio` event loop.
In TypeScript, use `async` functions with `await` syntax which leverages
`Promise.all()` for concurrent execution.
The following sections show examples of agent tools built for parallel
processing and asynchronous operations.

### Example of http web call

The following code example show how to modify the `get_weather()` function to
operate asynchronously and allow for parallel execution:

=== "Python"

    ```python
    async def get_weather(city: str) -> dict:
        async with aiohttp.ClientSession() as session:
            async with session.get(f"http://api.weather.com/{city}") as response:
                return await response.json()
    ```

=== "TypeScript"

    ```typescript
    import { FunctionTool } from '@google/adk';
    import { z } from 'zod';

    const getWeatherTool = new FunctionTool({
        name: 'get_weather',
        description: 'Get current weather for a city. Optimized for parallel execution.',
        parameters: z.object({
            city: z.string().describe('Name of the city'),
        }),
        execute: async ({ city }) => {
            const response = await fetch(`http://api.weather.com/${city}`);
            return await response.json();
        },
    });
    ```

### Example of database call

The following code example show how to write a database calling function to
operate asynchronously:

=== "Python"

    ```python
    async def query_database(query: str) -> list:
        async with asyncpg.connect("postgresql://...") as conn:
            return await conn.fetch(query)
    ```

=== "TypeScript"

    ```typescript
    import { FunctionTool } from '@google/adk';
    import { z } from 'zod';
    import { Pool } from 'pg';

    const pool = new Pool({ connectionString: 'postgresql://...' });

    const queryDatabaseTool = new FunctionTool({
        name: 'query_database',
        description: 'Execute a database query',
        parameters: z.object({
            query: z.string().describe('SQL query to execute'),
        }),
        execute: async ({ query }) => {
            const client = await pool.connect();
            try {
                const result = await client.query(query);
                return result.rows;
            } finally {
                client.release();
            }
        },
    });
    ```

### Example of yielding behavior for long loops

In cases where a tool is processing multiple requests or numerous long-running
requests, consider adding yielding code to allow other tools to execute, as
shown in the following code sample:

=== "Python"

    ```python
    async def process_data(data: list) -> dict:
        results = []
        for i, item in enumerate(data):
            processed = await process_item(item)  # Yield point
            results.append(processed)

            # Add periodic yield points for long loops
            if i % 100 == 0:
                await asyncio.sleep(0)  # Yield control
        return {"results": results}
    ```

=== "TypeScript"

    ```typescript
    import { FunctionTool } from '@google/adk';
    import { z } from 'zod';

    const processDataTool = new FunctionTool({
        name: 'process_data',
        description: 'Process a list of data items',
        parameters: z.object({
            data: z.array(z.unknown()).describe('Data items to process'),
        }),
        execute: async ({ data }) => {
            const results = [];
            for (let i = 0; i < data.length; i++) {
                const processed = await processItem(data[i]); // Yield point
                results.push(processed);

                // Add periodic yield points for long loops
                if (i % 100 === 0) {
                    await new Promise(resolve => setImmediate(resolve)); // Yield control
                }
            }
            return { results };
        },
    });
    ```

!!! tip "Important"
    In Python, use `asyncio.sleep(0)` for pauses. In TypeScript, use
    `setImmediate` or `setTimeout(resolve, 0)` to avoid blocking execution
    of other functions.

### Example of thread pools for intensive operations

When performing processing-intensive functions, consider creating thread pools
for better management of available computing resources, as shown in the
following example:

=== "Python"

    ```python
    async def cpu_intensive_tool(data: list) -> dict:
        loop = asyncio.get_event_loop()

        # Use thread pool for CPU-bound work
        with ThreadPoolExecutor() as executor:
            result = await loop.run_in_executor(
                executor,
                expensive_computation,
                data
            )
        return {"result": result}
    ```

=== "TypeScript"

    ```typescript
    import { FunctionTool } from '@google/adk';
    import { z } from 'zod';
    import { Worker } from 'worker_threads';

    const cpuIntensiveTool = new FunctionTool({
        name: 'cpu_intensive_tool',
        description: 'Perform CPU-intensive computation',
        parameters: z.object({
            data: z.array(z.unknown()).describe('Data to process'),
        }),
        execute: async ({ data }) => {
            // Use worker threads for CPU-bound work
            return new Promise((resolve, reject) => {
                const worker = new Worker('./expensive_computation_worker.js', {
                    workerData: data,
                });
                worker.on('message', resolve);
                worker.on('error', reject);
            });
        },
    });
    ```

### Example of process chunking

When performing processes on long lists or large amounts of data, consider
combining a thread pool technique with dividing up processing into chunks of
data, and yielding processing time between the chunks, as shown in the following
example:

=== "Python"

    ```python
    async def process_large_dataset(dataset: list) -> dict:
        results = []
        chunk_size = 1000

        for i in range(0, len(dataset), chunk_size):
            chunk = dataset[i:i + chunk_size]

            # Process chunk in thread pool
            loop = asyncio.get_event_loop()
            with ThreadPoolExecutor() as executor:
                chunk_result = await loop.run_in_executor(
                    executor, process_chunk, chunk
                )

            results.extend(chunk_result)

            # Yield control between chunks
            await asyncio.sleep(0)

        return {"total_processed": len(results), "results": results}
    ```

=== "TypeScript"

    ```typescript
    import { FunctionTool } from '@google/adk';
    import { z } from 'zod';

    const processLargeDatasetTool = new FunctionTool({
        name: 'process_large_dataset',
        description: 'Process a large dataset in chunks',
        parameters: z.object({
            dataset: z.array(z.unknown()).describe('Dataset to process'),
        }),
        execute: async ({ dataset }) => {
            const results: unknown[] = [];
            const chunkSize = 1000;

            for (let i = 0; i < dataset.length; i += chunkSize) {
                const chunk = dataset.slice(i, i + chunkSize);

                // Process chunk
                const chunkResult = await processChunk(chunk);
                results.push(...chunkResult);

                // Yield control between chunks
                await new Promise(resolve => setImmediate(resolve));
            }

            return { total_processed: results.length, results };
        },
    });
    ```

## Write parallel-ready prompts and tool descriptions

When building prompts for AI models, consider explicitly specifying or hinting
that function calls be made in parallel. The following example of an AI prompt
directs the model to use tools in parallel:

```none
When users ask for multiple pieces of information, always call functions in
parallel.

  Examples:
  - "Get weather for London and currency rate USD to EUR" → Call both functions
    simultaneously
  - "Compare cities A and B" → Call get_weather, get_population, get_distance in 
    parallel
  - "Analyze multiple stocks" → Call get_stock_price for each stock in parallel

  Always prefer multiple specific function calls over single complex calls.
```

The following example shows a tool function description that hints at more
efficient use through parallel execution:

=== "Python"

    ```python
    async def get_weather(city: str) -> dict:
        """Get current weather for a single city.

        This function is optimized for parallel execution - call multiple times for different cities.

        Args:
            city: Name of the city, for example: 'London', 'New York'

        Returns:
            Weather data including temperature, conditions, humidity
        """
        await asyncio.sleep(2)  # Simulate API call
        return {"city": city, "temp": 72, "condition": "sunny"}
    ```

=== "TypeScript"

    ```typescript
    import { FunctionTool } from '@google/adk';
    import { z } from 'zod';

    // Get current weather for a single city.
    // This function is optimized for parallel execution - call multiple times for different cities.
    const getWeatherTool = new FunctionTool({
        name: 'get_weather',
        description: `Get current weather for a single city.
            Optimized for parallel execution - call multiple times for different cities.`,
        parameters: z.object({
            city: z.string().describe('Name of the city, for example: "London", "New York"'),
        }),
        execute: async ({ city }) => {
            await new Promise(resolve => setTimeout(resolve, 2000)); // Simulate API call
            return { city, temp: 72, condition: 'sunny' };
        },
    });
    ```

## Next steps

For more information on building Tools for agents and function calling, see
[Function Tools](/adk-docs/tools-custom/function-tools/). For
more detailed examples of tools that take advantage of parallel processing, see
the samples in the
[adk-python](https://github.com/google/adk-python/tree/main/contributing/samples/parallel_functions)
repository or the TypeScript implementation in the
[adk-js](https://github.com/google/adk-js/tree/main/core/src/agents/functions.ts)
repository.
