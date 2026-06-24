# Simple Storage Service (S3)

In this tutorial, you'll store the images sent to your agent (and their corresponding predicted version) in an S3 bucket.  


## Create a Bucket

1. Open the Amazon S3 console at https://console.aws.amazon.com/s3/.
2. In the left navigation pane, choose **Buckets**\.
3. Choose **Create bucket**.

   The **Create bucket** wizard opens.

4. In **Bucket name**, enter a name for your bucket, e.g. `john-polyai-images` (must be unique across **all of Amazon S3**).

5. In **Region**, choose the AWS Region where you want the bucket to reside.

   Choose the Region where you provisioned your EC2 instance.

6. Under **Object Ownership**, leave ACLs disabled. By default, ACLs are disabled\. A majority of modern use cases in Amazon S3 no longer require the use of ACLs\. We recommend that you keep ACLs disabled, except in unusual circumstances where you must control access for each object individually\.

8. Make sure the default encryption with `SSE-S3` encryption type is enabled.

9. Choose **Create bucket**.

## Integrate S3 into the Agent and Yolo services

Your current implementation of agent -> yolo communication passes the image content directly in the HTTP request from the Agent to the Yolo service.
Your goal is to decouple the two services by using S3 as an intermediary store:

1. The Agent uploads the original image to S3 and sends **only the S3 object key** to the Yolo service.
2. The Yolo service downloads the image from S3, runs prediction, uploads the predicted image to S3, and returns the json response.

### Implementation notes

- For the Agent service, in the `/chat` endpoint handler, after receiving an image from the user:

   - [Upload the image to your S3 bucket](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/s3-uploading-files.html#uploading-files).
   - Call the Yolo service passing **only the S3 object key** (not the image bytes) in the request body, e.g. as a JSON field `{"image_s3_key": "<key>"}`.


- For the Yolo service, in the `/predict` endpoint handler:

   - Read the `image_s3_key` field from the request body.
   - Download the image from S3 using that key.
   - Run the YOLO prediction as usual.
   - Upload the predicted image to S3



- Use the `boto3` Python official SDK to upload and download images from S3. Don't forget to add it to the `requirements.txt` file.
- Consider organising your objects in the bucket by some value unique to the chat, such as `chat_id`, for example: `<chat_id>/<prediction_id>/original/<image_name>` and `<chat_id>/<prediction_id>/predicted/<image_name>`.

- Don't hard-code the bucket name or AWS region in your `.py` files. Instead, read them from environment variables:
  - `AWS_REGION` - your region code (e.g. `us-east-1`).
  - `AWS_S3_BUCKET` - the name of your bucket.

- Test your new feature locally by sending a message with image to predict to your agent.
- Test your new feature in dev instance. 


## Enable versioning on your bucket

What happen if you upload an object name that already exists? 

You'll notice that the new object overrides the old one, without any option to restore the older version. 
If this happens unintentionally or due to a bug in the application code, it can result in the permanent loss of data.

The risk of data loss can be mitigated by implementing **versioning** in S3. 
When versioning is enabled, each object uploaded to S3 is assigned a unique version ID, which can be used to retrieve previous versions of the object. 
This allows you to recover data that was accidentally overwritten or deleted, and provides a safety net in case of data corruption or other issues.

1. Open the Amazon S3 console at [https://console\.aws\.amazon\.com/s3/](https://console.aws.amazon.com/s3/)\.

2. In the **Buckets** list, choose the name of the bucket that you want to enable versioning for\.

3. Choose **Properties**\.

4. Under **Bucket Versioning**, choose **Edit**\.

5. Choose **Enable**, and then choose **Save changes**\.

6. Upload multiple object with the same key, make sure versioning is working.

## Create lifecycle rule to manage non-current versions

When versioning is enabled in S3, every time an object is overwritten or deleted, a new version of that object is created. Over time, this can lead to a large number of versions for a given object, many of which may no longer be needed for business or compliance reasons.

By creating lifecycle rules, you can define actions to automatically transition non-current versions of objects to a lower-cost storage class or delete them altogether. This can help you reduce storage costs and improve the efficiency of your S3 usage, while also ensuring that you are in compliance with data retention policies and regulations.

For example, you might create a lifecycle rule to transition all non-current versions of objects to `Standard-IA` storage after 30 days, and then delete them after 365 days. This would allow you to retain current versions of objects in S3 for fast access, while still meeting your data retention requirements and reducing storage costs for non-current versions.


1. Choose the **Management** tab, and choose **Create lifecycle rule**\.

1. In **Lifecycle rule name**, enter a name for your rule\.

1. Choose the scope of the lifecycle rule (in this demo we will apply this lifecycle rule to all objects in the bucket).

1. Under **Lifecycle rule actions**, choose the actions that you want your lifecycle rule to perform:
   + Transition *noncurrent* versions of objects between storage classes
   + Permanently delete *noncurrent* versions of objects

1. Under **Transition non\-current versions of objects between storage classes**:

   1. In **Storage class transitions**, choose **Standard\-IA**.

   1. In **Days after object becomes non\-current**, enter 30.

1. Under **Permanently delete previous versions of objects**, in **Number of days after objects become previous versions**, enter 90 days.

1. Choose **Create rule**\.

   If the rule does not contain any errors, Amazon S3 enables it, and you can see it on the **Management** tab under **Lifecycle rules**\.

# Exercises

### :pencil2: Lifecycle policy to delete images older than 30 days

[Create a lifecycle policy](https://docs.aws.amazon.com/AmazonS3/latest/userguide/how-to-set-lifecycle-configuration-intro.html)) to move images older than 30 days to `Standard-IA` storage class. 
Then, delete them after 180 days.


