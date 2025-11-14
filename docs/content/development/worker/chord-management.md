import { CodeBlock } from '@/components/CodeBlock'

# Chord Management and Monitoring

This document provides comprehensive information about managing Celery chords in the Rhesis worker system, including monitoring, troubleshooting, and best practices.

## What are Chords?

A **chord** in Celery is a pattern that allows you to execute a group of tasks in parallel and then run a callback function once all tasks in the group have completed. This is particularly useful for scenarios like:

- Running multiple test executions in parallel and then collecting the results
- Processing multiple files concurrently and then aggregating the output
- Performing parallel computati<CodeBlock filename="example.python" language="python">
{`from celery import chord

# Create a group of parallel tasks
tasks = [
    execute_single_test.s(test_id_1, ...),
    execute_single_test.s(test_id_2, ...),
    execute_single_test.s(test_id_3, ...),
]

# Define a callback to run after all tasks complete
callback = collect_results.s(start_time, config_id, ...)

# Execute the chord
job = chord(tasks)(callback)`}
</CodeBlock># Execute the chord
job = chord(tasks)(callback)
<CodeBlock filename="example.text" language="text">
{`
## How Rhesis Uses Chords

In the Rhesis system, chords are primarily used in test execution:

1. **Parallel Test Execution**: Individual tests are executed in parallel using `execute_single_test` tasks
2. **Result Collection**: Once all tests complete, `collect_results` is called to aggregate the results
3. **Status Updates**: The test run status is updated based on the aggreg<CodeBlock filename="example.python" language="python">
{`# Create tasks for parallel execution
tasks = []
for test in tests:
    task = execute_single_test.s(
        test_config_id=str(test_config.id),
        test_run_id=str(test_run.id),
        test_id=str(test.id),
        # ... other parameters
    )
    tasks.append(task)

# Collect results callback
callback = collect_results.s(
    datetime.utcnow().isoformat(),
    str(test_config.id),
    str(test_run.id),
    # ... other parameters
)

# Execute chord
job = chord(tasks)(callback)`}
</CodeBlock>
)

# Execute chord
job = chord(tasks)(callback)`}
</CodeBlock>

## Common Chord Issues

### 1. `chord_unlock` MaxRetriesExceededError

**Symptoms:**
<CodeBlock filename="example.text" language="text">
{`MaxRetriesExceededError: Can't retry celery.chord_unlock[task-id] args:(...) kwargs:{...}`}
</CodeBlock>

**Causes:**
- Individual tasks returning `None` instead of proper results
- Tasks failing without proper error handling
- Network interruptions during chord execution
- Worker processes being terminated unexpectedly

**Solutions:**
- Ensure all tasks return valid results (even on failure)
- Configure maximum retries for `chord_unlock` tasks
- Implement proper error handling in callback functions
- Use monitoring tools to detect and resolve stuck chords

### 2. Chord Never Completing

**Symptoms:**
- Callback function (`collect_results`) never executes
- Test runs remain in "IN_PROGRESS" status indefinitely
- Tasks appear to complete but no final status update

**Causes:**
- One or more subtasks in the chord failed silently
- Result backend issues preventing result storage
- Incorrect chord setup or callback configuration

## Chord Monitoring Tools

### Built-in Monitoring Script

The system includes a comprehensive monitoring script at `src/rhesis/backend/tasks/execution/chord_monitor.py` that provi<CodeBlock filename="Terminal" language="bash">
{`# Check for stuck chords running longer than 2 hours
python -m rhesis.backend.tasks.execution.chord_monitor check --max-hours 2

# Check with JSON output
python -m rhesis.backend.tasks.execution.chord_monitor check --max-hours 1 --json`}
</CodeBlock>ecution.chord_monitor check --m<CodeBlock filename="Terminal" language="bash">
{`# Display summary of active chord_unlock tasks
python -m rhesis.backend.tasks.execution.chord_monitor status`}
</CodeBlock>sis.backend.tasks.execution.cho<CodeBlock filename="Terminal" language="bash">
{`# Dry run - show what would be revoked
python -m rhesis.backend.tasks.execution.chord_monitor revoke --max-hours 1 --dry-run

# Actually revoke stuck chords
python -m rhesis.backend.tasks.execution.chord_monitor revoke --max-hours 1`}
</CodeBlock>sks.execution.chord_monitor revoke<CodeBlock filename="Terminal" language="bash">
{`# Get detailed information about a specific chord
python -m rhesis.backend.tasks.execution.chord_monitor inspect <chord-id> --verbose`}
</CodeBlock>ution.chord_monitor inspect <chord-id> <CodeBlock filename="Terminal" language="bash">
{`# Purge all tasks from all queues (use with extreme caution)
python -m rhesis.backend.tasks.execution.chord_monitor clean --force`}
</CodeBlock>kend.tasks.execution.chord_monitor clean --force
<CodeBlock filename="example.text" language="text">
{`
### Quick Fix Script

A simplified<CodeBlock filename="Terminal" language="bash">
{`# Check and interactively fix chord issues
python fix_chords.py`}
</CodeBlock>eractively fix chord issues
python fix_chords.py`}
</CodeBlock>

This script:
- Shows current chord status
- Detects stuck chords (>30 minutes)
- Offers to revoke stuck chords interactively
- Auto-revokes very stuck chords (>2 hours)
- Provides recommendations for next steps

## Monitoring Best Practices

### 1. Regular Monitoring

Set u<CodeBlock filename="Terminal" language="bash">
{`# Add to crontab to run every 15 minutes
*/15 * * * * cd /path/to/backend && python fix_chords.py`}
</CodeBlock> * * cd /path/to/backend && python fix_chords.py
<CodeBlock filename="example.python" language="python">
{`
### 2. Automated Clean<CodeBlock filename="example.python" language="python">
{`from rhesis.backend.tasks.execution.chord_monitor import setup_periodic_monitoring

# This can be called from a scheduled task
result = setup_periodic_monitoring()`}
</CodeBlock>eduled task
result = setup_periodic_monitoring()`}
</CodeBlock>

### 3. Logging and Ale<CodeBlock filename="Terminal" language="bash">
{`# Monitor for chord_unlock errors
tail -f celery_worker.log | grep "chord_unlock"

# Monitor for MaxRetriesExceededError
tail -f celery_worker.log | grep "MaxRetriesExceededError"`}
</CodeBlock>lery_worker.log | grep "MaxRetriesExceededError"
<CodeBlock filename="example.python" language="python">
{`
### 4. Health Checks

In<CodeBlock filename="example.python" language="python">
{`from rhesis.backend.tasks.execution.chord_monitor import get_active_chord_unlocks

def health_check():
    active_chords = get_active_chord_unlocks()
    stuck_chords = check_stuck_chords(max_runtime_hours=1)

    return {
        "active_chord_unlocks": len(active_chords),
        "stuck_chords": len(stuck_chords),
        "status": "unhealthy" if stuck_chords else "healthy"
    }`}
</CodeBlock>"unhealthy" if stuck_chords else "healthy"
    }`}
</CodeBlock>

## Configuration for Chord Stability

### Worker Configuration

<CodeBlock filename="example.python" language="python">
{`app.conf.update(
    # Chord configuration - prevent infinite retry loops
    chord_unlock_max_retries=3,
    chord_unlock_retry_delay=1.0,

    # Result backend configuration
    result_persistent=True,
    result_expires=3600,  # Results expire after 1 hour

    # Task tracking for better chord monitoring
    task_track_started=True,
    task_send_sent_event=True,
    worker_send_task_events=True,

    # Task-specific configurations
    task_annotations={
        'celery.chord_unlock': {
            'max_retries': 3,
            'retry_backoff': True,
            'retry_backoff_max': 60,
            'retry_jitter': True,
            'soft_time_limit': 300,  # 5 minutes soft limit
            'time_limit': 600,       # 10 minutes hard limit
        },
        'rhesis.backend.tasks.execution.results.collect_results': {
            'max_retries': 3,
            'retry_backoff': True,
            'retry_backoff_max': 60,
            'soft_time_limit': 600,  # 10 minutes soft limit
            'time_limit': 900,       # 15 minutes hard limit
        }
    }
)`}
</CodeBlock>       # 15 minutes hard limit
        }
    }
)
<CodeBlock filename="example.python" language="python">
{`
### Task Implementatio<CodeBlock filename="example.python" language="python">
{`@app.task(bind=True)
def execute_single_test(self, ...):
    try:
        result = perform_test_execution(...)

        # Ensure we always return a valid result
        if result is None:
            result = {
                "test_id": test_id,
                "status": "failed",
                "error": "Test execution returned None",
                "execution_time": 0
            }

        return result
    except Exception as e:
        # Return failure result instead of raising exception
        failure_result = {
            "test_id": test_id,
            "status": "failed",
            "error": str(e),
            "execution_time": 0
        }

        if self.request.retries < self.max_retries:
            try:
                self.retry(exc=e, kwargs=original_kwargs)
            except self.MaxRetriesExceededError:
                return failure_result

        return failure_result`}
</CodeBlock>rn failure_result

        return failure_resu<CodeBlock filename="example.python" language="python">
{`@app.task(bind=True)
def collect_results(self, results, ...):
    # Handle different result formats from chord execution
    processed_results = []
    if results:
        for result in results:
            if result is None:
                processed_results.append(None)
            elif isinstance(result, list) and len(result) == 2:
                # Handle [[task_id, result], error] format
                task_result = result[1] if result[1] is not None else None
                processed_results.append(task_result)
            else:
                processed_results.append(result)

    # Process results and handle failures gracefully
    failed_tasks = sum(1 for result in processed_results
                      if result is None or
                      (isinstance(result, dict) and result.get("status") == "failed"))`}
</CodeBlock>lt, dict) and result.get("status") == "failed"))`}
</CodeBlock>

## Troubleshooting Workflows

### When You Enc<CodeBlock filename="Terminal" language="bash">
{`   python fix_chords.py`}
</CodeBlock>sessment**
   ```bash
   python<CodeBlock filename="Terminal" language="bash">
{`   python -m rhesis.backend.tasks.execution.chord_monitor status`}
</CodeBlock>.backend.tasks.execution.chord_mon<CodeBlock filename="Terminal" language="bash">
{`   python -m rhesis.backend.tasks.execution.chord_monitor check --max-hours 1`}
</CodeBlock>s.execution.chord_monito<CodeBlock filename="Terminal" language="bash">
{`   tail -50 celery_worker.log | grep -E "(chord_unlock|MaxRetries|ERROR)"`}
</CodeBlock>g | grep -E "(chord_unlock|MaxRetr<CodeBlock filename="Terminal" language="bash">
{`   # Revoke stuck chords
   python -m rhesis.backend.tasks.execution.chord_monitor revoke --max-hours 0.5

   # Restart workers to pick up new configuration
   pkill -f celery
   celery -A rhesis.backend.worker.app worker --loglevel=INFO &`}
</CodeBlock>s.backend.worker.app worker --loglevel=INFO &
   <CodeBlock filename="example.text" language="text">
{`
### Emergency Recovery

If the system is completely stuck with m<CodeBlock filename="Terminal" language="bash">
{`   pkill -f celery`}
</CodeBlock>p All Workers**`}
</CodeBlock>bash
   pkill -f celery
 <CodeBlock filename="Terminal" language="bash">
{`   python -m rhesis.backend.tasks.execution.chord_monitor clean --force`}
</CodeBlock>d.tasks.execution.chord_moni<CodeBlock filename="Terminal" language="bash">
{`   celery -A rhesis.backend.worker.app worker --loglevel=INFO &`}
</CodeBlock>s.backend.worker.app worker -<CodeBlock filename="Terminal" language="bash">
{`   python fix_chords.py`}
</CodeBlock>Recovery**
   ```bash
   python fix_chords.py
   ```

## Monitoring Script Reference

### Command Line Options

| Command | Description | Example |
|---------|-------------|---------|
| `status` | Show current chord status | `python -m ...chord_monitor status` |
| `check` | Check for stuck chords | `python -m ...chord_monitor check --max-hours 2` |
| `revoke` | Revoke stuck chords | `python -m ...chord_monitor revoke --max-hours 1` |
| `inspect` | Inspect specific chord | `python -m ...chord_monitor inspect <chord-id>` |
| `clean` | Purge all tasks | `python -m ...chord_monitor clean --force` |

### Common Options

- `--max-hours N`: Consider chords stuck after N hours
- `--dry-run`: Show what would be done without executing
- `--json`: Output results in JSON format
- `--verbose`: Show detailed information
- `--force`: Required for destructive operations

### Return Codes

- `0`: Success, no issues found
- `1`: Issues found or errors occurred
- `130`: Operation cancelled by user

## Prevention Strategies

1. **Proper Task Design**: Always return valid results, handle exceptions gracefully
2. **Configuration**: Set appropriate timeouts and retry limits
3. **Monitoring**: Regular checks for stuck chords
4. **Testing**: Test chord behavior in development with various failure scenarios
5. **Logging**: Comprehensive logging to diagnose issues quickly
6. **Documentation**: Keep this documentation updated with new patterns and solutions

## Related Files

- `src/rhesis/backend/worker.py` - Celery configuration
- `src/rhesis/backend/tasks/execution/orchestration.py` - Chord implementation
- `src/rhesis/backend/tasks/execution/test.py` - Individual task implementation
- `src/rhesis/backend/tasks/execution/results.py` - Chord callback implementation
- `src/rhesis/backend/tasks/execution/chord_monitor.py` - Monitoring utilities
- `fix_chords.py` - Quick monitoring script
