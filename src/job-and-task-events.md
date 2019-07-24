# HPC Pack 2016 Job and Task Events in SignalR

You can recieve HPC Pack Job and Task events in a [SingalR](https://dotnet.microsoft.com/apps/aspnet/real-time) client. There're various clients for different programming languages. Here we take C# for an example of how to receive Job and Task events.

1. Install the C# client `Microsoft.AspNet.SignalR.Client`.
2. `using` `Microsoft.AspNet.SignalR.Client` and `Microsoft.AspNet.SignalR.Client.Transports` in your code.
3. Then you create an instance of `HubConnection` like:

   ```C#
   var hubConnection = new HubConnection(url)
   ```
   
   Here the `url` is "https://{your-cluster-name-or-ip}/hpc".
   
   You can optionally trace errors of the connection by
   
   ```C#
   var hubConnection = new HubConnection(url) { TraceLevel = TraceLevels.All, TraceWriter = Console.Error };
   ```
   
   And you can also add an error handler like: 
   
   ```C#
   hubConnection.Error += ex => Console.Error.WriteLine($"HubConnection Exception:\n{ex}");
   ```
   
   You must provide credential for accessing the events. HTTP Basic Auth is used here:
   
   ```C#
   hubConnection.Headers.Add("Authorization", BasicAuthHeader(username, password));
   ```

4. Then you create hub proxy objects from the hub connection and listen for events from them.
   
   For Job events, you create a `JobEventHub` and listen to the `JobStateChange` event:
   
   ```C#
   var jobEventHubProxy = hubConnection.CreateHubProxy("JobEventHub");
   jobEventHubProxy.On("JobStateChange", (int id, string state, string previousState) =>
   {
     Console.WriteLine($"Job: {id}, State: {state}, Previous State: {previousState}");
   });
   ```
   
   For Task events, you create a `TaskEventHub` and listen to the `TaskStateChange` event:
   
   ```C#
   var taskEventHubProxy = hubConnection.CreateHubProxy("TaskEventHub");
   taskEventHubProxy.On("TaskStateChange", (int id, int taskId, int instanceId, string state, string previousState) => 
   {
     Console.WriteLine($"Job: {id}, Task: {taskId}, Instance: {instanceId}, State: {state}, Previous State: {previousState}");
   });
   ```
   
5. Then you connect to the server by

   ```C#
   await hubConnection.Start(new WebSocketTransport())
   ```
   
   Here we explicitly specify the `WebSocketTransport`. SignalR supports various ways of transport. But we choose WebSocket. You may not get any events or fail to connect at all if you use transport other than WebSocket.

6. At last, you call `BeginListen` method on the hub proxy object to start listening. You must specify the job id which you want to listen to its job/task events.

   ```C#
   try
   {
     await jobEventHubProxy.Invoke("BeginListen", jobId);
     await taskEventHubProxy.Invoke("BeginListen", jobId);
   }
   catch (Exception ex)
   {
     Console.Error.WriteLine($"Exception on invoking server method:\n{ex}");
     return null;
   }
   ```
   
   The `catch` block will catch errors on server when invoking the `BeginListen` method. For other errors on hub connection, you may need to use an error handler like the one in previous step.
   
The whole code snippet is:

```C#
using Microsoft.AspNet.SignalR.Client;
using Microsoft.AspNet.SignalR.Client.Transports;

//...

string BasicAuthHeader(string username, string password)
{
    string credentials = $"{username}:{password}";
    byte[] bytes = Encoding.ASCII.GetBytes(credentials);
    string base64 = Convert.ToBase64String(bytes);
    return $"Basic {base64}";
}

async Task<HubConnection> ListenToJobAndTasks(string url, int jobId, string username, string password)
{
    var hubConnection = new HubConnection(url) { TraceLevel = TraceLevels.All, TraceWriter = Console.Error };
    hubConnection.Headers.Add("Authorization", BasicAuthHeader(username, password));
    hubConnection.Error += ex => Console.Error.WriteLine($"HubConnection Exception:\n{ex}");

    var jobEventHubProxy = hubConnection.CreateHubProxy("JobEventHub");
    jobEventHubProxy.On("JobStateChange", (int id, string state, string previousState) =>
    {
        Console.WriteLine($"Job: {id}, State: {state}, Previous State: {previousState}");
    });

    var taskEventHubProxy = hubConnection.CreateHubProxy("TaskEventHub");
    taskEventHubProxy.On("TaskStateChange", (int id, int taskId, int instanceId, string state, string previousState) =>
    {
        Console.WriteLine($"Job: {id}, Task: {taskId}, Instance: {instanceId}, State: {state}, Previous State: {previousState}");
    });

    Console.WriteLine($"Connecting to {url} ...");
    try
    {
        await hubConnection.Start(new WebSocketTransport());
    }
    catch (Exception ex)
    {
        Console.Error.WriteLine($"Exception on starting:\n{ex}");
        return null;
    }

    Console.WriteLine($"Begin to listen...");
    try
    {
        await jobEventHubProxy.Invoke("BeginListen", jobId);
        await taskEventHubProxy.Invoke("BeginListen", jobId);
    }
    catch (Exception ex)
    {
        Console.Error.WriteLine($"Exception on invoking server method:\n{ex}");
        return null;
    }

    return hubConnection;
}

```
