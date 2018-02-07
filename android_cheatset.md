Android Knowledge
=======================

## 1. onMeasure in CustomView

[StackOverflow ref](https://stackoverflow.com/a/12267248/2610716)

`onMeasure()` is your opportunity to tell Android how big you want your custom view to be dependent the layout constraints provided by the parent; it is also your custom view's opportunity to learn what those layout constraints are (in case you want to behave differently in a `match_parent` situation than a `wrap_content` situation).  These constraints are packaged up into the `MeasureSpec` values that are passed into the method.  Here is a rough correlation of the mode values:

 - **EXACTLY** means the `layout_width` or `layout_height` value was set to a specific value.  You should probably make your view this size.  This can also get triggered when `match_parent` is used, to set the size exactly to the parent view (this is layout dependent in the framework).
 - **AT_MOST** typically  means the `layout_width` or `layout_height` value was set to `match_parent` or `wrap_content` where a maximum size is needed (this is layout dependent in the framework), and the size of the parent dimension is the value.  You should not be any larger than this size.
 - **UNSPECIFIED** typically means the `layout_width` or `layout_height` value was set to `wrap_content` with no restrictions.  You can be whatever size you would like.  Some layouts also use this callback to figure out your desired size before determine what specs to actually pass you again in a second measure request.

The contract that exists with `onMeasure()` is that `setMeasuredDimension()` **MUST** be called at the end with the size you would like the view to be.  This method is called by all the framework implementations, including the default implementation found in `View`, which is why it is safe to call `super` instead if that fits your use case.

Granted, because the framework does apply a default implementation, it may not be necessary for you to override this method, but you may see clipping in cases where the view space is smaller than your content if you do not, and if you lay out your custom view with `wrap_content` in both directions, your view may not show up at all because the framework doesn't know how large it is!

Generally, if you are overriding `View` and not another existing widget, it is probably a good idea to provide an implementation, even if it is as simple as something like this:
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

    int desiredWidth = 100;
    int desiredHeight = 100;

    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    int heightSize = MeasureSpec.getSize(heightMeasureSpec);

    int width;
    int height;

    //Measure Width
    if (widthMode == MeasureSpec.EXACTLY) {
        //Must be this size
        width = widthSize;
    } else if (widthMode == MeasureSpec.AT_MOST) {
        //Can't be bigger than...
        width = Math.min(desiredWidth, widthSize);
    } else {
        //Be whatever you want
        width = desiredWidth;
    }

    //Measure Height
    if (heightMode == MeasureSpec.EXACTLY) {
        //Must be this size
        height = heightSize;
    } else if (heightMode == MeasureSpec.AT_MOST) {
        //Can't be bigger than...
        height = Math.min(desiredHeight, heightSize);
    } else {
        //Be whatever you want
        height = desiredHeight;
    }

    //MUST CALL THIS
    setMeasuredDimension(width, height);
}
```

## Activity restore from instanceState

There are two way to restore data from savedInstanceState
- override method `onCreate(Bundle savedInstanceState)` 
- implement callback `onRestoreInstanceState(Bundle savedInstanceState)`, this callback will be call after `onStart()`

## Common intent

#### Show a webpage

```java
Uri uri = Uri.parse("http://google.com");
Intent i = new Intent(Intent.ACTION_VIEW, uri);
startActivity(i);
```

#### Dial a phone number

```java
Uri uri = Uri.parse("tel:0974427143");
Intent i = new Intent(Intent.ACTION_DIAL, uri);
startActivity(i);
```

## Broadcast receiver

`sendBroadcast()` method to send event asynchronously. All receivers can receve event at the same time

`sendOrderedBroadcast()` method to send event synchronously. Deliver to one receiver at a time. Each receiver can propagate result to the next or abort the event. To control order of receiver, we can use `android:priority` of matching intent filter. Same priority can receive event at arbitrary order

## Force firebase refresh device token 

The Instance ID service initiates callbacks periodically (for example, every 6 months), requesting that your app refreshes its tokens. It may also initiate callbacks when:

- There are security issues; for example, SSL or platform issues.
- Device information is no longer valid; for example, backup and restore.
- The Instance ID service is otherwise affected.

```java
new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        // this command is blocking call, so it has to be call in background thread
                        FirebaseInstanceId.getInstance().deleteInstanceId();
                        Log.d(TAG, "firebase instance id deleted");

                        FirebaseInstanceId.getInstance().getToken();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
```


## Handle firebase notification

```java
class MessagingService : FirebaseMessagingService() {

    override fun onMessageReceived(remoteMessage: RemoteMessage?) {
        super.onMessageReceived(remoteMessage)

        // Not getting messages here? See why this may be: https://goo.gl/39bRNJ
        remoteMessage?.let {

            val myNoti = Notification()

            // Check if message contains a data payload.
            val data = it.data
            if (data.isNotEmpty()) {
                Log.d(TAG, "Message data payload: " + data)

                myNoti.id = data["id"]?.toInt() ?: 0
                myNoti.title = data["title"]
                myNoti.message = data["message"]
                myNoti.notifyType = data["notify_type"]
                myNoti.pageableType = data["pageable_type"]
                myNoti.pageableId = data["pageable_id"]?.toInt() ?: 0
                myNoti.detail = data["detail"]
                myNoti.read = data["read"]?.toBoolean() ?: false
                myNoti.createdAt = data["created_at"]
                myNoti.updatedAt = data["updated_at"]

                if (/* Check if data needs to be processed by long running job */ true) {
                    // For long-running tasks (10 seconds or more) use Firebase Job Dispatcher.
                    scheduleJob();
                } else {
                    // Handle message within 10 seconds
                    handleNow();
                }

            }

            // Check if message contains a notification payload.
            val notification = it.notification
            notification?.let {
                Log.d(TAG, "Message Notification: title=[" + it.title + "], body=[" + it.body + "] " + it.sound)

                myNoti.title = it.title
                myNoti.message = it.body
                myNoti.detail = it.body
            }

            // Also if you intend on generating your own notifications as a result of a received FCM
            // message, here is where that should be initiated. See sendNotification method below.
            sendNotification(myNoti)

        }

    }

    /**
     * Schedule a job using FirebaseJobDispatcher.
     */
    private void scheduleJob() {
        // [START dispatch_job]
        FirebaseJobDispatcher dispatcher = new FirebaseJobDispatcher(new GooglePlayDriver(this));
        Job myJob = dispatcher.newJobBuilder()
                .setService(MyJobService.class)
                .setTag("my-job-tag")
                .build();
        dispatcher.schedule(myJob);
        // [END dispatch_job]
    }

     /**
     * Handle time allotted to BroadcastReceivers.
     */
    private void handleNow() {
        Log.d(TAG, "Short lived task is done.");
    }

    /**
     * Create and show a simple notification containing the received FCM message.
     */
    private fun sendNotification(notification: Notification) {
        val intent = Intent(this, SplashActivity::class.java)
        intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP)
        intent.putExtra("id", notification.id)
        intent.putExtra("title", notification.title)
        intent.putExtra("message", notification.message)
        intent.putExtra("notify_type", notification.notifyType)
        intent.putExtra("pageable_type", notification.pageableType)
        intent.putExtra("pageable_id", notification.pageableId)
        intent.putExtra("detail", notification.detail)
        intent.putExtra("read", notification.read)
        intent.putExtra("created_at", notification.createdAt)
        intent.putExtra("updated_at", notification.updatedAt)
        val pendingIntent = PendingIntent.getActivity(this, 0 /* Request code */, intent,
                PendingIntent.FLAG_ONE_SHOT)

        //        String channelId = getString(R.string.default_notification_channel_id);
        val defaultSoundUri = RingtoneManager.getDefaultUri(RingtoneManager.TYPE_NOTIFICATION)
        val notificationBuilder = NotificationCompat.Builder(this)
                .setTicker(notification.detail)
                .setSmallIcon(R.mipmap.ic_launcher)
                .setContentTitle(notification.title)
                .setStyle(NotificationCompat.BigTextStyle().bigText(notification.detail))
                .setAutoCancel(true)
                .setLights(Color.BLUE, 500, 2000)
                .setVibrate(longArrayOf(100, 200, 300, 400, 500, 400, 300, 200, 400))
                .setSound(defaultSoundUri)
                .setContentIntent(pendingIntent)

        val notificationManager = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager

        notificationManager.notify(notification.id, notificationBuilder.build())
    }

    companion object {
        private val TAG = "FirebaseMsgService"
    }
}

public class MyJobService extends JobService {

    private static final String TAG = "MyJobService";

    @Override
    public boolean onStartJob(JobParameters jobParameters) {
        Log.d(TAG, "Performing long running task in scheduled job");
        // TODO(developer): add long running task here.
        return false;
    }

    @Override
    public boolean onStopJob(JobParameters jobParameters) {
        return false;
    }

}
```