# AWS SDK for Android: S3 Transfer Utility Tutorial

The Transfer Utility makes transferring data in and out of S3 quick, reliable, and easy. This tutorial will walk you through the Transfer Utility sample step-by-step to explain and demonstrate how to use the APIs. The Transfer Utility sample app allows users to select pictures and files from their phone, and transfer them into and out of S3 with the ability to pause, resume, cancel and delete transfers. 


        file);
observers.add(observer);
HashMap<String, Object> map = new HashMap<String, Object>();
Util.fillMap(map, observer, false);
transferRecordMaps.add(map);
```
 * Fills in the map with information in the observer so that it can be used
 * with a SimpleAdapter to populate the UI
 */
public static void fillMap(Map<String, Object> map, TransferObserver observer, boolean isChecked) {
    int progress = (int) ((double) observer.getBytesTransferred() * 100 / observer
            .getBytesTotal());
    map.put("id", observer.getId());
    map.put("checked", isChecked);
    map.put("fileName", observer.getAbsoluteFilePath());
    map.put("progress", progress);
    map.put("bytes",
            getBytesString(observer.getBytesTransferred()) + "/"
                    + getBytesString(observer.getBytesTotal()));
    map.put("state", observer.getState());
    map.put("percentage", progress + "%");
}
```

private static AmazonS3Client sS3Client;
private static CognitoCachingCredentialsProvider sCredProvider;
private static TransferUtility sTransferUtility;
```

 * Gets an instance of CognitoCachingCredentialsProvider which is
 * constructed using the given Context.
 *
 * @param context An Context instance.
 * @return A default credential provider.
 */
private static CognitoCachingCredentialsProvider getCredProvider(Context context) {
    if (sCredProvider == null) {
        sCredProvider = new CognitoCachingCredentialsProvider(
                context.getApplicationContext(),
                Constants.COGNITO_POOL_ID,
                Regions.US_EAST_1);
    }
    return sCredProvider;
}
```java
```
/**
 * Gets an instance of a S3 client which is constructed using the given
 * Context.
 *
 * @param context An Context instance.
 * @return A default S3 client.
 */
public static AmazonS3Client getS3Client(Context context) {
    if (sS3Client == null) {
        sS3Client = new AmazonS3Client(getCredProvider(context.getApplicationContext()));
    }
    return sS3Client;
}
```

 * Gets an instance of the TransferUtility which is constructed using the
 * given Context
 * 
 * @param context
 * @return a TransferUtility instance
 */
public static TransferUtility getTransferUtility(Context context) {
    if (sTransferUtility == null) {
        sTransferUtility = new TransferUtility(getS3Client(context.getApplicationContext()),
                context.getApplicationContext());
    }

    return sTransferUtility;
}
```

protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_upload);

    // Initializes TransferUtility, always do this before using it.
    transferUtility = Util.getTransferUtility(this);

    // Get the data from any transfer's that have already happened,
    initData();
    initUI();
}
``` 
observers = transferUtility.getTransfersWithType(TransferType.UPLOAD);
```
// transferRecordMaps which will display
// as a single row in the UI
HashMap<String, Object> map = new HashMap<String, Object>();
Util.fillMap(map, observer, false);
transferRecordMaps.add(map);
```

// non-terminal state
if (!TransferState.COMPLETED.equals(observer.getState())
        && !TransferState.FAILED.equals(observer.getState())
        && !TransferState.CANCELED.equals(observer.getState())) {

    observer.setTransferListener(new UploadListener());
}
```
	
```java
/*
 * A TransferListener class that can listen to a upload task and be notified
 * when the status changes.
 */
private class UploadListener implements TransferListener {

    // Simply updates the UI list when notified.
    @Override
    public void onError(int id, Exception e) {
        Log.e(TAG, "Error during upload: " + id, e);
        updateList();
    }

    @Override
    public void onProgressChanged(int id, long bytesCurrent, long bytesTotal) {
        updateList();
    }

    @Override
    public void onStateChanged(int id, TransferState newState) {
        updateList();
    }
}

 * Begins to upload the file specified by the file path.
 */
private void beginUpload(String filePath) {
    if (filePath == null) {
        Toast.makeText(this, "Could not find the filepath of the selected file",
                Toast.LENGTH_LONG).show();
        return;
    }
    File file = new File(filePath);
    TransferObserver observer = transferUtility.upload(Constants.BUCKET_NAME, file.getName(),
            file);
    observers.add(observer);
    HashMap<String, Object> map = new HashMap<String, Object>();
    Util.fillMap(map, observer, false);
    transferRecordMaps.add(map);
    observer.setTransferListener(new UploadListener());
    simpleAdapter.notifyDataSetChanged();
}
```
    @Override
    public void onClick(View v) {
        // Make sure the user has selected a transfer
        if (checkedIndex >= 0 && checkedIndex < observers.size()) {
            Boolean paused = transferUtility.pause(observers.get(checkedIndex).getId());
            /**
             * If paused does not return true, it is likely because the
             * user is trying to pause an upload that is not in a
             * pausable state (For instance it is already paused, or
             * canceled).
             */
            if (!paused) {
                Toast.makeText(
                        UploadActivity.this,
                        "Cannot pause transfer.  You can only pause transfers in a IN_PROGRESS or WAITING state.",
                        Toast.LENGTH_SHORT).show();
            }
        }
    }
});
```
    @Override
    public void onClick(View v) {
        transferUtility.pauseAllWithType(TransferType.UPLOAD);
    }
});
```
 * This async task queries S3 for all files in the given bucket so that they
 * can be displayed on the screen
 */
private class GetFileListTask extends AsyncTask<Void, Void, Void> {
    // The list of objects we find in the S3 bucket
    private List<S3ObjectSummary> s3ObjList;
    // A dialog to let the user know we are retrieving the files
    private ProgressDialog dialog;

    @Override
    protected void onPreExecute() {
        dialog = ProgressDialog.show(DownloadSelectionActivity.this,
                getString(R.string.refreshing),
                getString(R.string.please_wait));
    }

    @Override
    protected Void doInBackground(Void... inputs) {
        // Queries files in the bucket from S3.
        s3ObjList = s3.listObjects(Constants.BUCKET_NAME).getObjectSummaries();
        transferRecordMaps.clear();
        for (S3ObjectSummary summary : s3ObjList) {
            HashMap<String, Object> map = new HashMap<String, Object>();
            map.put("key", summary.getKey());
            transferRecordMaps.add(map);
        }
        return null;
    }

    @Override
    protected void onPostExecute(Void result) {
        dialog.dismiss();
        simpleAdapter.notifyDataSetChanged();
    }
}
```
 * Begins to download the file specified by the key in the bucket.
 */
private void beginDownload(String key) {
    // Location to download files from S3 to. You can choose any accessible
    // file.
    File file = new File(Environment.getExternalStorageDirectory().toString() + "/" + key);

    // Initiate the download
    TransferObserver observer = transferUtility.download(Constants.BUCKET_NAME, key, file);

    // Add the new download to our list of TransferObservers
    observers.add(observer);
    HashMap<String, Object> map = new HashMap<String, Object>();
    // Fill the map with the observers data
    Util.fillMap(map, observer, false);
    // Add the filled map to our list of maps which the simple adapter uses
    transferRecordMaps.add(map);
    observer.setTransferListener(new DownloadListener());
    simpleAdapter.notifyDataSetChanged();
}
```
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />


<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

    android:name="com.amazonaws.mobileconnectors.s3.transferutility.TransferService"
    android:enabled="true" />
```