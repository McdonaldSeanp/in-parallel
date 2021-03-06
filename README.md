# in-parallel
A lightweight Ruby library with very simple syntax, making use of Process.fork to execute code in parallel.

Other popular Ruby libraries that do parallel execution support one primary use case - crunching through a large queue of small tasks as quickly and efficiently as possible.  This library primarily supports the use case of executing a few larger tasks in parallel and managing the stdout and return values to make it easy to understand which processes are logging what, and what the outcome of the execution was. This library was created to be used by Puppet's Beaker test framework to enable parallel execution of some of the framework's tasks, and allow users to execute code in parallel within their tests.  This solution does not check to see how many processors you have, it just forks as many processes as you ask for.  That means that it will handle a handful of parallel processes well, but could definitely overload your system with ruby processes if you try to spin up a LOT of processes.  If you're looking for something intuitive and simple, but with a slight memory and processing overhead, and are on either Linux or Mac (forking processes is not supported on Windows), then this solution is  what you want.

If you are looking for something that excels at executing a large queue of tasks in parallel as efficiently as possible, you should take a look at the [parallel](https://github.com/grosser/parallel) project.

## Methods:

### run_in_parallel(timeout=nil, kill_all_on_error = false, &block)
1. Each method in a block will be executed in parallel (unless the method is defined in Kernel or BaseObject).
    1. Any methods further down the stack won't be affected, only the ones directly within the block.
2. You can assign return values to instance variables and it 'just works'.
3. Log STDOUT and STDERR chunked per process to the console so that it is easy to see what happened in which process.
4. Waits for each process in realtime and logs immediately upon completion of each process
5. If an exception is raised by a child process, it will optionally (kill_all_on_error) be re-raised in the primary process and kill all other still running child processes. The default will wait for all processes to complete execution before re-raising any unhandled exception from the child processes.
6. Times out by default at 30 minutes. Timeout default can be changed with InParallel::InParallelExecutor.parallel_default_timeout=X, or you can set the timeout param when calling the method

```ruby
  def method_with_param(name)
    ret_val = "hello #{name} \n"
    puts ret_val
    ret_val
  end
  
  def method_without_param
    # A result more complex than a string will be marshalled and unmarshalled and work
    ret_val = {:foo => "bar"}
    puts ret_val
    return ret_val
  end

  # Example:
  # will spawn 2 processes, (1 for each method) wait until they both complete, log chunked STDOUT/STDERR for
  # each process and assign the method return values to instance variables:
  run_in_parallel do
    @result_1 = method_with_param('world')
    @result_2 = method_without_param
  end
  
  puts "#{@result_1}, #{@result_2[:foo]}"
```
  
STDOUT would be:
```
Forked process for 'method_with_param' - PID = '49398'
Forked process for 'method_without_param' - PID = '49399'

------ Begin output for method_with_param - 49398
hello world
------ Completed output for method_with_param - 49398

------ Begin output for method_without_param - 49399
{:foo=>"bar"}
------ Completed output for method_without_param - 49399
hello world, bar
```

### Enumerable.each_in_parallel(identifier=nil, timeout=(InParallel::InParallelExecutor.timeout), kill_all_on_error = false, &block)
1. This is very similar to other solutions, except that it directly extends the Enumerable class with an each_in_parallel method, giving you the ability to pretty simply spawn a process for any item in an array or map.
2. Identifies the block location (or caller location if the block does not have a source_location) in the console log to make it clear which block is being executed
3. identifier param is only for logging, otherwise it will use the block source location.
4. If an exception is raised by a child process, it will optionally (kill_all_on_error) be re-raised in the primary process and kill all other still running child processes. The default will wait for all processes to complete execution before re-raising any unhandled exception from the child processes.
5. Times out by default at 30 minutes. Timeout default can be changed with InParallel::InParallelExecutor.parallel_default_timeout=X, or you can set the timeout param when calling the method

```ruby
  ["foo", "bar", "baz"].each_in_parallel { |item| puts item }

```
STDOUT:
```
'each_in_parallel' spawned process for '/Users/samwoods/parallel_test/test.rb:77:in `block (2 levels) in <top (required)>'' - PID = '51600'
'each_in_parallel' spawned process for '/Users/samwoods/parallel_test/test.rb:77:in `block (2 levels) in <top (required)>'' - PID = '51601'
'each_in_parallel' spawned process for '/Users/samwoods/parallel_test/test.rb:77:in `block (2 levels) in <top (required)>'' - PID = '51602'

------ Begin output for /Users/samwoods/parallel_test/test.rb:77:in `block (2 levels) in <top (required)>' - 51600
foo
------ Completed output for /Users/samwoods/parallel_test/test.rb:77:in `block (2 levels) in <top (required)>' - 51600

------ Begin output for /Users/samwoods/parallel_test/test.rb:77:in `block (2 levels) in <top (required)>' - 51601
bar
------ Completed output for /Users/samwoods/parallel_test/test.rb:77:in `block (2 levels) in <top (required)>' - 51601

------ Begin output for /Users/samwoods/parallel_test/test.rb:77:in `block (2 levels) in <top (required)>' - 51602
baz
------ Completed output for /Users/samwoods/parallel_test/test.rb:77:in `block (2 levels) in <top (required)>' - 51602
```

### run_in_background(ignore_results = true, &block)
1. This does basically the same thing as run_in_parallel, except it does not wait for execution of all processes to complete, it returns immediately.
2. You can optionally ignore results completely (default) or delay evaluating the results until later
3. You can run multiple blocks in the background and then at some later point evaluate all of the results

```ruby
  TMP_FILE = '/tmp/test_file.txt'
  
  def create_file_with_delay(file_path)
    sleep 2
    File.open(file_path, 'w') { |f| f.write('contents') }
    return true
  end
  
  # Example 1 - ignore results
  run_in_background { create_file_with_delay(TMP_FILE) }
  
  # Should not exist immediately upon block completion
  puts(File.exists?(TMP_FILE)) # false
  sleep(3)
  # Should exist once the delay from create_file_with_delay is done
  puts(File.exists?(TMP_FILE)) # true
  
  # Example 2 - delay results
  run_in_background(false) { @result = create_file_with_delay(TMP_FILE) }
  
  # Do something else
  
  run_in_background(false) { @result2 = create_file_with_delay('/tmp/someotherfile.txt') }
  
  # @result has not been assigned yet
  puts @result >> "unresolved_parallel_result_0"
  
  # This assigns all instance variables within the block and writes STDOUT and STDERR from the process to console.
  wait_for_processes
  puts @result # true
  puts @result2 # true
  
```

### wait_for_processes(timeout=nil, kill_all_on_error = false)
1. Used only after run_in_background with ignore_results=false
2. Optional args for timeout and kill_all_on_error
3. See run_in_background for examples
