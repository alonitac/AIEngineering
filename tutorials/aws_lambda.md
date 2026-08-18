# Face blurring using Lambda and Step Function State Machine

Face blurring is one of the best-known practices when anonymizing both images and videos.
We will implement an event-driven system for face blurring using composition of different Lambda functions and a state machine.

Here is the high level architecture:

![][aws_face-blur]

Let's get started.

**\* Deploy your resources in a region where [Amazon Rekognition is supported](https://docs.aws.amazon.com/general/latest/gr/rekognition.html).**

#### Create the "face-detection" function

1. Open the [Functions page](https://console.aws.amazon.com/lambda/home#/functions) of the Lambda console\.
2. Choose **Create function**\.
3. Choose **Author from scratch**.
4. Create a `Python 3.11` runtime function from scratch.
5. Copy and deploy `face-blur-lambdas/face-detection/*.py` in the function source code page. Using the console code editor to create new python files as needed.
6. The IAM role of this function should have the following permissions: `AmazonS3FullAccess`, `AmazonRekognitionFullAccess` and `AWSStepFunctionsFullAccess`. It's recommended to use the same IAM role for all functions!
7. Configure a trigger for an **All object create** events for a given S3 bucket on objects with `.mp4` suffix (create a bucket and enable event notification if needed).


#### Create the "check-rekognition-job-status" function

1. Create a `Python 3.11` runtime function from scratch. Choose the same IAM role as the above function.
2. Copy and deploy `face-blur-lambdas/check-rekognition-job-status/lambda_function.py` as the function source code.

#### Create the "get-rekognized-faces" function

1. Create a `Python 3.11` runtime function from scratch. Choose the same IAM role as the above function.
2. Copy and deploy `face-blur-lambdas/get-rekognized-faces/lambda_function.py` as the function source code.

#### Create the "blur-faces" function

1. Create a **Container image** Lambda function based on the Docker image built from `face-blur-lambdas/blur-faces/Dockerfile`. Use an existing Docker image, or create a **private** ECR and build the image by:

    2. Open the Amazon ECR console at [https://console\.aws\.amazon\.com/ecr/repositories](https://console.aws.amazon.com/ecr/repositories).
    3. In the navigation pane, choose **Repositories**\.
    4. On the **Repositories** page, choose **Create repository**\.
    5. For **Repository name**, enter a unique name for your repository\.
    6. Choose **Create repository**\.
    7. Select the repository that you created and choose **View push commands** to view the steps to build and push an image to your new repository\.

2. In your lambda page, choose **Configuration**, then choose **Environment variables**, add the following env var to this function:
   `OUTPUT_BUCKET=<bucket-name>` where `<bucket-name>` is another bucket to which the processes videos will be uploaded (create one if needed).
3. This function is CPU and RAM intensive since it processes the video frame-by-frame. Make sure this it has enough time and space to finish (in the **General Configuration** tab, increase the timeout to 5 minutes and the memory to 2048MB).

#### Create Step Function state machine

1. Open the [Step Functions page](https://console.aws.amazon.com/lambda/home#/stepfunctions) of the Lambda console\.
2. Choose **Create state machine**.
3. In the **Choose a templat**e dialog box, select **Blank**.
4. In the Workflow Studio, click the **code** button.
3. Edit the JSON in the definition pane as follows:
    1. Copy and paste `face-blur-lambdas/state_machine.json`
    2. Change `<check-rekognition-job-status ARN>`, `<get-rekognized-faces ARN>` and `<blur-faces ARN>` according to the corresponding Lambda functions ARN.
4. Click **Create**.
6. It's recommended to enable **all logging** (can be done after the function is created).
8. Add the following env var to the **face-detection** function (the first function you've created):
   `STATE_MACHINE_ARN=<state-machine-ARN>`

#### Test the system

1. Upload a sample short mp4 video to the "input" S3 bucket (you can download [this](../face-blur-lambdas/sample-video.mp4) video).
2. Observe the Lambda invocation, as well as the state machine execution flow.
3. Download the processes video from in "output" S3 bucket and watch the results.


[aws_face-blur]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/aws_face-blur.png




