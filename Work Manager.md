# WORK MANAGER

* An architectural component in the new android jetpack library.
  * Jetpack is a set of libraries which makes it easy to implement different components. It provides common infrastructure code so you can forget about API level and focus on features of your app.
  * Work manager is an architectural component of this library which let you create a task and hand it off to work manager instance to run immediately or at an appropriate time or with or without any constraints as the need may be.
  * Basically, Work manager is a library for managing deferrable background work.

## WHY WORK MANAGER

* Job Scheduler (API 23 and above)
* Firebase Job Dispatcher (Backward Compatible up to API level 14 but needs Google Play services)
* Alarm manager & Broadcast Receivers (Below API 14)

Therefore, to avoid writing code for different API’s, we can use the work manager.
* No need to write device logic.
* Backward compatible with or without google play services.
* Can check the status of work.
* Can be canceled.
* Can be chained.

## HOW IT WORKS

First of all, it checks if the API level of the device is above 23. If so then it hands off the task to Job Scheduler and it executes the work. If not then it again checks if the API level of the device is greater than or equal to 14 and Google play services is installed and firebase job dispatcher library is also installed. If it meets all the 3 requirements then it handover the task to Firebase Job Dispatcher and work is executed otherwise it hands off the task to plain old Alarm manager and Broadcast Receiver.

## ADDING WORK MANAGER LIBRARY
Add the following dependencies in your `build.gradle` file
```java
implementation “android.arch.work:work-runtime:$work_version” // Use -ktx for Kotlin support
implementation “android.arch.work:work-firebase:$work-version” // Firebase job dispatcher support
implementation “android.arch.work:work-testing:$work-version” // Testing Support
// Latest work-version is 1.0.0-alpha11
```

Three step process-
1. Extend the worker class where you write all the logic of your task.
2. We need to create a work request and if you need, you add few constraints.
3. Get an instance of work manager and enqueue the work.

For step 2, there can be 2 types of work request (discussed later):-
* One Time Work Request
* Periodic Work Request

Here I’ll be using One Time Work Request.

#### STEP1:
```java
public class MyWorker extends Worker {
  @Override
  public Worker.Result doWork() {

    //Do the work here.
    doMyWork();

    return Result.SUCCESS;
    //RETRY tells WorkMAnager to try this task again later.
    //FAILURE says not to try again until &unless app schedules to do it again
  }
}
```
### Enqueuing the request
#### STEP2: 
```java
OneTimeWorkRequest myWork = new OneTimeWorkRequest.Builder(MyWorker.class).build();
```
#### STEP3: 
```java	
WorkManager.getInstance().enqueue(myWork);
```
#### Optional STEP (Constraints):
```java
Constraints myConstraints = new Constraints.Builder()
                              .setRequiresDeviceIdle(true)
                              .setRequiresCharging(true)
                              .setRequiredNetworkType(NetworkType.CONNECTED)
                              .build();
OneTimeRequest compressionWork = new OneTimeWokRequest.Builder(MyWorker.class)
                                    .setInitialDelay(Duration.ofHours(2))
                                    .setConstraints(myConstraints)
                                    .addTag(“sometag”)
                                    .build();
```
### In case of PeriodicWorkRequest:
#### STEP2:
```java
PeriodicWorkRequest myWork = new PeriodicWorkRequest.builder(MyWorker.class, 20, TimeUnit.MINUTES).build();
// Here work will be repeated in every 20 minutes.
```
#### NOTE:- As per the library guidelines, the time interval should be greater than 9 lacs milliseconds which turn out to be 15 minutes.

### Observing your work

```java
LiveData<WorkInfo> workInfo = workManager.getWorkInfoByIdLiveData(myWork.id)	
```
If you want to keep track of your work then you can call `getWorkInfoByIdLiveData(work.id)`
This returns a `LiveData<WorkInfo>`. `WorkInfo` is a type that determines the state of your work and `LiveData` here is the lifecycle of our observable by which you can observe the state transitions of the work request.
```java	
workInfo.state == ENQUEUED
workInfo.state == RUNNING
workInfo.state == SUCCEEDED
```
### Observing work by tag

```java
LiveData<List<WorkInfo>>  workInfos =  workManager.getWorkInfosByTagLiveData(“sometag”);
```
Note that, we can associate the same tag to multiple work requests. That’s why `getWorkInfosByTagLiveData()` returns a list of workInfo with the same tag.

### Chaining
One of the coolest features of Work Manager is it can chain work.
```java
workManager.beginWith(a,b,c).then(d,e).then(f,g,h).enqueue();
```
Here Work Manager will begin with work requests a, b, c. Work requests d, e are eligible to run only when a, b, c are succeeded. Thus f, g, h will run only when all the preceding work is done.
Note that, here outputs of parent work requests become inputs for the children in the chain. This helps you manage state and send states from parent work to descendent work. 

### Cancelling Work by Id or Tag
```java
workManager.cancelWorkById(id);

workManager.cancelAllWorkByTag(tag);
```

## THREADING IN WORK MANAGER

There’s a single threaded internal task executor. Work request gets enqueued in it which is stored in a local database. Every app which uses work manager has a local database where we keep out track of states, inputs, outputs, everything. Then OS tells you if all the constraints are met or not. In case all the constraints are met, `WorkerFactory` creates a `Worker` which executes a task on the executor.


Suppose, if we don’t want to execute the task on the default executor then we can use the class named `ListenableFuture`. (A future that can have one or more listeners that can be invoked on a specified executor). So using `ListenableFuture`, another class `ListenableWorker` is derived which overrides only one method called `startWork()`. It will start work on the main thread and will set a future listener. Now we can perform any work on any thread we want and when we are done we’ve to just set a result on the future and `FutureListener` would be able to listen for it.

#### This means that Worker is a simple ListenableWorker.
```java
abstract ListenableWorker.Result doWork()
@Override
ListenableFuture<ListenableWorker.Payload> startWork(){
  ListenableFuture future = createFuture();
  backgroundExecutor.execute{
    ListenableWorker.Result result = doWork();
    future.set(ListenableWorker.Payload(result, outputData))
  }
  return future;
}
```
Therefore,
* `Workers` run `synchronously` on a `pre-specified background thread`.
* `ListenableWorkers` run `asynchronously` on an `unspecified thread`.

Now, if we create a `ListenableWorker` we have to create a `ListenableFuture` which is an interface. So either we can add Guava dependencies or we can use `ResolvableFuture` which is a lightweight implementation provided in `androidx.concurrent.concurrent-features`.
```java
// Example: getting Location
class LocationWorker(Context context, WorkerParameters params){
  ResolvableFuture<Payload> future = ResolvableFuture.create();
  @Override
  public ListenableFuture<Payload> startWork(){
    if(hasPermissions()){
      getLocation(getFusedLocationProviderClient().lastLocation)
    } else {
      future.set(Payload(Result.FAILURE, Data.EMPTY))
    }
    return future;
  }
  void getLocation(Task<Location>? pendingTask){
    pendingTask?.addOnCompleteListener{ task ->
      if(task.isSuccessful) {
        future.set{Payload(Result.SUCCESS, workDataOf(“location” to task.result))}
      } else {
        future.setException(task.exception)
      }
    } 
}
```
## OPERATIONS

Whenever we want to make changes to the local database or we want to perform some action after the `enqueue` or `cancel` happened, then we use an interface called `Operation`.
```java
interface Operation{
  public LiveData<Operation.State> getState()
  public ListenableFuture<Operation.State.SUCCESS> getResult()
}
```
Here functions `enqueue` and  `cancelWorkById` return `Operation` datatype.
```java
public Operation enqueue(List<WorkRequest> requests)
public Operation cancelWorkById(UUID id) 
```

## WHEN IS WORK STOPPED

* The constraints are no longer met.
* The OS pre-empted your work for some reason.
* Your work was cancelled.

## HOW DO WE STOP WORK
```java
void ListenableWorker onStopped()
ListenableFuture.cancel();
```
